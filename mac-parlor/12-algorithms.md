# 12 关键算法与复杂度分析

本章对 Parlor 的关键算法与复杂逻辑做**伪代码 + 时间/空间复杂度 + 优化建议**分析。全部基于实际源码。

---

## 12.1 Whisper 风格 log-mel 特征提取(turn_detector.py)

### 用途
把 16kHz 音频转成 `(80, 800)` log-mel 特征供 smart-turn-v3 ONNX 分类。8 秒窗口(128000 样本)。

### 算法步骤
```
输入: audio(1D float, 128000 样本)
1. 归一化:x = (x - mean) / sqrt(var + eps)           # 零均值单位方差,谱前
2. 反射填充:pad = N_FFT/2 = 200;x' = reflect_pad(x, pad)
3. 滑动窗:windows = sliding_window_view(x', 400)[::160]   # (num_frames, 400),零拷贝
4. 加窗 + rFFT:spec = rfft(windows * hann, axis=-1)        # 批量,替代 ~800 次 Python 循环
5. 功率谱:magnitudes = |spec|^2,转置 → (201, num_frames)
6. mel 投影:mel = MEL_FILTERS.T @ magnitudes             # (80, num_frames),Slaney 三角滤波器
7. 取下限:mel = max(MEL_FLOOR, mel)
8. log:log_spec = log10(mel);丢尾帧 log_spec[:,:-1]
9. 动态范围裁剪:log_spec = max(log_spec, log_spec.max() - 8)
10. 归一化到 ~[-1,1]:(log_spec + 4) / 4
返回 log_spec.astype(float32)  # (80, 800)
```

### 复杂度
- **时间**:
  - rFFT:O(num_frames · N_FFT · log N_FFT)= O(800 · 400 · 9) ≈ O(2.9M)。
  - mel 投影:O(80 · 201 · 800) ≈ O(12.9M)。
  - 总主导:mel 投影,**~O(N_mels · N_bins · N_frames)**。
- **空间**:`(80, 800)` float32 = 256KB 输出;中间 `magnitudes` (201,800) float64 ≈ 1.3MB。
- **实测**:~19ms(2 线程 onnxruntime 推理为主;特征提取占比小)。

### 关键优化
- `sliding_window_view`(零拷贝视图)+ 一次批量 rFFT 替代逐帧 Python 循环(注释:bit-identical,替代 ~800 次调用)。
- 预计算 `_HANN_WINDOW` / `_MEL_FILTERS`(模块加载时算一次)。

### 优化建议
- 流式场景可增量维护功率谱(当前按完整 utterance 调用,非瓶颈)。
- 用 FFT 实数对称性可减半,但 numpy rFFT 已优化。

---

## 12.2 流式句子切分(StreamParser._complete_sentences)

### 用途
增量切出完整句子送 TTS,且在遇到模仿的 `##` 标记后截断。

### 伪代码
```
输入: response(已积累正文), _emitted(已切偏移)
end = len(response)
hash_pos = response.find("##", _emitted)         # 模仿标记
if hash_pos != -1: end = min(end, hash_pos)
sentences = []
while True:
    m = SENTENCE_END_RE.search(response, _emitted, end)  # [.!?]+\s
    if not m: break
    sentence = response[_emitted:m.end()].strip()
    _emitted = m.end()                            # 推进,避免重扫
    if sentence: sentences.append(sentence)
return sentences
```

### 复杂度
- **时间**:O(L)(L = response 长度),`_emitted` 增量推进,每次 `feed` 只扫新增部分。`find("##", _emitted)` 与 `search(_, _emitted, end)` 都从 `_emitted` 起。
- **空间**:O(1) 额外(句子列表即返回值)。

### 优化建议
- 当前已最优(增量、避免重扫)。`SENTENCE_END_RE` 编译一次。

---

## 12.3 Barge-in 滑窗投票(app.js handleVadFrame)

### 用途
TTS 播放时判断用户是否真的开口打断(防回声误触)。

### 伪代码
```
输入: probs(当前帧 isSpeech 概率)
if state == "speaking":
    bargeWindow.push(probs.isSpeech > 0.85 ? 1 : 0)   # 0.85 = BARGE_SPEECH_P
    if len(bargeWindow) > 10: bargeWindow.shift()      # BARGE_WINDOW = 10
    if sum(bargeWindow) >= 6:                          # BARGE_HITS = 6
        bargeWindow = []
        triggerBargeIn()                               # 持续 6/10 帧才触发
    return
```

### 复杂度
- **时间**:O(W)= O(10) 每帧(滑动窗求和)。可 O(1) 用滚动计数器优化,但 W=10 微不足道。
- **空间**:O(W)。

### 设计理由(复杂度之外的逻辑)
- **持续而非单帧**:真实语音辅音/词边界下陷,连续帧计数器(要求 N 帧连续)在真麦永不触发;滑窗(最近 W 帧 ≥ HITS)能。
- 阈值 0.85 + grace 800ms + 服务端 cancel + ignoreIncomingAudio + phantom watchdog 多层。

### 优化建议
- 滚动计数器(入窗 +1,出窗 -1)省 sum,但收益可忽略。

---

## 12.4 指令回声检测(echoes_instruction)

### 用途
检测模型转写行/句子是否回声了轮指令文本(而非用户词)。

### 伪代码
```
输入: transcript, instruction, n(=5 句子 / 6 词)
words = norm(transcript)                          # 小写、去标点、切词
if len(words) < n: return False
instruction = strip_double_quoted_spans(instruction)  # 剥离 prompt 引用的预期短语
haystack = " " + " ".join(norm(instruction)) + " "
for i in 0 .. len(words)-n:
    ngram = " " + " ".join(words[i:i+n]) + " "
    if ngram in haystack: return True             # 任一 n 词连续命中 → 回声
return False
```

### 复杂度
- **时间**:O(W · n)(W = 转写词数)做子串匹配;ngram 构造 O(n)。haystack 一次构建 O(I)(I = 指令词数)。
- **空间**:O(I + W)。

### 当前调用模式
- `run_turn` 里每句跑一次 `echoes_instruction(sentence, instruction, n=6)`——**每次重建 haystack**。
- `clean_transcript` 对 transcript 跑一次 n=5。

### 优化建议(P3 技术债)
- 缓存 instruction 归一化结果:每轮 instruction 固定,`_instruction_text` 之后归一化一次,haystack 复用。省每句的 O(I) 重建。

---

## 12.5 历史轮转(rotate_history)

### 伪代码
```
输入: history(list[Message])
if len(history) <= 3: return history              # 不可轮转
keep = 1 + max(2, 3*(len-1)//4)                   # 系统 + 最近 3/4
while keep > 3 and history[-(keep-1)].role != "user":
    keep -= 1                                     # 强制落在 user
return [history[0]] + history[-(keep-1):]
```

### 复杂度
- **时间**:O(keep)(最坏线性回退找 user;实际 few steps)。
- **空间**:O(keep) 新列表。

### 正确性约束
- 保留 `history[0]`(系统提示,前缀缓存前缀)。
- kept 切片起点必须是 user(防孤立 assistant:回答着已不存在的用户词)。
- ≤3 条直接返回(切片 [system] 会复制系统提示)。

### 优化建议
- 当前 O(keep) 已足够(每会话偶发)。

---

## 12.6 Token 估算(estimate_tokens)

### 伪代码
```
total = 0
for m in messages:
    if content is str: total += len//4 + 8
    else for part:
        text: len(text)//4
        audio: (wav_bytes//32000) * 32           # 16kHz s16,32 token/s
        image: 300
    total += 8
```

### 复杂度
- **时间**:O(Σ parts)。
- **空间**:O(1)。

### 用途
驱动历史轮转**守卫**(粗糙即可);真实计数由 `prompt_tokens["last"]`(llama-server usage)盖过(ADR-012)。系数:`AUDIO_TOKENS_PER_SEC=32`、`IMAGE_TOKENS=300`、文本 `len//4`。

### 优化建议
- 无需(粗糙估算够用,真实计数兜底)。

---

## 12.7 三段重叠流水线(run_turn)的延迟建模

### 模型
三段:LLM 解码(P1)→ 句子切分(P2,可忽略)→ TTS 合成(P3)。

```
传统串行:总延迟 = P1(全文) + P3(全文)
Parlor 重叠:总延迟 ≈ TTFFirstSentence(P1 首句) + TTS(首句)
              后续句子在播放前一期间解码+合成
```

### 关键指标(timings)
- `ttfa_s`(首音延迟)≈ prefill_s + 首句解码 + 首句 TTS。
- overlap prefill(ADR-002)把 prefill 摊到「用户说话时」,所以 ttfa_s 主要剩首句解码 + TTS。

### 优化建议
- TTS 流式生成(首句未合成完即播)可再降 ttfa。
- 减首句长度(prompt 让模型先短答)。

---

## 12.8 端到端 token / 延迟预算(M3 Pro, e2b, 来自 README)

| 阶段 | 短问(~2s) | 长问(~9s,流式) |
|------|----------|----------------|
| VAD 静音检测 | ~200ms | ~200ms |
| 首音(e2b) | ~0.7s | ~1.3–1.4s |
| 整轮完成 | ~1.3s | ~2.1–2.3s |

> overlap prefill 让长问首音几乎不比短问慢太多(只多付尾部音频 prefill)。E4B 约为 e2b 的 1.8x。

---

下一章:[13 测试策略](13-testing.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)