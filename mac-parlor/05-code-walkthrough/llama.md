# 走读:llama.py(278 行)

> llama.cpp 后端:以子进程方式管理 `llama-server`,并封装其 OpenAI 兼容 chat API(阻塞 / 流式 / JSON / 缓存预热)。这是 Parlor 与推理引擎之间**唯一的**边界。

## 文件结构概览

```
llama.py
├─ 配置(22–39)        MODELS 表 / MODEL / PORT / URL / CTX / TEMPERATURE / _proc
├─ 模型路径解析(42–66) resolve_model_paths / model_label / host_port / _connect
├─ 构建版本门槛(84–131) MIN_BUILD / _hint / server_command / check_build
├─ 子进程生命周期(134–169) start / stop
├─ 请求体构建(172–192) _chat_body(cache_prompt + json_schema + stream_options)
├─ 阻塞请求(195–210)  chat_blocking
└─ 流式请求(213–278)  ChatStream(run / cancel)
```

---

## 一、配置(22–39)

### `MODELS`(24–31)
Google 官方 **QAT q4_0** 量化(quatization-aware training 训练进去的 q4_0 质量,比 K-quants 快)。三元组 `(huggingface_repo, gguf, mmproj)`:

```python
MODELS = {
    "e2b": ("google/gemma-4-E2B-it-qat-q4_0-gguf", "gemma-4-E2B_q4_0-it.gguf", "gemma-4-E2B-it-mmproj.gguf"),
    "e4b": ("google/gemma-4-E4B-it-qat-q4_0-gguf", ...),   # 默认,~6GB,答案更好
    "12b": ("google/gemma-4-12B-it-qat-q4_0-gguf", ...),   # 需 ~8GB
}
```

注释给出 M3 Pro 实测延迟参考:e2b ≈ 0.6–1.0s 首音,e4b ≈ 1.0–1.7s。

### 其他
```python
MODEL = os.environ.get("MODEL", "e4b").lower()
PORT = int(os.environ.get("LLAMA_PORT", "8081"))
URL  = os.environ.get("LLAMA_SERVER_URL", "")   # 设了就用外部 server,不自拉起
CTX  = int(os.environ.get("LLAMA_CTX", "16384"))
TEMPERATURE = float(os.environ.get("TEMPERATURE", "0.7"))
_proc = None
```

`load_dotenv()` 在顶部就调用——**下面的配置在 import 时读取,.env 必须先生效**。

---

## 二、模型路径解析(42–77)

### `resolve_model_paths()`(42–58)
优先级:`MODEL_PATH`+`MMPROJ_PATH`(完全覆盖)> 内置 MODEL。内置走 `huggingface_hub.hf_hub_download`,网络失败兜底 `local_files_only=True`(离线用本地缓存)。返回 `(model_path, mmproj_path)`。

### `model_label()`(61–66)
UI 标签:`MODEL_PATH` 设了用文件 stem;否则 `f"Gemma 4 {MODEL.upper()}"`(如 "Gemma 4 E4B")。

### `host_port()`(69–73)
`URL` 设了从 URL 解析(`split("//")[-1].partition(":")`);否则 `("127.0.0.1", PORT)`。

### `_connect(timeout)`(76–77)
返回 `http.client.HTTPConnection(*host_port(), timeout=timeout)`。**用裸 http.client 而非 requests/httpx**——最小化流式 SSE 解析开销与依赖。

---

## 三、构建版本门槛(84–131)——重要的鲁棒性设计

Gemma 4 需要较新的 llama.cpp,b9503 以下缺音频支持或加载 mmproj 崩溃(upstream #24084),12b mmproj 需 b9512。老安装会在模型加载时给一个**令人困惑的崩溃**,所以**前端强制门槛**。

```python
MIN_BUILD = 9503
MODEL_MIN_BUILD = {"12b": 9512}
```

### `_hint(verb)`(90–94)
macOS → `brew {verb} llama.cpp`;其他 → 上游安装指南。

### `server_command()`(97–108)
```python
binary = shutil.which("llama-server")
if binary: return [binary]
unified = shutil.which("llama")
if unified: return [unified, "serve"]   # llama.cpp 一键安装器的统一二进制,serve 子命令即同一 server
raise RuntimeError(f"llama-server not found — install llama.cpp: ...")
```
**发现逻辑**:兼容 Homebrew / release tarball 装的 `llama-server` 与 llama.app 一键安装器的统一 `llama` 二进制。

### `check_build(cmd, floor)`(111–131)
对根二进制跑 `--version`(统一 `llama serve --version` 可能不退出),正则 `version:\s*(\d+)`。版本行如 `version: 10150 (dee2a846b)`。**报告不可解析的自建树(`version: 0 (unknown)`)放行**——门槛是为陈旧安装,不是自建。低于 floor → `RuntimeError` 带安装方式感知的升级提示(unified 二进制 → 重跑安装器;否则 brew upgrade)。

---

## 四、子进程生命周期(134–169)

### `start()`(134–164)
```python
if URL: print(f"Using external llama-server at {URL}"); return   # 外部 server 跳过
cmd = server_command()
check_build(cmd, MODEL_MIN_BUILD.get(MODEL, MIN_BUILD))
model, mmproj = resolve_model_paths()
_proc = subprocess.Popen(
    cmd + ["-m", model, "--mmproj", mmproj, "-ngl", "99",   # 全部层上 GPU
           "--port", str(PORT), "-c", str(CTX), "-np", "1"], # 单 slot
    stdout=DEVNULL, stderr=DEVNULL,                          # 调 llama 时取消静音
)
deadline = time.time() + 180
while time.time() < deadline:
    if _proc.poll() is not None: raise RuntimeError(f"exited with code {_proc.returncode}")
    try:
        conn = _connect(timeout=2); conn.request("GET", "/health")
        if conn.getresponse().status == 200: print("ready."); return
    except OSError: pass
    time.sleep(1)
raise RuntimeError("did not become ready in 180s")
```
- `-ngl 99`:全部层卸载到 GPU(Apple Silicon Metal)。
- `-np 1`:单 slot——与前缀缓存配合,会话独占一个 slot。
- 轮询 `/health` 最多 180s。子进程早退 → 立即报错。

### `stop()`(167–169)
`_proc.terminate()`。`server.py` 的 lifespan 在退出时调用;崩溃的服务器会孤儿化子进程——`tests/conftest.py` 的 `shutdown()` 额外扫 llama 端口兜底。

---

## 五、`_chat_body`(172–192)——请求体构建

```python
body = {
    "messages": messages, "max_tokens": max_tokens,
    "temperature": TEMPERATURE if temperature is None else temperature,
    "stream": stream,
    "cache_prompt": True,                              # ★ 前缀缓存(每次重发历史的基础)
    "chat_template_kwargs": {"enable_thinking": False}, # 关 Gemma 4 thinking
}
if json_schema:
    body["response_format"] = {"type":"json_schema","json_schema":{"schema":json_schema}}  # llama-server 编译为 grammar
if stream:
    body["stream_options"] = {"include_usage": True}   # ★ 末 chunk 带 prompt_tokens
return body
```

两个承重细节:
1. **`cache_prompt=True`**:让「每次重发全部历史」几乎免费(命中前缀缓存),也是 overlap prefill 的基础。
2. **`stream_options.include_usage`**:流式末 chunk 带 `usage.prompt_tokens`——**真实上下文大小**,驱动历史轮转(估算会漂移)。

`json_schema` 由 llama-server 编译为 grammar——**输出结构上保证可解析**(action head 用)。

---

## 六、`chat_blocking`(195–210)

```python
def chat_blocking(messages, max_tokens, temperature=None, json_schema=None) -> str:
    conn = _connect(timeout=300)
    conn.request("POST", "/v1/chat/completions", json.dumps(_chat_body(..., stream=False, ...)), {"Content-Type":"application/json"})
    resp = conn.getresponse()
    data = json.loads(resp.read()); conn.close()
    if "error" in data: raise RuntimeError(f"llama-server: {data['error']}")
    return data["choices"][0]["message"].get("content") or ""
```

非流式,用于 action head(`actions._decide`)与 `prime_cache`。timeout 300s(首跑大上下文 prefill 慢)。`error` 显式抛出(空内容是合法的「丢弃」结果,见 cache priming)。

---

## 七、`ChatStream`(213–278)——★ 线程安全的流式驱动器

### 类 docstring
"Streaming chat request, driven from an executor thread. cancel() is thread-safe and actually aborts generation server-side (the connection close is observed by llama-server)."

### 字段
```python
self.body = _chat_body(messages, max_tokens, stream=True)
self.conn = None
self.cancelled = False
self.prompt_tokens = None    # 来自 usage chunk 的真实计数
```

### `run(on_delta)`(224–268)
```python
self.conn = _connect(timeout=300)   # 发请求前发布 conn,使中途 cancel 能拆 socket
self.conn.request("POST", "/v1/chat/completions", json.dumps(self.body), {...})
resp = self.conn.getresponse()
if resp.status != 200:
    body = resp.read()[:300]; self.conn.close()
    raise RuntimeError(f"HTTP {resp.status}: {body!r}")   # 静默空轮会进历史毒化后续
try:
    while True:
        line = resp.readline()
        if not line: break
        line = line.strip()
        if not line.startswith(b"data: "): continue
        payload = line[6:]
        if payload == b"[DONE]": break
        chunk = json.loads(payload)
        usage = chunk.get("usage")
        if usage and usage.get("prompt_tokens"): self.prompt_tokens = usage["prompt_tokens"]
        choices = chunk.get("choices") or []
        text = choices[0].get("delta", {}).get("content") if choices else None
        if text: on_delta(text)
except Exception as e:
    if not self.cancelled: print(f"LLM stream ended early: {type(e).__name__}: {e}")
finally:
    try: self.conn.close()
    except OSError: pass
```

要点:
- **SSE 手解**:`data: ` 前缀 + `[DONE]` 终止。
- **HTTP 非 200 抛错**:注释——静默空轮会被存进历史毒化后续。
- **异常分类**:cancel 期间的异常(http.client 清理竞相 cancel 可抛 AttributeError `_close_conn`)被吞;真正死掉的 server 在下次请求浮现。

### `cancel()`(270–278)
```python
self.cancelled = True
try:
    if self.conn and self.conn.sock: self.conn.sock.shutdown(socket.SHUT_RDWR)
    if self.conn: self.conn.close()
except OSError: pass
```
**关键**:`shutdown(SHUT_RDWR)` + `close()` 让 llama-server **观察到连接关闭而停止生成**——这是 barge-in 在服务端真正生效的基础(不是仅客户端停止播放)。线程安全:`conn` 在 `run` 发请求前就发布,中途 cancel 也能拆 socket。

---

## 设计模式与改进点

| 模式 | 体现 |
|------|------|
| **Fail-fast 配置校验** | 构建门槛、二进制发现都在 `start()` 前端 |
| **线程安全取消** | `cancel()` socket shutdown |
| **流式 SSE 手解** | 最小依赖、最低开销 |
| **结构化输出** | json_schema → grammar 编译 |

改进点:
1. **`_proc` 全局可变**:`start()`/`stop()` 非线程安全;单进程模型下可接受,但测试并发起多 server 需小心(conftest 用专用端口隔离)。
2. **180s 就绪超时硬编码**:极慢机器或大模型可能不够;可配置。
3. **无重连**:llama-server 崩溃后无自动重启,下次请求即暴露。生产化可加健康检查 + 重启。
4. **`http.client` 手解无重试**:网络抖动(本地 IPC 少见)直接失败。本地场景可接受。

---

← [pipeline.md](pipeline.md) | 下一个:[actions.md](actions.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)