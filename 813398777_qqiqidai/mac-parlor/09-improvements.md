# 09 改进建议、风险点与未来规划

本章基于源码分析,给出当前架构的优缺点、可扩展性 / 安全 / 性能 / 重构建议,以及一份带优先级的技术债清单。

---

## 9.1 架构优点

| 优点 | 体现 |
|------|------|
| **测量驱动** | 每个关键决策(llama.cpp、smart-turn、解耦 head、transcript-first、服务端时钟)都有 `benchmarks/` 可复现测量,决策不靠直觉 |
| **延迟极致优化** | overlap prefill + 流式三段管道 + transcript 先行 + 帧预取,短问 ~0.7s 首音 |
| **鲁棒性 defense-in-depth** | no_speech / echo / 退化音频 / 坏 WAV / 粘性 interrupted / processing 看门狗 / phantom capture 多层兜底,单坏轮不拖垮会话 |
| **隐私 by design** | 默认 100% 本地,reasoner 可选 opt-in |
| **数据驱动可扩展** | modes.py Mode dataclass,新模式 = 数据 + 触发,会话循环零改 |
| **解耦清晰** | 8 模块依赖单向无环,核心推理可独立单测 |
| **跨平台** | TTS 平台自适应(MLX/ONNX),无锁定 |
| **诚实注释** | 几乎每段代码注释「为什么」,含踩坑史与测量值,可维护性极高 |

---

## 9.2 架构缺点 / 风险

| 缺点 / 风险 | 影响 | 严重度 |
|------------|------|--------|
| **`websocket_endpoint` 过长(~516 行 + 大量闭包)** | 可读性、可测试性受限;状态散落闭包变量 | 中 |
| **无持久化 / 无崩溃恢复** | 服务重启丢失全部会话 | 低(单用户场景有意) |
| **无 CI/CD / 无容器** | 质量靠手动 bench/pytest;无回归守护 | 中 |
| **无结构化日志 / metrics / 告警** | 生产化运维困难 | 中 |
| **`sentence_q` 无背压** | 大模型 + 慢 TTS 下句子堆积(靠 MAX_OUTPUT_TOKENS 间接限流) | 低 |
| **回声依赖「戴耳机」** | 扬声器外放场景仍有回声风险(裸输出绕过系统处理) | 中(已多层缓解) |
| **协议无版本号** | 服务端升级改协议静默破坏旧客户端 | 低(同源) |
| **`MAX_OUTPUT_TOKENS=256` 硬上限** | 长回复被截断(有 fallback) | 低 |
| **全局可变对象**(tts_backend/detector) | 单事件循环可控,多 worker 需重设计 | 低 |
| **CDN 强依赖**(VAD/ORT) | 离线首跑需联网拉运行时库 | 低 |
| **action head 每轮都跑** | 即便「hello」也付 ~2s GPU(隐藏在 TTS 下) | 低(已测量评估) |
| **无鉴权 / TLS** | 仅因 localhost 安全;远程访问需自加 | 低(设计如此) |

---

## 9.3 可扩展性建议

### 水平扩展(多用户)
当前**每连接一协程 + 全局共享模型对象**,单进程。多用户扩展:
- **uvicorn 多 worker**:但全局 `tts_backend`/`detector` 与 llama-server 单 slot(`-np 1`)不兼容多 worker 抢占。需每 worker 独立 llama-server 或改用多 slot(`-np N`)。
- **会话状态外移**:history 从内存挪到 Redis(违背「端侧」,适合多用户服务化版本,如作者原 Bule AI 场景)。
- **推理服务池**:llama-server 池 + 负载均衡,会话亲和(前缀缓存按 slot 局部性)。

### 功能扩展
- **新模式**:`modes.py` 加 Mode + `actions.py` 加 `_MODE_CLAUSES` 条目,会话循环零改。候选:摄像头旁白模式、传译对(双向)。
- **多语言 TTS**:per-mode `tts_voice` 已预留;需 Kokoro 多语言音色 + translate 模式目标语言参数化。
- **持久化(可选 opt-in)**:会话存档(JSON,脱敏 audio)。
- **工具调用扩展**:action head 的 JSON schema 已支持 timer/mode/research;新增动作类型(发邮件、控制 IoT)只需扩 schema + `apply_decision` 分支。

---

## 9.4 安全加固建议

| 建议 | 优先级 |
|------|--------|
| 远程访问:反向代理(Caddy/nginx)+ TLS + Basic Auth / Token | 仅远程时高 |
| API key 轮换机制(env → vault) | 低(单机) |
| 输入 sanitize:audio/image 大小上限防 DoS | 中 |
| 依赖扫描(`pip-audit` / Dependabot) | 中 |
| 模型完整性校验(HF 哈希) | 低 |

---

## 9.5 性能优化建议

| 建议 | 预期收益 | 复杂度 |
|------|---------|--------|
| **TTS 流式生成**(Kokoro 逐段产出即播) | 首音进一步降低 | 高(需后端支持) |
| **缓存 instruction 归一化**(`echoes_instruction` 每句重建 haystack) | 每轮省微秒级 | 低 |
| **增量 log-mel**(流式场景) | turn_detector 微优化 | 中(非瓶颈) |
| **KV cache 复用调优**(llama.cpp `-nk`/`--context-shift`) | 长会话 prefill 更省 | 中 |
| **action head 预门**(1-token yes/no) | 省无动作轮的 ~2s GPU | 低(已测量不值,模型升级后重评估) |
| **背压**(sentence_q 限长) | 防积压 | 低 |

---

## 9.6 重构建议

| 重构 | 动机 | 风险 |
|------|------|------|
| **`Session` 类化**:`websocket_endpoint` 闭包 → 类,闭包变量变属性、嵌套函数变方法 | 可读、可测、状态集中 | 中(改动大,需保持行为字节一致以维持缓存命中) |
| **prompt 外部化**:常量 → YAML/文件,支持热加载 | 改 prompt 不改代码 | 低 |
| **协议版本化**:`/ws?v=2` 或首帧协商 | 升级安全 | 低 |
| **结构化日志**(`logging` + JSON) | 可观测性 | 低 |
| **配置 dataclass**:env → 类型化配置对象 | 校验、文档 | 低 |

---

## 9.7 技术债清单(按优先级)

| 优先级 | 债项 | 来源 |
|--------|------|------|
| P1 | `websocket_endpoint` 函数过长 / 闭包状态 | server.py:332–848 |
| P1 | 无 CI 回归守护 | 仓库无 .github |
| P2 | 无结构化日志 / metrics | 全局 print |
| P2 | `sentence_q` 无背压 | pipeline.py |
| P2 | 协议无版本号 | /ws |
| P3 | prompt 硬编码 | server.py 常量 |
| P3 | 全局可变对象 | server.py:288 |
| P3 | CDN 运行时依赖 | index.html |
| P3 | `MAX_OUTPUT_TOKENS` 硬编码 | pipeline.py |
| P3 | 无模型完整性校验 | hf_hub_download |

---

## 9.8 未来规划(基于代码注释与 README 蛛丝马迹)

1. **等端侧全双工模型**:README 明确——等某前沿公司发布端侧可跑、对齐 GPT-Live 的全双工模型,届时级联可被替代。微调全双工的首次尝试已失败。
2. **模型升级**:Gemma 系列迭代时,`MODELS` 表加新条目 + 重跑 `benchmarks/`(尤其 turnbench/archbench)验证决策仍成立。
3. **更便宜的 action 判定**:若 per-turn GPU 成本在实践中显现,重评估 1-token 预门(archbench B_gated 已 vendored)。
4. **多语言 / 多模式**:per-mode 音色与新模式(传译对、摄像头旁白)。
5. **可观测性**:生产化(若作者想服务化 Bule AI 继任者)需 metrics + 持久化 + 多用户。

---

下一章:[10 架构决策记录(ADR)](10-adrs.md)

---

☕️ 制作不易，请我喝咖啡☕️关注我➕