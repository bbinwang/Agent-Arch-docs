# 走读:pipeline.py(476 行)

> 流式轮次管道:消息内容辅助、增量响应/转写解析器(`StreamParser`)、`run_turn`(解码→句子→TTS 三段重叠流水线)、以及投机缓存预热(`prime_cache`)。这是**延迟优化的核心战场**。

## 文件结构概览

```
pipeline.py
├─ 常量(22–34)            正则、MAX_OUTPUT_TOKENS=256、TAIL_SILENCE_S=0.3、token 估算系数
├─ 消息内容辅助(40–104)   image_part/audio_part/text_part/valid_audio/user_content
│                          /estimate_tokens/wav_to_float32/pad_tail_silence
├─ WS 协议辅助(109–117)   send_json/release_client
├─ StreamParser(122–213)   ★ 增量解析状态机
├─ 转写清洗(218–260)      _norm_words/NO_SPEECH_RE/echoes_instruction/_instruction_text
├─ run_turn(263–461)       ★ 三段重叠流水线
└─ prime_cache(464–476)    投机预热
```

---

## 一、常量(22–34)

```python
TRANSCRIPT_TAG_RE = re.compile(r"#{2,}[ \t]*TRANSCRIPT[ \t]*:[ \t]*", re.IGNORECASE)
SENTENCE_END_RE = re.compile(r"[.!?]+\s")
MAX_OUTPUT_TOKENS = 256
TAIL_SILENCE_S = 0.3
AUDIO_TOKENS_PER_SEC = 32
IMAGE_TOKENS = 300
_DONE = object()
```

关键注释:`TRANSCRIPT_TAG_RE` 容忍 `### TRANSCRIPT : ...`(空格变体),但**冒号是必需的**,且冒号后只允许 `[ \t]`——因为这个正则跑在**部分流式 buffer** 上:可选冒号会在 `:` token 到达前匹配(把 `:` 泄漏进转写),`\s*` 会让换行 delta 提前终止空转写行。

`TAIL_SILENCE_S = 0.3`:补在末段 WAV 内(在 LLM 看到之前),修复 VAD 硬切导致的编码器幻觉补全。

---

## 二、消息内容辅助(40–104)

```python
def image_part(b64): {"type":"image_url","image_url":{"url":"data:image/jpeg;base64,"+b64}}
def audio_part(b64): {"type":"input_audio","input_audio":{"data":b64,"format":"wav"}}
def text_part(text): {"type":"text","text":text}

def valid_audio(b64) -> bool:
    # 至少 ~100ms 的 16kHz s16 WAV;空音频 llama-server 400,一条坏消息毒化所有后续请求
    return bool(b64) and len(b64) * 3 // 4 > 44 + 3200

def user_content(image_b64, audio_b64s) -> list:
    # 缓存稳定的规范顺序:image 先,音频段由旧到新
    parts = [image_part(image_b64)] if image_b64 else []
    parts += [audio_part(b) for b in audio_b64s]
    return parts
```

**`user_content` 的顺序是「缓存稳定」的关键**——预热与最终请求必须用相同顺序,否则前缀分叉。image 先、audio 由旧到新。

### `estimate_tokens`(66–82)
粗略 token 估算(驱动历史轮转守卫):文本 `len//4`;音频 `(wav_bytes//32000)*32`(16kHz s16,每秒 32 token);图 300。每消息 +8 开销。

### `wav_to_float32` / `pad_tail_silence`(85–104)
- `wav_to_float32`:base64 WAV → int16 PCM → float32 / 32768(给 turn_detector 用)。
- `pad_tail_silence`:在 WAV **内部**追加 0.3s 静音。注释:必须与语音在**同一个 WAV** 里——单独的静音 part 阻止不了编码器对硬切词尾的幻觉补全。

---

## 三、`StreamParser`(122–213)——★ 增量解析状态机

### 设计目标(类 docstring)
增量解析 `###TRANSCRIPT: <words>\n<response>`。**transcript 行先行**:模型先承诺它听到了什么再回答(生成在回复之后会把转写变成「凭记忆改写」——WER 0.39 vs 0.00)。`feed()` 返回已完整的回复句子(transcript 行 delta 返回空);`finalize()` 返回尾部残句 + transcript。`expect_transcript=False`(文本/图轮)时回复直接流式,模仿的尾随标签被切除永不朗读。

### 状态字段
```python
self.response = ""            # 已积累的回复正文
self.transcript = None        # 已解析的转写(或 None)
self._awaiting = expect_transcript   # 是否还在等标签
self._got_tag = False
self._buf = ""                # 等待标签 / 换行期间的缓冲
self._before_tag = ""         # 标签前的杂散文本 → 回复前缀
self._emitted = 0             # 已切出的句子偏移(避免重复扫描)
```

### `feed(delta)`(151–177)
- 若 `_awaiting`(等标签):
  1. `buf += delta`。
  2. 若没找到 tag → `return []`(继续等)。
  3. 找到 tag → `_got_tag=True`,`_before_tag = buf[:tag.start]`(标签前杂散文本留作回复前缀),`buf = buf[tag.end:]`。
  4. `buf.lstrip()`(防首换行 delta 终止空转写)。
  5. 找换行:找不到且 `len(buf)<600` → `return []`;超 600(失控转写行)→ 取首句作转写、其余流式(不让 TTS 被劫持)。
  6. `transcript = buf[:newline].strip() or None`;`response = (before_tag + buf[newline+1:]).lstrip()`;`_awaiting=False`。
- 否则:`response += delta`。
- 最后 `_complete_sentences()`。

### `_complete_sentences()`(179–195)
关键:**遇到模仿的 `##` 标记后截断**。`end = response.find("##", emitted)` 若存在则 `end = min(end, hash_pos)`——永远不朗读标记后的内容。然后用 `SENTENCE_END_RE` 从 `_emitted` 到 `end` 切完整句子,推进 `_emitted`。

### `finalize()`(197–213)
处理流截断 / 标签后无换行的退化:
- 若仍 `_awaiting` 且 `_got_tag`:无换行到达(流截断 / 模型把回复接在标签行)→ 首句作 transcript、其余作回复(绝不全吞)。
- 若 `_awaiting` 且没 tag:`response = buf`。
- 切句;尾段按 `#{2,}` 切除模仿标记。返回 `(sentences + [tail if tail], transcript)`。

### 设计精髓
`StreamParser` 是一个**面向不可靠增量输入的容错状态机**:它要处理「冒号还没流到」「换行 delta 提前到」「失控超长行」「流截断」「模型把回复接在标签行」「模型模仿 `##` 标记」六种退化,且**永远优先正确性**(转写宁可晚一点也不能泄漏标记、不能吞回复)。

---

## 四、转写清洗(218–260)

### `NO_SPEECH_RE`(228–231)
匹配「纯括号标注」转写—— sanctioned 的 `(no speech)` 或模型自创变体(`(noise)`/`[Silence]`/`*sigh*`/`(background noise)`)。真转写是纯词、永不全括号;长度上限防「真带括号的长篇」被误杀。

### `echoes_instruction(transcript, instruction, n=5)`(234–252)
检测模型转写行**回声了轮指令文本**(而非用户词)——例如 flush 轮的 `###TRANSCRIPT: The user paused mid-thought, so on a new line: ...`。逻辑:转写归一化后,任一连续 n 词在指令里逐字出现 → 回声。真语音不会复现 5 词的 prompt 片段(短重叠如「what you see」是常见英语)。**双引号片段先从指令剥离**:prompt 引用用户**预期会说**的短语(翻译 prompt 的退出例 "go back to normal conversation"),真复述不应被当回声。

### `_instruction_text`(255–260)
从 messages 最后一条取纯文本(支持 str 与 content list)。

---

## 五、`run_turn`(263–461)——★ 三段重叠流水线

这是延迟优化的核心。签名:

```python
async def run_turn(ws, messages, interrupted, active, tts_backend,
                   expect_transcript=True, p_complete=None,
                   tts_voice="af_heart", proactive=False, fallback=None
                   ) -> tuple[str, int|None, bool, bool]:
# 返回 (raw_text, prompt_tokens, no_speech, spoke)
```

### 返回值语义(来自 docstring)
- `raw_text`:原始生成文本。
- `prompt_tokens`:llama-server 报告的真实 prompt token 数(驱动历史轮转),无则 None。
- `no_speech`:模型写了**必须被拒绝**的转写行(no-speech 标注或指令回声)——无用户词撑腰,调用者不得存/不得执行。
- `spoke`:是否发了 TTS 音频;调用者的 floor 簿记据此(而非文本)——纯转写轮有文本但无声。
- `proactive`:服务端发起轮(委派投递),transcript 行是模型自身回声,不发 transcript 帧,`turn_final` 带 `proactive=True`。

### 5.1 三段流水线架构(291–335)

```python
loop = asyncio.get_event_loop()
chunk_q = asyncio.Queue()        # token: producer线程 → 主循环
stream = llama.ChatStream(messages, MAX_OUTPUT_TOKENS)
active["stream"] = stream        # 暴露给 interrupt 取消
raw = {"text": ""}

def produce():                   # 在 executor 线程里跑
    def on_delta(text):
        raw["text"] += text
        loop.call_soon_threadsafe(chunk_q.put_nowait, text)  # 线程安全入队
    stream.run(on_delta)
    loop.call_soon_threadsafe(chunk_q.put_nowait, _DONE)
    # 异常 → loop.call_soon_threadsafe(chunk_q.put_nowait, e)

producer = loop.run_in_executor(None, produce)

sentence_q = asyncio.Queue()     # sentence: 主循环 → tts_worker
audio_state = {"started":False, "first_audio_at":None, "chunks":0}

async def tts_worker():          # 消费 sentence_q
    while True:
        sentence = await sentence_q.get()
        if sentence is _DONE: return
        if interrupted.is_set(): continue   # 继续排空
        pcm = await loop.run_in_executor(None, lambda: tts_backend.generate(sentence, voice=tts_voice))
        if interrupted.is_set(): continue
        # 首句 → audio_start;转 int16 → audio_chunk
```

**三段重叠**:
- 第 1 段(producer 线程):`ChatStream.run` 解码,每 token 经 `call_soon_threadsafe` 入 `chunk_q`。
- 第 2 段(主循环):从 `chunk_q` 取 token,喂 `parser.feed`,得句子入 `sentence_q`。
- 第 3 段(tts_worker 协程):从 `sentence_q` 取句子,TTS 合成发客户端。

**第一个完整句子一产生就进 TTS**,不等整段解码完——这是首音延迟下限的关键。

### 5.2 `dispatch(sentences)`(356–386)
对每个句子做发送前清洗:
- 非 proactive 且 `echoes_instruction(sentence, instruction, n=6)` → 抑制(实测「CRIPT: Begin your reply with one line:」被朗读)。n=6 让用户可能合法触发的短引用短语仍能朗读。
- `NO_SPEECH_RE.match(sentence)` → 抑制(注释替身)。
- 首 sentence → 记 `tts_started_at`,发 `text_delta`,入 `sentence_q`。

**抑制是 TTS/显示层 only**:句子留在 raw text 里(混合轮:真转写 + 一个回声句,存了回声——全回声轮经 no_speech 整体丢弃,那才是毒化情形)。

### 5.3 主消费循环(388–428)
```python
while True:
    item = await chunk_q.get()
    if item is _DONE: break
    if isinstance(item, Exception): raise item
    timings["prefill_s"] = ...    # 首个 token 到达
    if not interrupted.is_set():
        await dispatch(parser.feed(item))
        if parser.transcript and not transcript_sent:
            transcript_sent = True
            timings["transcript_s"] = ...
            shown = clean_transcript(parser.transcript)
            if shown and not proactive:
                send_json(ws, {"type":"transcript", "transcription":shown, "p_complete":p_complete})

tail, transcript = parser.finalize()
timings["llm_time"] / ["decode_s"] = ...
if not interrupted.is_set():
    await dispatch(tail)
    if tts_started_at is None and fallback:   # 仅有转写行、无回复
        raw["text"] += "\n" + fallback
        await dispatch([fallback])
finally:
    active["stream"] = None
    sentence_q.put_nowait(_DONE)
    await tts_task          # producer 线程必须在 slot 复用前完成
    await producer
```

**fallback 兜底**:轮可能以「仅有转写行」结束(任何大小模型对截断音频都这样),不能静默收场——朗读调用者的 fallback,保持历史与实际所说一致。

### 5.4 timings 与终结帧(430–461)
- `audio_state["first_audio_at"]` → `ttfa_s`(首音延迟)。
- `tts_started_at` → `tts_time`。
- 打印 `LLM (Xs, prefill Ys) heard: ... → ...`。
- `turn_no_speech()`:`not proactive and parser.transcript is not None and clean_transcript(...) is None`。
- interrupted → 返回(不发终结帧)。
- 否则发 `turn_final`(transcription / proactive / timings / p_complete / spoke)。`spoke=False` 时客户端不得等 audio_end。若 spoke → 发 `audio_end`。

### 5.5 `clean_transcript`(341–354)
客户端把它当用户词显示——剔除指令回声、剔除 no-speech 标注。两种都 `print` 诊断。

---

## 六、`prime_cache`(464–476)

```python
async def prime_cache(messages) -> None:
    t0 = time.time()
    try:
        await loop.run_in_executor(None, lambda: llama.chat_blocking(messages, max_tokens=1))
        print(f"Primed cache ({time.time()-t0:.2f}s)")
    except Exception as e:
        print(f"Cache priming failed: {e}")
```

fire-and-discard:把前缀推过 llama-server 缓存(`cache_prompt=True` 在 `_chat_body` 默认开)。**失败不上报**——最终请求照常,只是全量 prefill。内容必须是纯媒体追加(尾随 text block 会让前缀分叉)。

---

## 关键算法复杂度与改进点

| 项 | 复杂度 | 说明 |
|----|--------|------|
| `StreamParser.feed` | O(n) 摊销 | `_emitted` 避免重扫;`find("##", _emitted)` 增量 |
| `estimate_tokens` | O(messages × parts) | 每轮一次,cheap |
| `echoes_instruction` | O(W·n) | W=转写词数,n=5/6 滑窗;haystack 一次构建 |

**改进点**:
1. `MAX_OUTPUT_TOKENS=256` 硬上限——长回复被截断(但有 fallback 与 finalize 兜底)。可按模式调整。
2. `sentence_q` 无背压——若 TTS 慢于解码,句子堆积。当前靠 `MAX_OUTPUT_TOKENS` 间接限流,大模型 + 慢 TTS 下可能积压。
3. `dispatch` 里 `echoes_instruction` 每句跑一次(重建 haystack)——可缓存 instruction 归一化结果(当前每轮 instruction 固定,缓存于 `_instruction_text` 之后即可)。
4. 异常路径:`raise item` 把 producer 异常抛到主循环 try 外,由 server 的 except 兜住。链路清晰但跨函数,文档化值得。

---

← [server.md](server.md) | 下一个:[llama.md](llama.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕