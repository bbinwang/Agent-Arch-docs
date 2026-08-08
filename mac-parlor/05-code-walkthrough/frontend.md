# 走读:前端 web/(app.js 820 行 + index.html 60 行)

> 原生 Vanilla JS,无构建步骤、无框架。负责采集、VAD、~3s 语音块流式、帧预取、barge-in 滑窗投票、流式无缝播放、UI 状态机、模式/委派/定时器芯片。前端逻辑密度与后端相当。

## 一、index.html(60 行)结构

页面骨架,CDN 引入 `onnxruntime-web@1.22.0` 与 `@ricky0123/vad-web@0.0.29`:

```
header     → logo + {{model}} 标签 + 状态丸
portal-stack
  ├ viewportWrap     → 摄像头视口(video + glow 层)
  ├ waveform-wrap    → 波形 canvas
  ├ messages         → 转写区
  └ controls         → cameraToggle / modeChip(+stop) / 状态指示 / On-device 丸
script → ort.min.js + vad bundle + /static/app.js
```

`{{model}}` 由 server.py `root()` 替换为 `model_label()`。

---

## 二、app.js 全局状态与状态机

```js
let ws, mediaStream, myvad;
let cameraEnabled = true, audioCtx, state = 'loading', ignoreIncomingAudio = false;
let streamSampleRate = 24000, streamNextTime = 0, streamSources = [];
const STATE_COLORS = { listening, processing, speaking, loading };  // 每 state [glow, glow-dim]
```

### `setState(newState)`(110–158)
状态机核心:`loading / listening / processing / speaking`。
- 设状态点 class、标签文字、viewport className。
- 解析 `[glow, glow-dim]` 写入 CSS 变量,波形据此着色。
- speaking → `updateSpeakingGlow()`(按 TTS 音量调制 glow)。
- **`myvad.setOptions({positiveSpeechThreshold: speaking?0.92:0.5})`**:TTS 播放时抬高 VAD 分段阈值(回声少开 speech 段);barge-in 读原始概率,不受此阈值门控。
- listening → 连 micSource 到 analyser(波形);非 listening 断开。
- **processing 看门狗**(147–157):30s 后若仍 processing → 强制回 listening(防服务端出错未发终态帧,卡死)。

### `goListening(announce=true)` / `goProcessing()`(164–172)
- `goListening`:`announce && !speechActive` 时清 `ignoreIncomingAudio` + 发 `ready`(只在真空闲时发;mic 可能活的 barge-in 时不发)。
- `goProcessing`:进 thinking。

---

## 三、WebSocket 收发(174–285)

### `connect()`(181–285)
- `ws.onopen`:已过 loading → listening。
- `ws.onclose`:重置 per-turn 状态(framePrefetched/serverHoldingAudio/气泡/芯片/模式/flush 计时器)→ 2s 重连。**重连 = 全新服务端会话**。
- `ws.onmessage`:按 `msg.type` 分发(见 [§07 API](../07-api.md) 全表)。重点:
  - `turn_incomplete`:设 `serverHoldingAudio=true`,气泡 meta 显示完成度%,goListening,startFlushTimer。
  - `transcript`:`fillUserBubble`(转写先行上屏)。
  - `turn_final`:proactive 不填气泡;`!spoke && !transcription` 删空气泡;否则填;`!spoke` → goListening(无音频跟随)。
  - `audio_start`/`audio_chunk`/`audio_end`:流式播放(见 §五)。
  - `mode_changed`/`delegation_*`/`timer_*`:更新对应芯片 UI。

---

## 四、VAD 与采集(491–654)——★ 核心

### 语音块流式(496–575)
```js
const CHUNK_SECONDS = 3.0;     // 说话中每 ~3s 发块,服务端预填
const PRE_FRAMES = 10;          // ~320ms 预语音上下文
let uttFrames = [], preFrames = [], uttSamplesSent = 0, chunkSeq = 0, speechActive = false;
```

- `handleVadFrame(probs, frame)`(517–554):
  - **speaking 时**:滑窗投票(见 barge-in)。
  - **speechActive && listening**:phantom 看门狗(见下);推 frame;累计达 3s → `sendSpeechChunk`。
  - **!speechActive**:推 preFrames(滚动缓冲,超 PRE_FRAMES 丢最旧)。
- `sendSpeechChunk`(569–575):concat 后 subarray 未发送部分;<4800 样本(<300ms)不发(不值得预热请求)。
- `startUtteranceCapture`(579–585):`uttFrames=[...preFrames]`(带预语音上下文),reset 计数。
- `handleSpeechStart`(598–603):clearFlushTimer;listening → `prefetchFrame()` + `startUtteranceCapture`。
- `handleSpeechEnd(audio)`(619–654):见下「完整发送」。

### Barge-in 滑窗投票(506–617)——★ 回声防御
```js
const BARGE_WINDOW = 10, BARGE_HITS = 6, BARGE_SPEECH_P = 0.85, BARGE_IN_GRACE_MS = 800;
```
speaking 时,每帧 `isSpeech > 0.85` 记 1,最近 10 帧 ≥6 才触发。注释:**必须持续**——真实语音辅音/词边界下陷,「连续帧计数器」在真麦上永不触发,而滑窗能。`triggerBargeIn`:超 800ms echo grace → `stopPlayback()` + `ignoreIncomingAudio=true` + 发 `interrupt` + `goListening(false)`(mic 活,不发 ready)+ prefetchFrame + startUtteranceCapture。

### Phantom capture 看门狗(528–545)
barge-in 在 VAD 自身状态机外启动采集(`startUtteranceCapture`)。若 VAD 从未真正进 speech(一个 (0.85,0.92] 带的回声突发能单独触门),永无 speech-end → 会永远流静音块。watchdog:~1s 低于负阈值(0.25)30 帧 → reset 采集 + goListening(重发 idle,防服务端 interrupted 标志搁浅排队投递)。

### 帧预取(590–596)
`prefetchFrame()`:speech **开始瞬间**发摄像头帧——服务端在用户还说话时就预填图像 token(~75% 更快首 token on camera turns)。`framePrefetched` 防重复。

### 完整发送(`handleSpeechEnd`)
```js
const wasStreaming = speechActive && uttSamplesSent > 0;
speechActive = false;   // 永远 reset,泄漏会永远流麦音
if (state !== 'listening' || !wsOpen()) { framePrefetched=false; return; }   // 丢弃
goProcessing(); addUserLoadingBubble(withCamera?'with camera':'');
currentAssistantEl = null; ignoreIncomingAudio = false;
if (wasStreaming) {
    const tail = concat(uttFrames).subarray(uttSamplesSent);   // 只发未送尾部(可空)
    if (tail.length >= 1600) payload.audio = float32ToWavBase64(tail);
    payload.chunked = true;
} else payload.audio = float32ToWavBase64(audio);
if (imageBase64) payload.image = imageBase64;
wsSend(payload);
```
**流式优化**:已流式的块只发尾部;服务端从已预填块响应。

---

## 五、流式无缝播放(674–740)

```js
function queueAudioChunk(base64Pcm) {
    // base64 → Int16 → Float32
    audioBuffer = createBuffer(1, float32.length, streamSampleRate);
    source = createBufferSource(); source.buffer = audioBuffer;
    source.connect(audioCtx.destination); source.connect(analyser);
    startAt = Math.max(streamNextTime, audioCtx.currentTime);   // 无缝拼接
    source.start(startAt); streamNextTime = startAt + audioBuffer.duration;
    streamSources.push(source);
    source.onended = () => { splice; 若空 && speaking → goListening(); };
}
```
**关键注释(683–690)**:曾试过经 `MediaStreamDestination + <audio>` 让 Chrome 回声消除拿参考信号(crbug 687574),但 macOS 上会启用系统语音处理——**每轮给 TTS 音色着色**且播放时压制用户麦(杀 barge-in)。改为**裸输出**(保持音色干净 + 麦诚实),回声防御靠:持续语音 barge-in 门控 + TTS 后 grace + prompt 回声规则(戴耳机即无问题)。

---

## 六、Flush 计时器(427–455)

```js
let flushTimer = null, serverHoldingAudio = false;
function startFlushTimer() {
    clearFlushTimer();
    flushTimer = setTimeout(() => {
        if (state==='listening' && !speechActive && wsOpen()) {   // 不冲掉实时语音
            serverHoldingAudio = false; currentAssistantEl=null; ignoreIncomingAudio=false;
            addUserLoadingBubble(); wsSend({type:'flush'}); goProcessing();
        }
    }, 2500);
}
```
incomplete 轮安静 2.5s → flush。一次 hold 一次 flush;`serverHoldingAudio` 让清掉的计时器(误触/barge-in)能重新武装。

---

## 七、UI 辅助:气泡 / 芯片 / 波形

- **pendingUserBubble / fillUserBubble**(386–425):每 utterance-group 一个 pending 气泡,转写先到先填(early transcript 胜 turn_final 的副本)。
- **delegationChips / timerChips**(326–384):研究任务 chip(转圈直到答案);定时器 chip(倒计时 + ✕,500ms ticker 自停)。
- **波形 `drawWaveform`**(44–85):40 柱,有 analyser 数据按频段映射,否则 ambient 漂移;按 state 着色。
- **`float32ToWavBase64`**(657–672):手写 44 字节 WAV 头 + PCM,16kHz s16 mono。

---

## 设计模式与改进点

| 模式 | 体现 |
|------|------|
| **显式状态机** | loading/listening/processing/speaking + 看门狗 |
| **defense-in-depth 回声防御** | 滑窗投票 + grace + phantom watchdog + prompt 规则 |
| **流式优化** | 块流式预填 + 尾部发送 + 无缝播放拼接 |
| **乐观 UI + 服务端权威** | pending 气泡先占位,转写/turn_final 填充 |

改进点:
1. **全局变量多**:`app.js` 顶层大量 `let`,无模块封装。可包成 IIFE / 模块。
2. **无错误 UI**:WebSocket 错误仅 console;用户只见 disconnected 重连。
3. **CDN 强依赖**:VAD/ORT 从 CDN 加载,离线首跑需联网(模型本地化但运行时库未本地化)。
4. **回声仍靠「戴耳机」**:`echoCancellation:true` 在 constraints 里但裸输出绕过系统处理;扬声器外放场景回声仍可能。
5. **波形 ambient 漂移用 `+= 0.0001` 累加**:长会话 `ambientPhase` 浮点累加精度(实际无碍,RAF 帧率下)。

---

← [modes-reasoner.md](modes-reasoner.md) | [走读索引](index.md) | 下一章:[06 数据模型](../06-data-model.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)