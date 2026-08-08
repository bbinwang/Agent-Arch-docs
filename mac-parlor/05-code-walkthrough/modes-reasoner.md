# 走读:modes.py(50 行)+ reasoner.py(77 行)

两个体量小但设计精巧的模块:**模式数据**(modes.py)与**后台研究**(reasoner.py)。

---

## Part A:modes.py —— 会话模式即数据

> 会话模式决定轮次如何被门控与提示。默认是对话;translate 让 Parlor 变同声传译;listen 让它变静默速记。**模式是数据不是行为**:server.py 查这些标志位而非硬编码,所以未来新模式(传译对、摄像头旁白)只是这里加一条 + 触发逻辑,不是重写会话循环。

### `Mode` 数据类(19–27)

```python
@dataclass(frozen=True)
class Mode:
    name: str
    uses_smart_turn: bool     # 声学完整性门控 + hold/flush
    allows_delegation: bool   # 后台 reasoner 标签与投递
    wants_camera: bool        # 接受 + 缓存预填摄像头帧
    wants_time_note: bool     # 轮指令上的流逝安静注记
    speaks_fallback: bool     # 无语音产出的轮仍说一句
    tts_voice: str            # per-mode Kokoro 音色(承载语言)
```

6 个标志位把「这个模式如何被门控与提示」完全编码。frozen 便于并发安全传递。

### `MODES`(30–50)——三个模式

```python
MODES = {m.name: m for m in [
    Mode("conversation", uses_smart_turn=True,  allows_delegation=True,
         wants_camera=True,  wants_time_note=True,  speaks_fallback=True,
         tts_voice="af_heart"),
    Mode("translate",    uses_smart_turn=False, allows_delegation=False,
         wants_camera=False, wants_time_note=False, speaks_fallback=True,
         tts_voice="af_heart"),
    Mode("listen",       uses_smart_turn=False, allows_delegation=False,
         wants_camera=False, wants_time_note=True,  speaks_fallback=False,
         tts_voice="af_heart"),
]}
```

逐模式注释(设计理由):

| 标志 | conversation | translate | listen | 理由 |
|------|:---:|:---:|:---:|------|
| uses_smart_turn | ✓ | ✗ | ✗ | 传译只等短停顿不等「完整思想」;只听模式反正不回答 |
| allows_delegation | ✓ | ✗ | ✗ | 英文答案不应闯入传译/安静聆听 |
| wants_camera | ✓ | ✗ | ✗ | 传译/只听不应混入摄像头闲聊 |
| wants_time_note | ✓ | ✗ | ✓ | 传译会把时间注记也翻译进去;只听保留(喂退出问句「我忙了多久?」) |
| speaks_fallback | ✓ | ✓ | ✗ | 只听模式的静默轮就是**目的**,不给兜底台词 |

### 设计精髓
**数据驱动策略**:server.py 主循环里大量 `if mode.uses_smart_turn` / `if mode.allows_delegation` / `if mode.speaks_fallback` 等查询,而非 `if mode.name == "listen"`。新增模式只需定义新 `Mode` + 在 actions.py 加触发 clause,会话循环零改动。

### 改进点
1. **`tts_voice` 全是 `af_heart`**:per-mode 音色能力已预留但未用(翻译模式理论上可换对应语言音色,但 Kokoro 音色与语言耦合,当前只英语输出)。
2. **模式间无转换矩阵**:`switch_mode` 允许任意转换,actions.py 的 `allowed` 白名单硬编码;可把合法转换也放进 Mode。
3. **无「模式进入/退出钩子」**:当前进入逻辑在 `switch_mode`(清状态);若新模式需要特殊初始化需改 switch_mode。

---

## Part B:reasoner.py —— 后台研究委派

> 把语音模型的一个任务交给任意 OpenAI 兼容端点上的前沿文本模型,返回简短可朗读答案。**未设 `REASONER_API_KEY` 时禁用**——此时委派指令从系统提示中缺席,Parlor 保持完全端侧。默认 OpenRouter(`REASONER_WEB_SEARCH=1` 默认)给任意模型追加 `:online` 后缀启用 provider 侧网搜;其他端点忽略该标志。

### 配置(17–31)

```python
BASE_URL = os.environ.get("REASONER_BASE_URL", "https://openrouter.ai/api/v1").rstrip("/")
API_KEY  = os.environ.get("REASONER_API_KEY", "")
MODEL    = os.environ.get("REASONER_MODEL", "openai/gpt-5.6-luna")
TIMEOUT_S = float(os.environ.get("REASONER_TIMEOUT", "90"))

# OpenRouter 网搜通过 model id 本身请求,所以这里 baked in;其他端点会拒该后缀
if (os.environ.get("REASONER_WEB_SEARCH","1")=="1" and "openrouter.ai" in BASE_URL
        and not MODEL.endswith(":online")):
    MODEL += ":online"
```

### `SYSTEM_PROMPT`(35–41)
答案会被语音模型近逐字朗读,所以返回时**必须已是语音形态**:
```
You are the background research assistant behind a real-time voice AI. Answer the
task directly, in plain conversational text that will be read aloud: no markdown,
no lists, no headings, no URLs. Lead with the answer, keep it to 2-6 short sentences,
and include the concrete facts (names, numbers, places) the task asks for.
```

### `enabled()`(44–45)
```python
def enabled() -> bool: return bool(API_KEY)
```
**整个委派系统的总开关**。server.py 据此决定是否拼 `RESEARCH_NOTE`、是否允许 `spawn_delegation`。

### `ask(task)`(48–77)

```python
def ask(task) -> str:
    # api.openai.com 拒 reasoning 模型的 max_tokens,要 max_completion_tokens;
    # 其他 OpenAI 兼容(OpenRouter/llama-server/vLLM)收 max_tokens。仅字段名不同。
    field = ("max_completion_tokens" if "api.openai.com" in BASE_URL else "max_tokens")
    req = urllib.request.Request(
        BASE_URL + "/chat/completions",
        data=json.dumps({
            "model": MODEL,
            "messages": [{"role":"system","content":SYSTEM_PROMPT},
                         {"role":"user","content":task}],
            field: 4096,
        }).encode(),
        headers={"Content-Type":"application/json","Authorization":f"Bearer {API_KEY}"},
    )
    with urllib.request.urlopen(req, timeout=TIMEOUT_S) as resp:
        body = json.load(resp)
    answer = (body["choices"][0]["message"]["content"] or "").strip()
    if not answer: raise RuntimeError("reasoner returned an empty answer")
    return answer
```

要点:
1. **api.openai.com 特判**:`max_completion_tokens`(reasoning 模型)vs `max_tokens`(其他)。注释:reasoning 模型从同一预算花隐藏思考 token,答案仍短因系统提示要求几句——上限只是防长篇。
2. **`urllib.request` 阻塞**:调用者(server.py `run_delegation`)放 `REASONER_POOL` executor,不阻塞事件循环。
3. **空答案抛异常**:调用者转成口头道歉(`DELIVER_FAILED_PROMPT`)。
4. **异常上抛**:transport / HTTP 错误由 `run_delegation` 的 try 捕获 → `ok=False`。

### 设计精髓
- **职责单一**:只做「发请求拿答案」,策略(何时调用、答案怎么投递、失败怎么兜底)全在 server.py。
- **OpenAI 兼容抽象**:任意端点(OpenRouter / 直连 provider / 第二个本地 llama-server)即插即用。
- **网搜机制诚实**:注释明确 `:online` 是 OpenRouter 机制,其他端点研究仍跑但无实时网。

### 改进点
1. **无重试**:网络抖动直接失败(由 server 重试一次兜底)。可加指数退避。
2. **`max_tokens=4096` 固定**:答案被 server 截断到 1500 字符;此处上限偏宽。
3. **无流式**:整答案返回;语音模型转述时无法边到边读(但委派本就是后台,延迟不敏感)。
4. **API key 仅 env**:无 vault / 轮换机制(单机个人用例可接受)。

---

← [tts.md](tts.md) | 下一个:[frontend.md](frontend.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕