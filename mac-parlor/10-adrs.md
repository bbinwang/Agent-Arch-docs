# 10 架构决策记录(ADR)

> Parlor v2 的每个关键决策都**由测量驱动**。本章以 ADR(Architecture Decision Record)形式记录决策的**背景、选项、决定、依据(benchmark / 观测)**。所有依据均可追溯到 `benchmarks/` 或源码注释。

---

## ADR-001:采用经典级联架构,而非端到端全双工模型

- **状态**:Accepted(v2 基础)
- **背景**:目标是单机本地对齐 GPT-Live。GPT-Live 是全双工实时语音模型。
- **选项**:
  - A. 微调 Gemma 4 12B 全双工化(嫁接决策 tick + 语音头)。
  - B. 经典级联(VAD → 端点分类器 → LLM → TTS)。
- **决定**:选 B。
- **依据**:选项 A 经多次试验**失败**(README "Why?")。作者结论:在端侧全双工模型出现前,级联更优;且级联每阶段可独立测量、替换、打补丁。
- **后果**:多阶段延迟叠加需用 overlap prefill / 流式管道弥补(见 ADR-002/003);换来可维护性与可测量性。

---

## ADR-002:推理后端用 llama.cpp(替换 v1 的 LiteRT-LM)

- **状态**:Accepted(v2.0.0,breaking change)
- **背景**:v1 用 LiteRT-LM。v2 需要前缀缓存做 overlap prefill。
- **决定**:迁移到 llama.cpp(`llama-server` 子进程 + OpenAI 兼容 API + `cache_prompt`)。
- **依据**(CHANGELOG v2):
  - 前缀缓存解锁「说话重叠式预fill」——音频块与摄像头帧在用户说话时就推过缓存,长问与短问首音几乎持平。
  - 延迟大幅下降:短问 1.5s→**0.7s**,长问 2.9s→**1.3s**(e2b,M3 Pro)。
  - Gemma 4 官方 QAT q4_0 GGUF 质量与速度优于 K-quants。
- **后果**:需强制 llama.cpp 构建门槛(b9503+/b9512+);`llama.py` `check_build` 前端校验。LiteRT-LM 后端移除。

---

## ADR-003:轮次结束判定用 smart-turn-v3 分类器,而非 LLM

- **状态**:Accepted(v2.0.0,替换 LLM FINISHED/WAIT)
- **依据**(`benchmarks/turnbench.py`,标注人类语音):
  - smart-turn-v3(8M 参数):**准确率 0.96,@19ms**。
  - 每个 Gemma 变体:接近随机,且耗时 **0.6–3.6s**。
- **决定**:用专用小分类器;LLM prompt 完全不带 FINISHED/WAIT 机制(server.py:48–51 注释)。
- **后果**:思考中途停顿被 smart-turn hold(等续说或 flush);退化音频下分类器也可能误判,故有 hold/flush + 2.5s flush 计时器兜底(ADR-005)。`TURN_MODE` 配置移除。

---

## ADR-004:动作判定用解耦 JSON head,而非带内控制标签

- **状态**:Accepted(v2.0.0)
- **背景**:定时器 / 模式 / 研究如何触发?
- **选项**:
  - A. 带内标签(`<timer>3min</timer>` 嵌语音,服务端切除)。
  - B. 独立 grammar-forced JSON 请求(decoupled head)。
- **依据**(`benchmarks/archbench.py`,e4b,19 口语案例 ×2):
  - 标签:召回 **0.955**;漏判是「ack-without-action」(「I will be quiet」无标签)——**服务端从不兑现的口头承诺,语音助手最糟的失败**。
  - head:召回 **1.0**;唯一错误是可取消的多余定时器(0.062)。
  - 结构优势:head 不泄漏进语音、temp 0 跑、把模型确认当证据、自己做时长数学(任意语言)、历史保持纯语音。
- **成本**:head 每轮 ~35 JSON token(~2s GPU e4b),隐藏在 TTS 下。1-token 预门曾测量(archbench B_gated):召回 1.0 但几乎每轮 yes(连 "how are you"),不值(ADR-011)。
- **决定**:选 B。head 必须跑**同一** llama-server 以复用前缀缓存(actions.py 顶部注释)。
- **后果**:控制标记永不泄漏 TTS(结构不可能);口头承诺总被兑现。

---

## ADR-005:未说完的轮次 hold + flush,而非立即回答或无限等

- **状态**:Accepted
- **背景**:smart-turn 判 `complete=False`(思考停顿)时怎么办?
- **决定**:hold 音频段(`held_audio`),发 `turn_incomplete`,客户端续听 + 2.5s flush 计时器;用户续说则合并,静默则 flush 用 FLUSH_PROMPT 回答已有内容。
- **依据**:hold 段留在下次请求内容**且留在缓存**(已 prime),续说无缝;flush 用专用 prompt(加后缀会让模型只输出 transcript 行就停,server.py:222 注释)。flush 计时器自限制(一次 hold 一次 flush、不冲实时语音)。
- **后果**:翻译/只听模式 `uses_smart_turn=False`,不走此分支。

---

## ADR-006:transcript 行 LEADS 回复(leading,非 trailing)

- **状态**:Accepted
- **依据**(server.py:177–192 注释):
  - trailing(回复后转写):转写变成「凭记忆改写」,WER **0.39**。
  - leading(先转写再回复):WER **0.00**(干净 33 词 utterance)。
  - leading 还让转写在回复仍解码时就上屏(~0.3s 内),给用户即时反馈。
- **成本**:leading 付其解码时间(~0.2s 短 / ~0.7s 长 utterance)于首音前——测量值得。
- **备选**:grammar-forced JSON `{transcript, response}` 也测量过:退化音频下格式 1–3/3 破坏、chunked 3/3 破坏——**不回结构化输出**。
- **决定**:leading `###TRANSCRIPT:` 行 + 容错 `StreamParser`。

---

## ADR-007:定时器时钟由服务端持有,而非模型

- **状态**:Accepted
- **依据**(`benchmarks/timerprobe.py`):**turn-based 模型无法在静默中响起**——它只在被请求时生成,沉默期什么都不做。
- **决定**:定时器是服务端 `asyncio.sleep` 任务;到点经 `msg_queue` 串行化投递,`proactive_turn` 用模型朗读提醒(任意模式都响)。
- **后果**:需 floor 调度(不打断自己)、parking(翻译/只听中暂存)、并发与上限守卫(`MAX_PENDING_TIMERS=3`、`MAX_TIMER_S=8h`)。

---

## ADR-008:No-speech 条款 + 标注形状拒绝

- **状态**:Accepted
- **背景**:无声音频(呼吸/咳嗽/房间噪声)如何处理?
- **依据**(server.py:185–192 注释,temp 0.7 实测):无 sanctioned out 时 prompt **索要**词,模型在无声音频上**发明**词——一声呼吸变成:
  - "Hi, can you help me with my homework?"(还回答了)
  - "can you translate everything I say from now on?"(竟切换模式!)
  - 或逐字复制上一轮问题。
- **决定**:prompt 给 `###TRANSCRIPT: (no speech)` sanctioned out;`NO_SPEECH_RE` 拒绝标注形状的转写(`(noise)`/`[Silence]`/`*sigh*`);`echoes_instruction` 拒绝指令回声。no_speech 轮**永不入历史、永不执行动作**。
- **后果**:非语音不再变成捏造用户词 / 自答 / 切换模式。

---

## ADR-009:浏览器侧裸 TTS 输出,绕过系统回声消除

- **状态**:Accepted
- **依据**(app.js:683–690 注释):曾试经 `MediaStreamDestination + <audio>` 让 Chrome 回声消除拿参考信号(crbug 687574),但 macOS 上启用系统语音处理——**每轮给 TTS 音色着色**且播放时压制用户麦(杀 barge-in)。
- **决定**:裸输出到 WebAudio destination(音色干净 + 麦诚实);回声防御靠:持续语音 barge-in 滑窗门控(ADR-010)+ TTS 后 800ms grace + prompt 回声规则。戴耳机即无问题。
- **后果**:扬声器外放仍有回声风险(已多层缓解,非根除)。

---

## ADR-010:Barge-in 用滑窗投票,而非连续帧计数

- **状态**:Accepted
- **依据**(app.js:507–516 注释):barge-in 须**持续**(最近 10 帧 ≥6 帧 `isSpeech>0.85`),而非单帧——否则麦收到自己 TTS 就打断(回声)。但真实语音在辅音/词边界下陷,**连续帧计数器在真麦上永不触发**;滑窗能。
- **决定**:滑窗投票 + 800ms echo grace + 服务端 `stream.cancel()` 真正中止 + 客户端 `ignoreIncomingAudio` + phantom capture 看门狗(误触重置)。
- **后果**:barge-in 在真麦上可靠触发。

---

## ADR-011:Action head 每轮都跑,不加 1-token 预门

- **状态**:Accepted(可重评估)
- **依据**(`benchmarks/archbench.py` B_gated):1-token yes/no 预门召回完美 1.0,但几乎每轮答 yes(连 "how are you"),所以加 ~0.9s 后还得跑 head——不值其复杂度。
- **决定**:每轮直接跑 head,隐藏在 TTS 播放下。
- **重评估条件**:若 per-turn GPU 成本在实践中显现。

---

## ADR-012:历史轮转用真实 prompt_tokens,而非估算

- **状态**:Accepted
- **背景**:早期版本用固定 2000 阈值,在小 `LLAMA_CTX` 下几乎为零阈值,每轮都轮转——历史永不累积,e2e 多轮测试被静默跳过。
- **决定**:`used = max(estimate_tokens(history), prompt_tokens["last"])`,真实计数(来自 llama-server usage)盖过估算;`CONTEXT_HEADROOM` 随 CTX 缩放;丢 1/4 而非一半;强制落在 user 消息。
- **依据**:估算会漂移(尤其摄像头轮),llama-server 会静默截断最旧轮(读作模型「遗忘」)。server.py:604–619 注释。

---

## ADR-013:仅监听 localhost(非 0.0.0.0)

- **状态**:Accepted
- **依据**(server.py:853–855 注释):浏览器把 `http://localhost` 当安全上下文,但不当 `http://0.0.0.0`;无安全上下文 `getUserMedia`(麦/摄像头)不存在。
- **决定**:`host="localhost"`。
- **后果**:天然不暴露公网;远程访问需自加反代 + TLS + 鉴权。

---

## ADR-014:补 0.3s 尾静音修复 VAD 截断的词尾

- **状态**:Accepted
- **依据**(pipeline.py:26–29 注释):VAD 在截止处硬切的音频让编码器**幻觉式自信补全**最后一个词;补一段静音修复。必须补在**同一 WAV 内**(单独静音 part 无效)。
- **决定**:`pad_tail_silence`(TAIL_SILENCE_S=0.3)补末段。

---

## ADR-015:模式是数据(Mode dataclass),不是行为

- **状态**:Accepted
- **依据**(modes.py 顶部注释):server.py 查标志位而非硬编码 `if mode.name==...`,未来模式是新 Mode + 触发,非重写会话循环。
- **决定**:6 标志位 `Mode` dataclass;`MODES` 字典。
- **后果**:translate/listen 只需定义数据;e2e 测出的细粒度策略(listen 不朗读 fallback、不打断翻译)用标志位组合表达。

---

## 决策矩阵速览

| ADR | 决策 | 核心依据 |
|-----|------|---------|
| 001 | 级联 > 全双工 | 微调全双工失败 |
| 002 | llama.cpp | 前缀缓存,延迟 1.5s→0.7s |
| 003 | smart-turn | 0.96 @19ms vs LLM 随机 |
| 004 | 解耦 JSON head | 召回 1.0 vs 标签 0.955 |
| 005 | hold + flush | 思考停顿 |
| 006 | transcript leading | WER 0.00 vs 0.39 |
| 007 | 服务端持时钟 | turn-based 无法静默响 |
| 008 | no-speech 条款 | 防捏造词/自答/切模式 |
| 009 | 裸 TTS 输出 | 系统处理着色+杀麦 |
| 010 | 滑窗 barge-in | 真麦辅音下陷 |
| 011 | 无预门 | 预门每轮 yes 不值 |
| 012 | 真实 token 轮转 | 估算漂移致「遗忘」 |
| 013 | localhost only | 安全上下文 |
| 014 | 尾静音 | 修 VAD 截断幻觉 |
| 015 | 模式即数据 | 可扩展 |

---

下一章:[11 开发者上手指南](11-developer-guide.md)

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)