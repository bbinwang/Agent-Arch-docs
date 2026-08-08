# 走读:server.py(860 行)

> 系统编排器。FastAPI app、所有 prompt 常量、会话状态、轮次主循环、模式 / 定时器 / 委派 / 历史轮转全部集中于此。它是体量最大、策略最密集的模块——这是有意设计:**对话策略可在一个文件里完整追溯、便于 benchmark**。

## 文件结构概览

```
server.py
├─ 模块 docstring + imports + 全局配置(line buffering)
├─ PROMPT 常量区(52–246)   SYSTEM_PROMPT / CAPABILITY_NOTE / RESEARCH_NOTE
│                            / TRANSLATE_PROMPT / LISTEN_PROMPT / TIMER_*
│                            / DELIVER_* / RESPOND/FLUSH_PROMPT / NO_SPEECH_CLAUSE ...
├─ 工具函数(141–286)        duration_phrase / elapsed_phrase / rotate_history
├─ 全局对象 + 模型加载(288–312) tts_backend / detector / load_models / lifespan
├─ FastAPI app + 路由(312–330) / /static / ws 端点
├─ turn_instruction(322–330)
├─ websocket_endpoint(332–848) ★ 会话循环(核心)
└─ main(851–860)             uvicorn 启动
```

---

## 一、Prompt 常量区(52–246 行)

这是理解 Parlor「行为」的钥匙。每个常量都带大段注释解释**为什么这么措辞**(多为测量结论)。重点逐个讲:

### `SYSTEM_PROMPT`(52–58)
基础人设:友好、对话式、语音输出不格式化;**关键条款**——「如果音频只是你自己上一句回复在回放(回声),别回答它,简短问想聊什么」。这是回声防御的最后一道(prompt 层),配合浏览器的 barge-in 滑窗门控。

### `CAPABILITY_NOTE`(68–74)
让模型**知道**有定时器 / 翻译 / 只听能力(这样口头确认才自然、不与决策器矛盾),但**明确**「实际执行由另一个系统负责,所以永远别说你做不到、也别编造它的结果」。放系统提示是因为它不改变逐轮行为(不像旧带内标签要和音频争注意力)。

### `RESEARCH_NOTE`(83–92)——「承重」注释
仅在配置 reasoner 时追加。注释特别强调末尾「everything else, answer yourself」是**承重的**:历史里一旦出现一次「我去查」,模型就开始把稳定通用知识问题(「法国首都」)也推诿——两次 e2e 迁移失败都是这个。所以必须明确划界:只有「当前 / 变化类」才推诿。

### `TRANSLATE_PROMPT` / `LISTEN_PROMPT`(97–116)
每轮指令(替代默认 respond/flush)。都用 `###TRANSCRIPT: ... \n 回复` 格式。translate 只输出英语翻译不评论;listen 只转写不回复。**退出检测不在这些 prompt 里**——由 action decider 在 prompt 运行前判定,保持 prompt 纯粹。

### `TIMER_*` / `DELIVER_*`(126–171)
服务端发起轮(proactive turn)的 prompt。都以「System note (not user audio)」开头——因为模型偶尔会把纯文本轮误读成「自己上一句在回放」而给反回声响应(实测)。`DELIVER_PROMPT` 要求逐字转述答案、永不丢改名字/数字/地名。`TIMER_FALLBACK` / `DELIVER_FALLBACK` 在模型产出无可朗读内容时兜底。

### `NO_SPEECH_CLAUSE`(193–197)——最被反复测量的条款
给模型一个** sanctioned out**(认可的退路):无清晰词(噪声/呼吸/静默)时写 `###TRANSCRIPT: (no speech)` 而非瞎猜。注释记录:没这条时,prompt **索要**词,于是在无声音频上模型**发明**词——temp 0.7 下,一声呼吸变成「Hi, can you help me with my homework?」(还回答了)、「can you translate everything I say?」(竟切换了模式!)、或逐字复制上一轮问题。措辞「this message's audio」(不是「their audio message」)把转写指向历史中众多 clip 里最新的那个。

### `RESPOND_PROMPT` / `FLUSH_PROMPT`(199–229)
默认对话轮与 flush 轮指令。都以 `###TRANSCRIPT:` 行开头(leading transcript)。FLUSH_PROMPT 是**专用** prompt(而非给 RESPOND_PROMPT 加后缀)——注释:加后缀时模型有时只输出 transcript 行就停,让轮静默。

### 时间感:`elapsed_phrase` / `duration_phrase` / `SESSION_CLOCK` / `TIME_NOTE`(141–262)
模型无时钟。`SESSION_CLOCK` 在连接时格式化一次(会话开始时间);`elapsed_phrase` 把安静时长转成**粗粒度**自然短语(「a few seconds / about 20 seconds / about a minute / about N minutes / about an hour」)——故意粗化,精确数字会被模型像数据一样生硬复述。`TIME_NOTE_MIN_S`(默认 120s)控制触发阈值。

### `CONTEXT_HEADROOM` / `rotate_history`(265–286)
见 [§历史轮转 3.8](../03-flows.md)。`CONTEXT_HEADROOM = max(512, min(2000, CTX//8))` 随上下文缩放。`rotate_history` 保留系统提示 + 最近 3/4,且强制落在 user 消息上。

---

## 二、全局对象与模型加载(288–312)

```python
tts_backend = None
detector = None
REASONER_POOL = ThreadPoolExecutor(max_workers=4, thread_name_prefix="reasoner")
```

- `tts_backend` / `detector` 是**全局共享**对象(所有会话复用),在 `load_models()` 一次性加载并预热。
- `REASONER_POOL` 是**专用 4 线程池**,与默认 executor 分离。注释:reasoner 调用阻塞最多 `REASONER_TIMEOUT`,不能占用服务延迟敏感路径(llama 流式 / TTS / turn 分类 / 缓存预热)的默认 executor 线程。

### `load_models()`(297–302)
顺序:`llama.start()` → `TurnDetector()`(含 warmup)→ `tts.load()`(含 warmup)。任一失败会冒泡(在 `lifespan` 里被 executor 跑,异常会让启动失败)。

### `lifespan`(305–309)
FastAPI lifespan 上下文管理器:启动时 `run_in_executor(load_models)`(阻塞加载不卡事件循环),`yield` 服务,退出时 `llama.stop()` 终止子进程。

---

## 三、FastAPI app 与路由(312–330)

```python
app = FastAPI(lifespan=lifespan)
app.mount("/static", StaticFiles(.../web/static), name="static")

@app.get("/")
async def root():  # 读 index.html,替换 {{model}} → llama.model_label()
    ...

@app.websocket("/ws")
async def websocket_endpoint(ws): ...  # ★ 核心
```

- `root()` 读 `web/index.html` 文本,把 `{{model}}` 替换为 `model_label()`(如 "Gemma 4 E4B"),返回 HTMLResponse。这是唯一的模板渲染。
- `/static` 挂载静态目录(app.js / style.css)。

---

## 四、`turn_instruction`(322–330)

```python
def turn_instruction(msg, has_image, has_audio) -> str:
    if has_audio:
        camera = CAMERA_CLAUSE if has_image else ""
        prompt = FLUSH_PROMPT if msg.get("type") == "flush" else RESPOND_PROMPT
        return prompt.format(camera=camera)
    if has_image:
        return "The user is showing you their camera. Describe what you see."
    return msg.get("text", "Hello!")
```

四象限决策:有音频→RESPOND/FLUSH_PROMPT(带 camera clause);无音频有图→纯视觉描述;都无→文本(text turn)。**这是「这一轮是什么」的默认决策**,但 translate/listen 模式会在主循环里用 `MODE_PROMPTS` 覆盖它。

---

## 五、`websocket_endpoint`(332–848)——★ 会话循环

这是系统心脏。**每个 WebSocket 连接 = 一个独立协程 = 一份独立会话状态**。结构是「初始化状态 → 启动 receiver 任务 → while 主循环分发消息」。

### 5.1 初始化(334–400)

```python
await ws.accept()
delegation = reasoner.enabled()
clock = time.strftime("%A at %I:%M %p").replace(" 0", " ")
system = SYSTEM_PROMPT + SESSION_CLOCK.format(clock=clock) + CAPABILITY_NOTE + (RESEARCH_NOTE if delegation else "")
history = [{"role": "system", "content": system}]
mode = MODES["conversation"]
interrupted = asyncio.Event()
active = {"stream": None}
msg_queue = asyncio.Queue()
```

**系统提示在连接时一次性拼装**——这是前缀缓存的前缀,后续每轮都重发它,所以它必须稳定。`delegation` 决定是否拼 `RESEARCH_NOTE`。

随后是一组**会话状态闭包变量**(368–400),用 dict/list 包裹以便嵌套函数修改:

| 变量 | 含义 |
|------|------|
| `frame_image` | 当前 utterance 持有的摄像头帧(已预填) |
| `speech_chunks` | 已流式送达、已缓存预填的语音块 |
| `held_audio` | 未完成轮(incomplete)暂存的音频段 |
| `delegation_ids` / `delegation_tasks` | 研究任务 id 计数器 / 进行中任务集合 |
| `ready_events` | floor 忙时「已完成但待投递」的后台事件队列 |
| `timer_ids` / `pending_timers` | 定时器 id 计数器 / {id: ring 任务} |
| `playing_since` | 最近一句回复「可能还在播放」的时间戳 |
| `last_activity` | 线路最后活跃时间(对话结束或用户说话流式) |
| `prompt_tokens` | 真实上下文大小(来自 llama-server usage) |

### 5.2 `receiver()` 协程(347–365)

后台任务,持续 `ws.receive_text()`:
- `type == "interrupt"` → `interrupted.set()` + `active["stream"].cancel()`(真正中止生成)。
- 否则 `msg_queue.put(msg)`。
- `DISCONNECT_ERRORS` → pass;`finally` 永远 `msg_queue.put(None)` 解除主循环阻塞(即便意外错误)。

**单生产者(msg_queue)、单消费者(主循环)** 模型:receiver 把所有客户端消息塞进队列,主循环逐个处理。后台事件(timer_done / delegation_done)也通过这个队列注入,保证**投递与真实轮次串行化**。

### 5.3 一组嵌套辅助函数

#### `remember(user_msg, raw_text, no_speech)`(372–388)
存一轮进历史。**两类永不入历史**:(a) 无内容;(b) no_speech。注释极详细:退化消息毒化后续每个请求;且语音轮必须保留原始音频(实验:存转写文本会让模型在 temp 0 确定性地把上一轮文本复制为新轮 transcript)。

#### `floor_busy()`(402–410)
判断「楼层」是否被占用:用户在 hold / 语音块流式 / barge-in 中,或我们的语音还在播放(`playing_since` 30s 内)。30s 陈旧逃生口防止丢失 `ready` 搁浅结果。

#### `drain_ready()`(412–426)
floor 空闲时,从 `ready_events` 取一个事件重新入队。**扫描而非 pop 队首**:定时器任何模式都响,研究结果要等翻译/只听结束,所以一个搁置的研究答案不能挡住后面的定时器。

#### `switch_mode(name)`(428–452)
按名进模式。未知 / 已是当前 → no-op。切换清空 `frame_image/speech_chunks/held_audio`(防陈旧 held 让 floor_busy 永真),发 `mode_changed`,`drain_ready()`。

#### `run_delegation(task_id, task)`(454–471)
后台 reasoner 调用。`run_in_executor(REASONER_POOL, reasoner.ask, task)`,答案 >1500 截断,异常 → `ok=False`。结果 `msg_queue ← delegation_done`。

#### `spawn_delegation(task)`(473–492)
守卫:`delegation and mode.allows_delegation`;任务 ≥3 词;并发 ≤3。发 `delegation_started`,create_task。

#### `proactive_turn(prompt, fallback)`(494–514)
服务端发起轮(委派投递 / 定时器响)。`run_turn(..., proactive=True, fallback=...)`。proactive 标志让 transcript 行不发给客户端、`turn_final` 带 `proactive=True`。`remember` 入历史。**服务端发起轮永不做动作**——decider 只判真实用户轮。

#### `deliver_delegation(done)`(516–529)
`interrupted.clear()` → 发 `delegation_resolved` → `proactive_turn(DELIVER_PROMPT 或 DELIVER_FAILED_PROMPT, fallback)`。

#### `run_timer` / `spawn_timer` / `deliver_timer`(531–590)
定时器三件套(见 [§3.6](../03-flows.md))。`spawn_timer` 守卫 ≤8h、≤3 并发。

#### `apply_decision(decision)`(555–579)
按「转换两端都触发」策略执行动作(见 [§3.5](../03-flows.md))。conversation: 先 spawn timer/research 再 switch;从 translate/listen 退出:先 switch 再 spawn(保持 allows_delegation 门控)。

#### `prime(audio_b64s)`(592–596)
`prime_cache(history + [user_content(frame_image, audio_b64s)])`。读实时 history 与 held frame,所以必须留在此闭包作用域。

### 5.4 主循环 `while True`(598–848)

每轮 `msg = await msg_queue.get()`,`None` → break。分发逻辑(按消息类型):

#### (a) 历史轮转(604–619)
每次取消息先估算 token,超 `CTX - 2*HEADROOM` 则 `rotate_history`。用真实 `prompt_tokens["last"]` 盖过估算。

#### (b) `ready`(621–631)
客户端回到空闲监听(播放结束 / 误 barge-in 未成 utterance)。清 `playing_since`、`interrupted.clear()`、`drain_ready()`。**粘性 interrupted 不能搁浅排队投递**。

#### (c) `set_mode`(633–638)
UI 逃生舱(模式芯片 stop),直接 `switch_mode`。

#### (d) `delegation_done`(640–662)
研究完成。若翻译/只听 → `delegation_parked` + 存 `ready_events`;floor 忙 → 存。否则 `deliver_delegation`(异常 try/except 保留会话,可重试一次)。

#### (e) `timer_done`(664–678)
到点。已取消 → continue;floor 忙 → 存;否则 `deliver_timer`。

#### (f) `cancel_timer`(680–692)
取消 sleep 任务 + 清 ready_events,发 `timer_resolved(cancelled)`。

#### (g) `frame`(694–699)
若 `mode.wants_camera` 且有图 → 存 `frame_image`,清 `speech_chunks`,`prime(held_audio)`。

#### (h) `speech_chunk`(701–708)
seq==0 重置;`valid_audio` 校验 → 追加,更新 `last_activity`,`prime(held_audio + speech_chunks)`。

#### (i) 真实轮(710–840)——★ 核心
```python
interrupted.clear()
audio_b64s = held_audio + (speech_chunks if msg.get("chunked") else [])
speech_chunks = []
if valid_audio(msg.get("audio")): audio_b64s.append(msg["audio"])
image = (msg.get("image") or frame_image) if mode.wants_camera else None
has_audio = bool(audio_b64s)
is_flush = msg.get("type") == "flush"
```

随后一个 `try`:
1. **smart-turn 门控**(737–755):`has_audio and not is_flush and mode.uses_smart_turn` → `detector.predict(pcm)`。未说完 → `held_audio = audio_b64s`,发 `turn_incomplete`(先于慢预热释放客户端),`prime(held_audio)`,`continue`。
2. **补尾静音**(760–764):`pad_tail_silence(末段)`。
3. **构建 content + instruction**(762–796):
   - listen + audio → **pre_decide**(`decide_before`);若 decision.mode(退出问句)→ 用 RESPOND_PROMPT 当普通对话轮回答。
   - translate/listen + audio → `MODE_PROMPTS`。
   - 否则 → `turn_instruction(msg, image, has_audio)`。
   - 时间注记:`has_audio and mode.wants_time_note and gap >= TIME_NOTE_MIN_S` → 追加 `TIME_NOTE`。
   - fallback 策略(797–806):exit 轮 → AUDIO_FALLBACK;listen(`not speaks_fallback`)→ None;flush → FLUSH_FALLBACK;有音频 → AUDIO_FALLBACK;否则 None。
4. **run_turn**(808–813):`run_turn(ws, history+[user_msg], ..., expect_transcript=has_audio, p_complete, tts_voice=mode.tts_voice, fallback)`。返回 `(raw_text, pt, no_speech, spoke)`。
5. **decision**(814–830):非 pre_decided 且非 no_speech 且非 interrupted → `decide_after(history+[user_msg]+[assistant reply])`(隐藏在 TTS 下)。no_speech/interrupted → `NONE`。`apply_decision`。
6. `remember`;`last_activity` 更新。
7. `except DISCONNECT_ERRORS: raise`;`except: traceback + release_client`。
8. `drain_ready()`。

#### `finally`(843–848)
`recv_task.cancel()`,取消所有 delegation / timer 任务。**连接断开 = 会话彻底销毁**,无持久化。

---

## 六、`main()`(851–860)

```python
def main():
    port = int(os.environ.get("PORT", "8000"))
    uvicorn.run(app, host="localhost", port=port)
```

**host="localhost" 而非 0.0.0.0**:注释——浏览器把 `http://localhost` 当安全上下文,但不当 `http://0.0.0.0`,而 getUserMedia(麦克风/摄像头)在非安全上下文不存在。`[project.scripts] parlor = "parlor.server:main"` 暴露为 `uv run parlor`。

---

## 设计模式与潜在改进点

| 模式 | 体现 |
|------|------|
| **会话即协程** | 每连接一个协程 + 闭包状态,天然隔离,无锁 |
| **生产者-消费者队列** | receiver → msg_queue → 主循环,串行化所有事件 |
| **数据即配置** | `modes.py` Mode dataclass,server 查标志位 |
| **测量驱动 prompt** | 几乎每个 prompt 常量都注释测量结论 |
| **defense-in-depth 兜底** | no_speech / echo / 退化音频 / 坏 WAV 多层防线 |

潜在改进点:

1. **`websocket_endpoint` 函数过长(~516 行)**:嵌套闭包多,可读性受影响。可考虑抽成一个 `Session` 类,把闭包变量变实例属性、嵌套函数变方法。但当前闭包形式让「会话状态」作用域清晰,且测量驱动改动集中,重构需谨慎。
2. **全局可变对象**(`tts_backend`/`detector`):非线程安全共享,但在单事件循环 + executor 模型下可控。若未来多 worker 需重新设计。
3. **prompt 硬编码**:所有 prompt 在源码常量里,无外部化机制(如加载自文件)。改动 prompt 需改代码重启。
4. **无指标/可观测性**:仅 `print` 日志,无 metrics/prometheus。生产化可加。

---

← [走读索引](index.md) | 下一个:[pipeline.md](pipeline.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)