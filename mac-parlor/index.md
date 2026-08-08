# Parlor 架构文档

> Parlor **v2.0.0** —— 完全端侧、实时多模态(语音 + 视觉)AI 的深度技术文档。
>
> 本文档严格基于 `/Users/wangbin/projects/parlor` 下的实际源码逐行分析生成,所有结论均可追溯到具体文件、函数与行号,杜绝臆造。全部图表使用标准 Mermaid 语法,可直接渲染。

---

## 这是什么项目

Parlor 是一个**完全运行在单台机器本地**(目标硬件:Apple Silicon MacBook,如 M3 Pro)的实时语音 + 视觉 AI 助手,功能对标 OpenAI 的 [GPT-Live](https://openai.com/index/introducing-gpt-live/)。浏览器采集麦克风音频与摄像头画面,经 WebSocket 送至本地 FastAPI 服务;服务端通过 **llama.cpp(Gemma 4 QAT 量化模型)** 推理、**smart-turn-v3 端点分类器**判定轮次结束、**Kokoro TTS** 按句合成语音,再以**流式**方式回传转写文本与音频分块。所有推理在本地完成,不依赖云端;唯一的可选外呼是「后台研究(reasoner)」——即便如此,默认也是关闭的,关闭时 100% 端侧。

## 核心一句话

> 用「经典级联系统」(VAD → 端点分类器 → LLM → TTS)+ llama.cpp 前缀缓存做出来的**说话重叠式预填(prefill)**,在单机本地逼近 GPT-Live 级别的对话延迟与自然度。

## 文档导航

| 文档 | 内容 |
|------|------|
| [01 项目概述](01-overview.md) | 目标、价值、完整技术栈、架构风格、功能与非功能需求 |
| [02 C4 架构模型](02-c4.md) | Context / Container / Component / Code 四层视图(Mermaid + 详解) |
| [03 系统流程与时序图](03-flows.md) | 9 个核心业务流程与时序图(每个含 Mermaid + 文字解读) |
| [04 模块结构与依赖](04-modules.md) | 完整目录树、各模块职责、模块依赖图 |
| [05 核心代码走读](05-code-walkthrough/) | 逐文件、逐函数、逐类的深度走读(8 个子文档) |
| [06 数据模型与会话状态](06-data-model.md) | 会话历史结构、消息 Part schema、内存状态、数据类(诚实说明:无传统数据库) |
| [07 API 与接口设计](07-api.md) | WebSocket 双向协议全量消息表 + REST 接口 |
| [08 部署、运维与基础设施](08-deployment.md) | 启动流程、模型下载、版本门槛、安全上下文、资源占用 |
| [09 改进建议、风险与未来规划](09-improvements.md) | 优缺点、扩展/安全/性能建议、技术债清单 |
| [10 架构决策记录(ADR)](10-adrs.md) | 每个关键决策的历史、理由与测量依据 |
| [11 开发者上手指南](11-developer-guide.md) | 本地运行、调试、测试、常见问题 |
| [12 关键算法与复杂度分析](12-algorithms.md) | log-mel 特征、句子切分、barge-in 投票、历史轮转等 |
| [13 测试策略](13-testing.md) | e2e pytest 套件、mock reasoner、基准体系 |

## 源码规模速览

| 维度 | 数值 |
|------|------|
| 后端 Python 源码 | 8 个模块,约 **2196 行**(`src/parlor/*.py`) |
| 前端 | `index.html`(60 行)+ `app.js`(820 行)+ `style.css` |
| 基准测试 | 10 个脚本,约 **2523 行**(`benchmarks/*.py`) |
| 测试 | `tests/` 下 14 个文件,约 **87 个 e2e 用例** |
| 依赖管理 | `uv` + `pyproject.toml` + `uv.lock` |
| 许可证 | Apache 2.0 |

## 阅读建议

- **想快速理解系统**:先读 [01 项目概述](01-overview.md) → [02 C4 架构模型](02-c4.md) → [03 系统流程](03-flows.md)。
- **想改代码**:读 [05 核心代码走读](05-code-walkthrough/) 对应模块 → [11 开发者上手指南](11-developer-guide.md)。
- **想知道为什么这么设计**:读 [10 架构决策记录(ADR)](10-adrs.md),每条都有 benchmark 测量支撑。
- **想评估质量与风险**:读 [09 改进建议](09-improvements.md)。

---

*本文档由架构文档生成技能(github-architecture-doc)基于真实源码产出。*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)