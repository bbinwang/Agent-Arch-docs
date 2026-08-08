# 走读:actions.py(155 行)

> 动作决策器:用一个**独立的 grammar-forced JSON 请求**判断「用户这一轮要求助手做什么」(定时器 / 模式 / 研究),取代带内(in-band)控制标签。这是 v2「动作与语音解耦」的核心。

## 文件结构概览

```
actions.py
├─ 模块 docstring(1–26)   为何用第二个请求 + 为何 head 必须在同一 llama-server
├─ prompt 构件(39–83)    _HEAD_COMMON / _MODE_CLAUSES / HEAD_SCHEMA
├─ ActionDecision(86–97)  冻结数据类 + NONE 单例
├─ decide_after(100–108)  回复后判定
├─ decide_before(111–122) 回复前判定(listen/translate)
└─ _decide(125–155)       单次 grammar head 调用 + 解析
```

---

## 一、为什么用第二个请求(模块 docstring)

docstring 给出 `benchmarks/archbench.py` 的测量(e4b,19 个口语案例 ×2):
- **带内标签**:召回 0.955,漏判是「ack-without-action」(「I will be quiet」无标签)——一个**服务端从不兑现的口头承诺**,语音助手最糟的失败。
- **解耦 head**:召回 **1.0**,唯一错误是一个可取消的多余定时器误触发(0.062)。

结构优势(为什么解耦 head 天生更好):
1. **head 永远不会泄漏进语音**(没有要切除的东西)。
2. head 在 **temperature 0** 跑,而语音保持采样温度。
3. head 把模型自己的确认当**证据**(「three minutes, I'll let you know」正是 head 锁定的)。
4. head 自己做时长数学(「twenty minutes」→1200),任意语言。
5. **历史保持纯语音**——没有存储的标签能教会模型一个服务端没有的状态。

**成本**:head 在缓存前缀上解码 ~35 JSON token(e4b ~2s GPU,隐藏在 TTS 播放下),每轮。一个更便宜的 1-token yes/no 预门曾测量(archbench B_gated):召回完美 1.0,但几乎每轮都答「yes」(连「how are you」),所以加 ~0.9s 后还得跑 head——不值,除非实际 GPU 成本成问题。

**关键约束**:head 必须跑在**同一个 llama-server**——单独模型每轮要全量 prefill 历史 + 音频,共享前缀缓存才让判定便宜。

---

## 二、Prompt 构件(39–83)

### `_HEAD_COMMON`(39–53)
```
System note (not user audio): you are the action decider. From the user's last
message (the assistant's reply above may help), report what the user asked the
assistant to DO, as JSON. timer_seconds: ... timer_label: ... {mode_clause}
research_task: ... A duration or topic merely mentioned in passing is NOT a
request: report an action only when the user asked for it.
```
要点:
- 「System note (not user audio)」防模型把纯文本轮误读成回声(同 server.py 的 DELIVER_PROMPT)。
- 明确「仅提及的时长/话题不是请求」——只有用户**要求**了才报告动作。
- 「asking the assistant to be quiet for some time is a mode request, never a timer」——防把「安静一会儿」误判为定时器。

### `_MODE_CLAUSES`(54–72)
每种模式一个 clause,精确描述「no change 是什么」「exit 命令长什么样」:
- **conversation**:mode 是要**切换到**的(translate/listen);已在对话中所以默认 none。
- **translate**:只有命令「停止翻译回到正常对话」(任意语言)才是 conversation;其余是被翻译的内容。
- **listen**:只有明确要求「重新开始回应」(「okay I'm done」「what do you think?」「stop listening」)才是 conversation;**仍在自言自语(即便提到助手)仍是 none**——e2e 测出的措辞(「the thing I wanted to ask you about is…」仍是思考)。

### `HEAD_SCHEMA`(73–83)
JSON Schema,经 llama-server 编译为 grammar:
```python
{"type":"object","properties":{
    "timer_seconds":{"type":"integer"},
    "timer_label":{"type":"string"},
    "mode":{"type":"string","enum":["none","translate","listen","conversation"]},
    "research_task":{"type":"string"},
},"required":[...]}
```
**结构上保证可解析**——head 输出必符合 schema。

---

## 三、`ActionDecision`(86–97)

```python
@dataclass(frozen=True)
class ActionDecision:
    timer: tuple[int, str] | None = None   # (seconds, label)
    mode: str | None = None                # 目标模式,已 ≠ 当前
    research: str | None = None            # 自包含任务
    def any(self) -> bool: return bool(self.timer or self.mode or self.research)

NONE = ActionDecision()   # 「无动作」单例
```
冻结、可空字段。`any()` 判是否有任何动作。

---

## 四、`decide_after` / `decide_before`

### `decide_after(messages, current_mode)`(100–108)
判断**已完成的对话轮**(阻塞,跑 executor)。`messages` 以 assistant 回复结尾,与轮请求逐字节相同 + 回复——**整个前缀已在 slot 缓存**,所以只有 decider prompt 付 prefill。回复是证据。

### `decide_before(history, content, current_mode)`(111–122)
判断**回复尚不存在**的 utterance(阻塞)。用于 translate/listen(回复是什么取决于决策:要渲染的内容 vs 对助手的命令)。decider prompt **骑在同一 user 消息上**作为音频——生产指令形状,且扩展已预填的前缀而非开第二个 user turn(chat template 不喜连续同 role 消息)。

---

## 五、`_decide`(125–155)——单次 grammar head 调用

```python
def _decide(build, current_mode) -> ActionDecision:
    try:
        head_prompt = _HEAD_COMMON.format(mode_clause=_MODE_CLAUSES[current_mode])
        raw = llama.chat_blocking(build(head_prompt), max_tokens=192, temperature=0.0, json_schema=HEAD_SCHEMA)
        head = json.loads(raw)
    except Exception as e:
        print(f"Action decider failed: {e}")
        return NONE    # 丢失的判定 = 无操作轮,绝不抛进会话循环
    timer = None
    seconds = head.get("timer_seconds") or 0
    if isinstance(seconds, int) and seconds > 0:
        timer = (seconds, str(head.get("timer_label", "")).strip())
    target = head.get("mode")
    # translate/listen 中唯一允许的转换是「出去」
    allowed = (("translate","listen","conversation") if current_mode=="conversation" else ("conversation",))
    mode = target if target in allowed and target != current_mode else None
    research = str(head.get("research_task","")).strip() or None
    decision = ActionDecision(timer=timer, mode=mode, research=research)
    if decision.any(): print(f"Action decision: {decision}")
    return decision
```

关键点:
1. **失败返 NONE**:任何异常(网络 / json 解析)→ NONE。注释「丢失的判定是无操作轮,绝不抛进会话循环」——会话鲁棒性。
2. **`max_tokens=192`**:必须清 JSON 骨架(~30 token)**外加**长的重述研究任务——截断 JSON 会让 `json.loads` 失败、静默丢失一个回复已承诺的动作(review finding)。grammar 约束形状,所以上限给得宽裕。
3. **`temperature=0.0`**:确定性(与套件一致;生产语音仍 0.7)。
4. **mode 合法性**:translate/listen 中只允许 `conversation`(出去);其他 target 是误读(幻觉 listen→translate 跳跃会在 listen 中途朗读回答)。
5. **`timer_label` 默认空**:server 端 `spawn_timer` 兜底为 "timer"。

---

## 复杂度与改进点

| 项 | 说明 |
|----|------|
| 单次 HTTP | O(1) 请求;延迟 = prefill(decider prompt)+ ~35 token 解码 |
| 复用缓存 | 因前缀已缓存,只付 prompt 增量 |

改进点:
1. **每轮都跑 head**:即便显然无动作(「hello」)。注释提到 1-token 预门被测量但不值——若实际 GPU 成本显现可再评估。
2. **`max_tokens=192` 上限**:超长任务仍可能截断。grammar 保证形状但内容可能不完整。
3. **温度 0 的固有限制**:production 召回在 0.7 下由 `tagbench --production` 验证,非套件职责——但生产偶发漏判理论上存在。
4. **mode 合法性白名单硬编码**:`allowed` 元组逻辑可抽到 modes.py 与 Mode 数据关联,提升一致性。

---

← [llama.md](llama.md) | 下一个:[turn_detector.md](turn_detector.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕