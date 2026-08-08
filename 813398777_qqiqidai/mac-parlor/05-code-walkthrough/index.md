# 05 核心代码走读(Code Walkthrough)

本目录对 Parlor 的核心文件进行**逐函数 / 逐类 / 逐模块**深度走读,按重要程度排列。每个文件讲清:功能、参数、返回值、核心逻辑、设计模式、潜在问题与改进点。

| 子文档 | 模块 | 行数 | 重要度 |
|--------|------|------|--------|
| [server.md](server.md) | `server.py` | 860 | ★★★★★ 系统编排器,所有策略集中于此 |
| [pipeline.md](pipeline.md) | `pipeline.py` | 476 | ★★★★★ 流式管道 + StreamParser |
| [llama.md](llama.md) | `llama.py` | 278 | ★★★★ 推理后端客户端 |
| [actions.md](actions.md) | `actions.py` | 155 | ★★★★ 动作决策器 |
| [turn_detector.md](turn_detector.md) | `turn_detector.py` | 230 | ★★★ 端点分类器 + log-mel |
| [tts.md](tts.md) | `tts.py` | 69 | ★★★ 平台自适应 TTS |
| [modes-reasoner.md](modes-reasoner.md) | `modes.py` + `reasoner.py` | 50 + 77 | ★★★ 模式数据 + 后台研究 |
| [frontend.md](frontend.md) | `web/static/app.js` + `index.html` | 820 + 60 | ★★★★ 浏览器逻辑(VAD/barge-in/播放) |

> 阅读顺序建议:`server` → `pipeline` → `llama` → `actions` → 其余。

---

☕️ 制作不易，请我喝咖啡☕️关注我➕