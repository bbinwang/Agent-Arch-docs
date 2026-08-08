# 11 开发者上手指南

从零到改代码、跑测试、做 benchmark 的完整流程。

---

## 11.1 环境准备

```bash
# 1. 克隆
git clone https://github.com/fikrikarim/parlor.git
cd parlor

# 2. 装 uv(Astral 的 Python 包/项目管理器)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. 装 llama.cpp(macOS;其他平台见上游 install guide)
brew install llama.cpp
#   需 build b9503+(2026-06);MODEL=12b 需 b9512+
#   llama.py check_build 会在启动时强制校验并给升级提示

# 4. 装依赖(自动建 .venv,锁 uv.lock)
uv sync

# 5. (可选)配置 .env
cp .env.example .env
#   按需改 MODEL / REASONER_API_KEY 等;详见 docs/configuration.md
```

**Python 版本**:必须 3.12(pyproject `>=3.12,<3.13`)。`uv` 会自动管理。

**首次运行会下载模型**(~5.7GB E4B QAT + mmproj + TTS),需联网 + HuggingFace 访问。

---

## 11.2 运行

```bash
uv run parlor
#  → uvicorn 监听 http://localhost:8000(仅 localhost)
```

打开浏览器 → 授权摄像头/麦克风 → 开始说话。

### 常用环境变量

```bash
MODEL=e2b uv run parlor           # 更快/更省内存(~4GB),适合开发迭代
MODEL=e4b uv run parlor           # 默认,答案更好(~6GB)
PORT=8001 uv run parlor           # 换端口
LLAMA_SERVER_URL=http://127.0.0.1:8081/v1 uv run parlor  # 用已运行的 llama-server
KOKORO_ONNX=1 uv run parlor       # Apple Silicon 强制 CPU TTS(调试)
```

---

## 11.3 项目布局心智模型

改代码前必读:
- **改对话行为 / prompt** → `server.py`(prompt 常量 52–246 + 主循环 598–840)。
- **改流式 / 延迟 / 解析** → `pipeline.py`。
- **改推理后端** → `llama.py`。
- **改动作判定(定时器/模式/研究)** → `actions.py`。
- **加新模式** → `modes.py`(+ `actions.py` `_MODE_CLAUSES`)。
- **改端点判定** → `turn_detector.py`。
- **改 TTS** → `tts.py`。
- **改前端(VAD/采集/播放)** → `web/static/app.js`。

详见 [§04 模块结构](04-modules.md) 与 [§05 代码走读](05-code-walkthrough/)。

---

## 11.4 调试

### 看日志
启动后 stdout 行缓冲流式输出。关键诊断:
```
Starting llama-server with gemma-4-E4B_q4_0-it.gguf (ctx=16384)...
llama-server ready.
TTS: mlx-audio (Apple GPU, sample_rate=24000)
Turn detector loaded (smart-turn-v3.2).
Primed cache (0.31s)
Turn detector: p(complete)=0.92 (19ms)
LLM (0.74s, prefill 0.31s) heard: 'hello there' → 'Hi! How can I help?'
Action decision: ActionDecision(...)
```

### 调 llama.cpp 本身
`llama.py:147` 把子进程 stdout/stderr 设为 `DEVNULL`。调试 llama 时取消静音:
```python
_proc = subprocess.Popen(cmd + [...], stdout=None, stderr=None)
```

### 前端调试
浏览器 DevTools Console:`app.js` 有大量 `console.log/warn`(VAD 初始化、barge-in、phantom reset、processing watchdog)。Network → WS 帧可见全部 JSON 消息。

### 关掉确定性(测试用)
生产 `TEMPERATURE=0.7`;套件用 `0` 保证确定性。开发调试可临时设 0 复现。

---

## 11.5 测试

```bash
uv run pytest           # ~1 分钟 + 模型加载;~87 e2e 用例
```
- 套件**自拉起真实服务端**(真实 llama.cpp/TTS/turn detector)在专用端口(8821/8822/8823)。
- 用合成语音(含退化音频:截断词尾/噪声/他人声)驱动,复现真麦失败模式。
- `test_stream_parser.py` 是**纯单元测试**(无服务端依赖)——改 `StreamParser` 后秒级验证。
- `PARLOR_TEST_URL=ws://localhost:8000/ws uv run pytest` 指向已运行服务(跳过自拉起,log/process 测试 skip)。
- 详见 [§13 测试策略](13-testing.md)。

### 单测 StreamParser(无需模型)
```bash
uv run pytest tests/test_stream_parser.py -q
```

---

## 11.6 Benchmark(测量驱动开发)

**改任何架构决策前,先跑 before;改完跑 after,对比。**

### 端到端延迟
```bash
# 终端 1
uv run parlor
# 终端 2
uv run python benchmarks/bench.py --label before --out benchmarks/results/before.json
# ...改代码,重启服务...
uv run python benchmarks/bench.py --label after --out benchmarks/results/after.json
uv run python benchmarks/compare.py benchmarks/results/before.json benchmarks/results/after.json
```

### 各 benchmark 用途

| 脚本 | 测什么 | 何时跑 |
|------|-------|-------|
| `bench.py` | 端到端延迟(说完→首音、整轮) | 改延迟相关 |
| `turnbench.py` | smart-turn vs LLM 端点准确率 | 换端点模型/逻辑 |
| `archbench.py` | 带内标签 vs 解耦 head 召回 | 改 action 架构 |
| `tagbench.py` | 生产温度(0.7)动作召回 | 验证生产召回 |
| `camerabench.py` | 每轮附帧 vs 摄像头作工具调用 | 改摄像头策略 |
| `timerprobe.py` | 服务端持时钟的必要性 | 论证 timer 设计 |
| `benchmark_tts.py` | TTS 性能 | 换 TTS 后端 |

> 退役架构(`legacy_tags.py` 等)被 **vendored**,基线可对未来模型重跑。

---

## 11.7 常见问题

| 问题 | 解决 |
|------|------|
| `llama.cpp build XXXX is too old` | 按 `check_build` 提示升级(brew upgrade / 重跑安装器) |
| `llama-server not found` | `brew install llama.cpp`(macOS)或见上游 install guide |
| 首次启动很慢 | 正在下载模型(~5.7GB);之后走缓存 |
| 浏览器拿不到麦/摄像头 | 必须用 `http://localhost:8000`(非 0.0.0.0 / IP);授权 |
| 回声严重 | 戴耳机;或调 barge-in 阈值 |
| 测试端口被占 | `lsof -ti :8821` 杀残留;conftest 会 fail 提示 |
| `uv sync` 失败 | 确认 Python 3.12;`uv python install 3.12` |

---

## 11.8 贡献流程建议(仓库未定义,推荐实践)

1. 改前跑 `bench.py --label before`。
2. 小步改 + 注释「为什么」(本项目注释文化极强)。
3. 跑 `uv run pytest` 全绿。
4. 跑 `bench.py --label after` + `compare.py`,附结果到 PR。
5. 若改协议,同步前端 `app.js`。
6. CHANGELOG 记录(测量值)。

---

下一章:[12 关键算法与复杂度分析](12-algorithms.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕