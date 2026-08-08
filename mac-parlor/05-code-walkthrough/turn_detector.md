# 走读:turn_detector.py(230 行)

> 用 smart-turn-v3 ONNX 分类器做端点(轮次结束)检测。一个 8M 参数的音频原生小模型(Pipecat,BSD-2),专为「说话人是否说完」训练,远比 prompt 一个 2B LLM 可靠,CPU 上几十毫秒响应。本模块还**自带一份 vendored 的纯 numpy Whisper 风格 log-mel 特征提取**,免去对 `transformers` 的依赖。

## 文件结构概览

```
turn_detector.py
├─ 配置(12–22)        HF_REPO / HF_FILE / SAMPLE_RATE / WINDOW_SECONDS=8
├─ TurnDetector(24–58) 加载 ONNX + warmup + predict
└─ Whisper log-mel 特征(61–230)  vendored 纯 numpy 实现
   ├─ 常量(74–79)
   ├─ Slaney mel 换算(82–103)  _hertz_to_mel_slaney / _mel_to_hertz_slaney
   ├─ mel 滤波器组(106–149)    _build_mel_filterbank / _periodic_hann_window
   ├─ 功率谱(152–176)          _power_spectrogram(sliding_window_view 向量化)
   └─ 主函数(179–230)          compute_whisper_log_mel_features
```

---

## 一、配置与 `TurnDetector`

```python
HF_REPO = "pipecat-ai/smart-turn-v3"
HF_FILE = "smart-turn-v3.2-cpu.onnx"
SAMPLE_RATE = 16000
WINDOW_SECONDS = 8   # 模型判最近 8 秒
```

### `TurnDetector.__init__`(25–42)
```python
model_path = hf_hub_download(HF_REPO, HF_FILE)   # 网络失败 → local_files_only=True
so = ort.SessionOptions()
so.execution_mode = ORT_SEQUENTIAL
so.inter_op_num_threads = 1
so.intra_op_num_threads = 2
so.graph_optimization_level = ORT_ENABLE_ALL
self._session = InferenceSession(model_path, sess_options=so)
self.predict(np.zeros(SAMPLE_RATE, dtype=np.float32))   # warmup:首次跑付图初始化
print("Turn detector loaded (smart-turn-v3.2).")
```
- **单线程串行 + 2 线程内算**:为低延迟 CPU 推理调优,避免与 llama-server 抢核。
- **warmup**:首次推理触发图初始化,避免首轮用户请求时卡顿。

### `predict(audio)`(44–58)
```python
max_samples = WINDOW_SECONDS * SAMPLE_RATE          # 128000
if len(audio) > max_samples: audio = audio[-max_samples:]   # 截最近 8s
elif len(audio) < max_samples: audio = np.pad(audio, (max_samples-len(audio), 0))  # 前补零
log_mel = compute_whisper_log_mel_features(audio, do_normalize=True)
outputs = self._session.run(None, {"input_features": np.expand_dims(log_mel, 0)})
probability = outputs[0][0].item()
print(f"Turn detector: p(complete)={probability:.2f} ({elapsed_ms:.0f}ms)")
return probability > 0.5, probability
```
返回 `(complete, probability)`。**取最近 8 秒**判断,过短前补零(对齐模型输入),过长截尾。

---

## 二、Vendored Whisper log-mel 特征(61–230)

### 为什么 vendored
注释:复制 `transformers.WhisperFeatureExtractor` 的输出,使 smart-turn-v3 能**不导入 `transformers` 包**就计算特征。数学镜像 `transformers.models.whisper.feature_extraction_whisper` 与 `transformers.audio_utils`(Apache-2.0)。License 头保留(BSD-2,Daily)。

### 常量(74–79)
```python
_N_FFT = 400; _HOP_LENGTH = 160; _N_MELS = 80
_SAMPLING_RATE = 16000; _MEL_FLOOR = 1e-10; _NORM_VARIANCE_EPS = 1e-7
```
预计算:`_HANN_WINDOW`(周期 Hann)与 `_MEL_FILTERS`(80 mel 滤波器)在模块加载时算好,避免每次 predict 重算。

### Slaney mel 换算(82–103)
`_hertz_to_mel_slaney` / `_mel_to_hertz_slaney`:Slaney 尺度。1000Hz 以下线性(3·f/200),以上对数(`min_log_mel + log(f/1000)·logstep`)。这是 librosa/transformers 的标准 mel 尺度。

### `_build_mel_filterbank`(106–133)
构建 Slaney 归一化三角 mel 滤波器组,形状 `(num_frequency_bins, num_mel_filters)`。三角斜率由 `filter_freqs` 差分算,Slaney 面积归一(`enorm`)。

### `_periodic_hann_window`(136–138)
`np.hanning(N+1)[:-1]`——周期 Hann,匹配 `torch.hann_window`。

### `_power_spectrogram`(152–176)——★ 向量化优化
```python
pad = frame_length // 2
padded = np.pad(waveform.astype(float64), (pad, pad), mode="reflect")
windows = sliding_window_view(padded, frame_length)[::hop_length]   # 所有帧的零拷贝视图
spec = np.fft.rfft(windows * win, axis=-1)                          # 一次批量 rFFT 替代 ~800 次 Python 循环
return (np.abs(spec) ** 2).T
```
注释强调:**零拷贝滑动窗口视图 + 一次批量 rFFT**,替代此前 ~800 次 Python 级逐帧调用,逐位相同。这是把特征提取压到几十毫秒的关键。

### `compute_whisper_log_mel_features`(179–230)
```python
x = audio float32; pad/truncate 到 128000
if do_normalize: x = (x - x.mean()) / np.sqrt(x.var() + eps)   # 零均值单位方差,谱前
magnitudes = _power_spectrogram(x, HANN, N_FFT, HOP)
mel_spec = np.maximum(MEL_FLOOR, MEL_FILTERS.T @ magnitudes)
log_spec = np.log10(mel_spec)
log_spec = log_spec[:, :-1]                       # 丢尾帧
log_spec = np.maximum(log_spec, log_spec.max() - 8.0)
log_spec = (log_spec + 4.0) / 4.0
return log_spec.astype(float32)                   # (80, 800),约 [-1,1]
```
返回 `(80, 800)` log-mel。归一化在谱**前**做(镜像 transformers),float32 精度匹配参考实现。

---

## 复杂度与改进点

| 项 | 复杂度 | 说明 |
|----|--------|------|
| log-mel 特征 | O(N log N) | N=FFT 点数;向量化后常数极小 |
| ONNX 推理 | ~19ms | 8M 参数,2 线程 |

改进点:
1. **`predict` 每次重算完整特征**:8s 窗口固定,流式场景可增量计算(但当前按完整 utterance 调用,非瓶颈)。
2. **`intra_op_num_threads=2` 硬编码**:不同核数机器可调;与 llama-server 抢核的边界靠线程数控制。
3. **阈值 0.5 硬编码**:`probability > 0.5`。无配置;某些场景(嘈杂)可能想调。但 smart-turn 已是该任务的 SOTA 小模型,阈值经验已优。
4. **模型固定 v3.2**:升级需改常量;无自动版本探测。

---

← [actions.md](actions.md) | 下一个:[tts.md](tts.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)