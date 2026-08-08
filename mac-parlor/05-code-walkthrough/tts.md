# 走读:tts.py(69 行)

> 平台自适应 Kokoro TTS:Apple Silicon 上用 mlx-audio(GPU via MLX),其他平台用 kokoro-onnx(CPU)。统一接口,运行时按平台选后端。

## 文件结构概览

```
tts.py
├─ _is_apple_silicon()(10–11)
├─ TTSBackend 基类(14–20)   sample_rate / generate 抽象
├─ MLXBackend(23–36)        Apple Silicon GPU
├─ ONNXBackend(39–54)       CPU
└─ load()(57–69)            按平台 + KOKORO_ONNX 选后端
```

---

## 一、平台检测与基类

```python
def _is_apple_silicon() -> bool:
    return sys.platform == "darwin" and platform.machine() == "arm64"

class TTSBackend:
    sample_rate: int = 24000
    def generate(self, text, voice="af_heart", speed=1.1) -> np.ndarray:
        raise NotImplementedError
```
统一接口:`generate(text, voice, speed) -> float32 PCM ndarray`。默认采样率 24000(Kokoro)。`speed=1.1`(略快于 1.0,语音助手语速)。

---

## 二、`MLXBackend`(23–36,Apple Silicon)

```python
def __init__(self):
    from mlx_audio.tts.generate import load_model
    self._model = load_model("mlx-community/Kokoro-82M-bf16")
    self.sample_rate = self._model.sample_rate
    list(self._model.generate(text="Hello", voice="af_heart", speed=1.0))   # warmup:触发 phonemizer/spacy 等初始化

def generate(self, text, voice="af_heart", speed=1.1) -> np.ndarray:
    results = list(self._model.generate(text=text, voice=voice, speed=speed))
    return np.concatenate([np.array(r.audio) for r in results])
```
- **mlx-community/Kokoro-82M-bf16**:MLX 优化的 bf16 Kokoro。
- **warmup**:首次生成触发整条管线(phonemizer、spacy)初始化,避免首轮卡顿。
- `generate` 返回多段音频拼接(Kokoro 按句/段产出多个 result)。

依赖(平台条件):`mlx-audio>=0.4.2`、`misaki[en]>=0.9.4`(phonemizer)、`num2words>=0.5.14`(数字转词)。

---

## 三、`ONNXBackend`(39–54,CPU / Linux)

```python
def __init__(self):
    import kokoro_onnx
    from huggingface_hub import hf_hub_download
    model_path = hf_hub_download("fastrtc/kokoro-onnx", "kokoro-v1.0.onnx")
    voices_path = hf_hub_download("fastrtc/kokoro-onnx", "voices-v1.0.bin")
    self._model = kokoro_onnx.Kokoro(model_path, voices_path)
    self.sample_rate = 24000

def generate(self, text, voice="af_heart", speed=1.1) -> np.ndarray:
    pcm, _sr = self._model.create(text, voice=voice, speed=speed)
    return pcm
```
- **fastrtc/kokoro-onnx**:ONNX Runtime CPU 版 Kokoro。
- 两个文件:模型 + voices(音色库)。
- `create` 返回 `(pcm, sample_rate)`。

依赖(平台条件):`kokoro-onnx>=0.5.0`(仅 Linux)。

---

## 四、`load()`(57–69)——后端选择

```python
def load() -> TTSBackend:
    if _is_apple_silicon() and not os.environ.get("KOKORO_ONNX"):
        try:
            backend = MLXBackend()
            print(f"TTS: mlx-audio (Apple GPU, sample_rate={backend.sample_rate})")
            return backend
        except ImportError:
            print("TTS: mlx-audio not installed, falling back to kokoro-onnx")
    backend = ONNXBackend()
    print(f"TTS: kokoro-onnx (CPU, sample_rate={backend.sample_rate})")
    return backend
```
选择逻辑:
1. **Apple Silicon 且未设 `KOKORO_ONNX`** → 尝试 MLXBackend。`ImportError`(mlx-audio 没装)→ 降级 ONNX。
2. **其他情况**(Linux / 设了 `KOKORO_ONNX` 调试逃生舱 / 降级)→ ONNXBackend。

`KOKORO_ONNX=1` 是 Apple Silicon 上强制 CPU 后端的调试逃生舱。

---

## 设计模式与改进点

| 模式 | 体现 |
|------|------|
| **策略模式** | TTSBackend 抽象 + MLX/ONNX 两策略 |
| **平台自适应工厂** | `load()` 运行时选后端 |
| **优雅降级** | MLX ImportError → ONNX |

改进点:
1. **无流式生成**:Kokoro 整句合成(`generate` 阻塞),靠 `pipeline.tts_worker` 按句并行掩盖延迟。若后端能流式产出 PCM,首音可进一步降低。
2. **`speed=1.1` 硬编码默认**:不可配置(per-mode tts_voice 可配但 speed 固定)。
3. **单音色默认 `af_heart`**:`modes.py` 所有模式都 `af_heart`。多语言 / 多角色场景需扩。
4. **无 fallback 链**:ONNXBackend 加载失败无进一步降级(无 TTS = 无语音)。可加最简的「嘟嘟声」兜底,但当前依赖外部模型资产。

---

← [turn_detector.md](turn_detector.md) | 下一个:[modes-reasoner.md](modes-reasoner.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)