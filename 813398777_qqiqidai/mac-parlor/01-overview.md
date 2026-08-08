# 01 项目概述(Project Overview)

## 1.1 项目目标与核心价值

### 一句话定位

Parlor 是一个**完全端侧(on-device)、实时、多模态(语音 + 视觉)的 AI 对话助手**,在单台消费级笔记本(目标硬件 Apple Silicon MacBook M3 Pro)上,提供对标 OpenAI [GPT-Live](https://openai.com/index/introducing-gpt-live/) 的自然语音交互体验。

### 它解决的问题

项目的诞生有明确的故事背景(见 `README.md` 的 "Why?" 段,人类撰写部分):

1. 作者此前自托管了一个**免费语音 AI**(Bule AI),帮助数百名月活用户练习英语口语,投入数周打磨到接近 ChatGPT Advanced Voice 水平。
2. OpenAI 发布 GPT-Live 后,体验大幅领先,使作者的自托管应用「显得过时」。
3. 作者面临选择:把用户导向 GPT-Live,或把应用升级到与之对齐。他选择了后者,并施加一个硬约束:**必须 100% 在本地运行**。
4. 第一次尝试——**微调 Gemma 4 12B 让它表现为全双工(full-duplex)模型**(在模型上嫁接决策 tick + 语音头)——经多次试验后失败。
5. 当前结论:**在拥有端侧全双工模型之前,经典级联系统(cascade system)仍是更优解**。Parlor 就是这个级联系统的工程实现。

因此 Parlor 的核心价值可以归纳为:

| 价值维度 | 说明 |
|---------|------|
| **隐私与完全本地** | 麦克风音频、摄像头画面、对话内容**永不离开本机**(除非用户显式开启可选的后台研究)。这对隐私敏感场景(医疗、法律、个人)与离线场景至关重要。 |
| **零成本、零依赖** | 不调用任何付费云 API,无需账号、无需网络(默认配置下)。 |
| **可学习、可改造** | 全部代码开源(Apache 2.0),架构基于「测量驱动决策」,每个关键选择都有 benchmark 支撑,适合研究与二次开发。 |
| **低延迟自然对话** | 通过说话重叠式预填(overlap prefill)、流式分句 TTS、专用端点分类器等手段,把「说完话到听到第一个音」压到 ~0.7s(短问,e2b 模型,M3 Pro)。 |

### 目标用户

- **隐私敏感的个人用户**:希望拥有一个不把对话上传云端的语音助手。
- **英语口语学习者**:作者原始用例,语音 AI 帮助练习口语。
- **AI 工程师 / 研究者**:研究端侧多模态实时系统、级联架构、延迟优化的参考实现。
- **离线 / 弱网环境用户**:无可靠网络也能使用语音 AI。

## 1.2 技术栈完整清单

下表是 Parlor v2 的完整技术栈,均来自 `pyproject.toml`、`README.md` 与源码 import 分析。

### 语言与运行时

| 项 | 版本 / 说明 | 来源 |
|----|-----------|------|
| Python | **>=3.12, <3.13**(硬性约束) | `pyproject.toml` requires-python |
| 包管理 | **uv**(Astral) | `uv.lock` + README quick start |
| 构建后端 | hatchling | `pyproject.toml` build-backend |

### 后端(Python)

| 库 | 版本要求 | 用途 |
|----|---------|------|
| **fastapi** | >=0.135.3 | Web 框架,提供 HTTP 与 WebSocket 端点 |
| **uvicorn[standard]** | >=0.43.0 | ASGI 服务器,承载 FastAPI |
| **numpy** | >=2.4.4 | 音频 PCM 数组操作、特征提取 |
| **onnxruntime** | >=1.24.4 | 运行 smart-turn-v3 端点分类器 ONNX 模型 |
| **python-dotenv** | >=1.0.0 | 加载 `.env` 配置 |
| **pillow** | >=11.0.0 | 图像处理(依赖) |
| **soundfile** | >=0.13.1 | 音频读写 |
| **websockets** | >=16.0 | 测试套件用同步 WebSocket 客户端 |

### 平台条件依赖

| 库 | 平台条件 | 用途 |
|----|---------|------|
| **mlx-audio** | macOS(darwin) | Apple Silicon 上 Kokoro TTS 的 MLX GPU 后端 |
| **misaki[en]** | macOS | MLX 后端的音素化(phonemizer) |
| **num2words** | macOS | 数字转英文单词(MLX TTS) |
| **kokoro-onnx** | Linux | 非 Apple Silicon 的 CPU TTS 后端 |

### 外部原生依赖(非 Python)

| 依赖 | 说明 |
|------|------|
| **llama.cpp** | 核心推理引擎。以子进程方式启动 `llama-server`。macOS 用 `brew install llama.cpp`,需 **build b9503+(2026 年 6 月)**,`MODEL=12b` 需 **b9512+** |
| **HuggingFace Hub** | 通过 `huggingface_hub`(间接)自动下载 GGUF 模型、TTS 模型、端点分类器 |

### 模型资产

| 模型 | 作用 | 来源 |
|------|------|------|
| **Google Gemma 4**(E2B / E4B / 12B,官方 QAT q4_0 GGUF)| 语音 + 视觉多模态主模型 | `llama.py` MODELS 表 |
| **Gemma 4 mmproj** | 音频 + 视觉编码器投影 | 同上 |
| **smart-turn-v3.2-cpu.onnx**(Pipecat,8M 参数)| 端点(轮次结束)分类器 | `turn_detector.py` |
| **Kokoro-82M**(TTS,mlx-community / fastrtc)| 文本转语音 | `tts.py` |
| **Silero VAD**(`@ricky0123/vad-web`,浏览器侧)| 语音活动检测 | `app.js` |

### 前端

| 项 | 说明 |
|----|------|
| 原生 HTML / CSS / **Vanilla JS** | 无构建步骤、无框架。`web/index.html` + `web/static/{app.js,style.css}` |
| **WebAudio API** | 流式音频播放、波形可视化、analyser |
| **getUserMedia** | 采集麦克风 + 摄像头 |
| **Silero VAD(vad-web 0.0.29)** | 浏览器端 VAD,CDN 加载 |
| **onnxruntime-web 1.22.0** | 浏览器端运行 VAD 的 ONNX |

### 可选外部服务

| 服务 | 触发条件 | 用途 |
|------|---------|------|
| **OpenRouter**(或任意 OpenAI 兼容端点) | 设置 `REASONER_API_KEY` 时 | 后台研究:把「查天气 / 查新闻 / 深度研究」委派给前沿模型(默认 `openai/gpt-5.6-luna` + `:online` 网搜)|

## 1.3 架构风格及理由

Parlor v2 采用的是**经典级联架构(cascade / pipeline architecture)**,而非端到端全双工神经网络。其核心形态是:

```
浏览器(采集 + VAD + 播放)
      ↕  WebSocket(PCM 音频 + JPEG 帧)
FastAPI 服务端(每连接一个会话循环)
      ├─ smart-turn-v3(~20ms)        → 你说完了吗?
      ├─ Gemma 4 via llama.cpp        → 听 + 看,流式回复
      │     └─ action head(同模型,JSON) → 定时器 / 模式 / 研究
      ├─ Kokoro TTS                    → 逐句合成
      └─ 后台 reasoner(可选)          → 前沿模型联网研究
      ↕  WebSocket(转写 + 流式音频块)
浏览器(播放 + 显示转写)
```

### 为什么是级联而非全双工

这是 v2 最根本的架构选择,详见 [ADR-001](10-adrs.md)。要点:

1. **全双工模型微调失败**:作者的第一次尝试是微调 Gemma 4 12B 让它全双工化(嫁接决策 tick + 语音头),多次试验后放弃。
2. **等待端侧全双工模型**:作者认为,需要等到某家前沿 AI 公司发布端侧可跑、性能对齐 GPT-Live 的全双工模型。在那之前,级联系统更优。
3. **级联可被测量与优化**:每个阶段(VAD、端点、LLM、TTS)可独立 benchmark、独立替换、独立打补丁。Parlor v2 的每个架构决策都附带测量数据。

### 关键架构特征

| 特征 | 实现 | 理由 |
|------|------|------|
| **流式管道(streaming pipeline)** | LLM 解码 → 句子切分 → TTS,三者**重叠**进行(`pipeline.py` run_turn) | 把「等模型说完再合成」变成「边解码边合成边播」,逼近首音延迟下限 |
| **说话重叠式预填(overlap prefill)** | 用户还在说话时,就把摄像头帧 + ~3s 音频块推过 llama.cpp 前缀缓存 | 最终请求只需处理「尾部」,长问与短问的首音延迟几乎一样 |
| **专用小模型做端点判定** | smart-turn-v3(8M,~19ms)而非让 LLM 判定 | benchmark:小分类器 0.96 准确率,所有 Gemma 变体都接近随机且耗时 0.6–3.6s |
| **动作与语音解耦** | 定时器 / 模式 / 研究由**单独的 grammar-forced JSON 请求**决定,而非带内(in-band)控制标签 | benchmark:head 召回 1.0 vs 标签 0.955,且标签的漏判是「口头承诺但服务端不执行」——语音助手最糟的失败;控制标记永远不会泄漏进 TTS |
| **会话状态在内存、按连接** | 每个 WebSocket 连接一个独立的对话历史列表 | 无状态服务、无持久化、天然隔离;代价是重连丢失历史(有意为之) |
| **平台自适应后端** | TTS 在 Apple Silicon 用 MLX(GPU),其他平台用 ONNX(CPU)| 发挥硬件,保持跨平台 |

## 1.4 关键功能特性

### 面向用户的功能

1. **免按住对话(hands-free turn-taking)**:浏览器端 Silero VAD 以 ~200ms 静音切分;smart-turn-v3 再判定「你是否真的说完了一整句话」——思考中途的停顿会被静默 hold,等你继续说或安静下来才回答。
2. **一切皆流式**:回复以「先转写你说了什么」开头(立即上屏),随后**边生成边逐句朗读**。摄像头帧与语音在你说的时候就被推入缓存预填。
3. **打断(barge-in)**:你可以直接开口打断 AI,生成在服务端被真正中止。回声处理不依赖浏览器回声消除(或干脆戴耳机)。
4. **后台研究(可选)**:"现在罗马哪家披萨最好"会被交给前沿模型联网查,对话继续;答案到达后在下一个空闲时刻被朗读。
5. **定时器**:"给意面设个三分钟定时器"——服务端持有时钟,到点用任何模式朗读提醒,UI 有可取消的倒计时芯片。
6. **实时翻译模式**:"把我说的都翻译成英语"——Parlor 变成同声传译:每句话在短停顿后翻译成英语,不做对话式回复。
7. **只听模式(just-listen)**:"你先听会儿,我想自言自语"——变成静默速记:每句话上屏转写,不回话,直到你再次喊它。
8. **时间感**:模型被告知每轮前有多久的安静、研究花了多久、会话何时开始——于是「我离开多久了?」能得到真实答案。

### 非功能性需求(NFR)

| NFR | 目标 / 现状 | 测量方式 |
|-----|-----------|---------|
| **延迟** | 短问(~2s 语音)首音 ~0.7s;长问(~9s)~1.3–1.4s(e2b,M3 Pro,不含 ~200ms VAD 静音检测) | `benchmarks/bench.py` 端到端 |
| **资源占用** | 默认 E4B ~6 GB RAM;E2B ~4 GB;12B ~8 GB | README |
| **隐私** | 默认 100% 本地;仅 reasoner 可选外呼 | 架构 |
| **可用性 / 鲁棒性** | 一条坏 WAV 不能毒化历史导致后续全部 400;掉帧、空转写、回声、no-speech 均有兜底 | e2e 退化音频测试 |
| **可维护性** | 模块化、配置外置、测量驱动、~87 e2e 测试 | `tests/` |
| **可移植性** | macOS(Apple Silicon 优先)+ Linux(需支持的 GPU) | pyproject 平台条件依赖 |
| **可扩展性** | 新增「模式」=在 `modes.py` 加一条 Mode 数据 + 触发逻辑,无需改会话循环 | `modes.py` 设计注释 |
| **安全** | 仅监听 localhost(浏览器才把 http://localhost 当作安全上下文,否则 getUserMedia 不存在) | `server.py` main() |

## 1.5 版本脉络

| 版本 | 日期 | 要点 |
|------|------|------|
| **v1.0.0** | 2026-04-07 | 初版:LiteRT-LM + Gemma + Kokoro + 浏览器 WebSocket 客户端 |
| **v2.0.0** | 2026-08-03 | 近乎完全重建:推理迁至 llama.cpp;轮次判定迁至 smart-turn 分类器;动作迁出语音流为 JSON head;新增后台研究、定时器、翻译 / 只听模式;延迟大幅下降(短问 1.5s→0.7s) |

> 详见 `CHANGELOG.md`。v2 的每个决策都在 `benchmarks/` 留下可复现的测量。

---

下一章:[02 C4 架构模型](02-c4.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕