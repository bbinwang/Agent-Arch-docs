# markdown-it 架构文档

> **项目**: [markdown-it/markdown-it](https://github.com/markdown-it/markdown-it)  
> **版本**: 14.3.0  
> **许可**: MIT  
> **作者**: Vitaly Puzrin, Alex Kocharin  

---

## 1. 项目概述

markdown-it 是一个用 JavaScript 编写的 Markdown 解析器，遵循 CommonMark 规范，并内置 GFM 扩展（表格、删除线）。核心理念是 **"正确解析 Markdown，快速且易于扩展"**。

### 1.1 核心特性

- **CommonMark 兼容**: 严格遵循 [CommonMark spec](http://spec.commonmark.org/)，通过全部 conformance 测试
- **GFM 扩展内置**: 表格 (Table) 和删除线 (Strikethrough) 默认启用
- **插件架构**: 所有语法规则都是独立函数，可通过 `Ruler` 系统自由增删替换
- **安全优先**: 默认禁用 HTML，过滤危险 URL 协议（`javascript:`, `vbscript:`, `file:`, `data:`），输出安全无需额外 sanitizer
- **高性能**: CommonMark 模式 ~1568 ops/sec，默认模式 ~743 ops/sec（基准测试），速度不因灵活性打折
- **零运行时依赖安全风险**: 仅 6 个直接依赖，均为成熟的窄功能库

### 1.2 技术栈

| 层面 | 技术 |
|------|------|
| 语言 | JavaScript (ES Modules, `.mjs`) |
| 运行时 | Node.js / 浏览器 (UMD build) |
| 构建 | 自定义 `support/build-dist.mjs` (Vite + terser) |
| 测试 | Node.js 内置 `node:test` runner |
| 规范测试 | CommonMark spec (600+ test cases) |
| Lint | ESLint 9 + neostandard |
| 文档生成 | ndoc |
| 基准测试 | tinybench |

### 1.3 运行时依赖

```
argparse ^2.0.1     — CLI 参数解析
entities ^4.5.0     — HTML 实体编解码
linkify-it ^5.0.2   — URL 自动识别
mdurl ^2.0.0        — URL 解析与编码
punycode.js ^2.3.1  — 域名 Punycode 编码
uc.micro ^2.1.0     — Unicode 字符属性类
```

### 1.4 项目规模

- **核心源码**: ~40 个 `.mjs` 模块，`lib/` 目录
- **Block 规则**: 11 个（table, code, fence, blockquote, hr, list, reference, html_block, heading, lheading, paragraph）
- **Inline 规则**: 12 个 tokenize + 4 个 post-process（balance_pairs, strikethrough, emphasis, fragments_join）
- **Core 规则**: 7 个（normalize, block, inline, linkify, replacements, smartquotes, text_join）
- **测试**: CommonMark spec 全量 + markdown-it 自有 fixtures + pathological + build 验证

---

## 2. 架构设计

### 2.1 三层解析管线 (Three-Stage Pipeline)

markdown-it 的核心设计是一个 **三层嵌套规则链** 架构，而非传统 AST 解析器：

```
┌─────────────────────────────────────────────────────┐
│                   MarkdownIt                        │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │              Core Pipeline                   │   │
│  │  normalize → block → inline → linkify →      │   │
│  │  replacements → smartquotes → text_join      │   │
│  └──────────┬──────────────────┬───────────────┘   │
│             │                  │                    │
│     ┌───────▼───────┐  ┌──────▼────────┐           │
│     │ ParserBlock   │  │ ParserInline  │           │
│     │ (11 rules)    │  │ (12 rules)    │           │
│     │               │  │               │           │
│     │ table         │  │ text          │           │
│     │ code          │  │ linkify       │           │
│     │ fence         │  │ newline       │           │
│     │ blockquote    │  │ escape        │           │
│     │ hr            │  │ backticks     │           │
│     │ list          │  │ strikethrough │           │
│     │ reference     │  │ emphasis      │           │
│     │ html_block    │  │ link          │           │
│     │ heading       │  │ image         │           │
│     │ lheading      │  │ autolink      │           │
│     │ paragraph     │  │ html_inline   │           │
│     │               │  │ entity        │           │
│     │   + ruler2:   │  │               │           │
│     │   balance     │  │   + ruler2:   │           │
│     │   strike      │  │   balance     │           │
│     │   emphasis    │  │   strike      │           │
│     │   fragments   │  │   emphasis    │           │
│     └───────────────┘  │   fragments   │           │
│                        └───────────────┘           │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │              Renderer                        │   │
│  │  Token Stream → HTML String                  │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

**数据流**：

```
输入字符串 → Core.normalize → Core.block (Block解析) 
  → Core.inline (Inline解析) → Core.linkify 
  → Core.replacements → Core.smartquotes → Core.text_join
  → Token Stream → Renderer → HTML 输出
```

### 2.2 为什么用 Token Stream 而不是 AST？

markdown-it 刻意选择 **Token 流**（一个扁平数组）而非树状 AST：

- **简化设计**：遵循 KISS 原则，不需要遍历树结构
- **渲染高效**：Renderer 只需顺序遍历数组
- **易于扩展**：插件直接操作 token 数组，无需理解树结构
- **灵活渲染**：可将 token 流转换为任意格式（HTML / JSON / XML / 自定义 AST）

Token 类型结构：

```
Token {
  type:     'paragraph_open'    // token 类型名
  tag:      'p'                 // HTML 标签名
  attrs:    [['class', 'foo']]  // 属性列表 [name, value] 对
  nesting:  1                   // 1=open, -1=close, 0=self-closing
  level:     0                  // 嵌套层级
  children:  null               // inline 容器的子 token
  content:   ''                 // 文本内容
  markup:    ''                 // 源标记符号 (* _ # 等)
  info:      ''                 // 附加信息 (fence 语言、autolink 标记等)
  map:       [0, 1]             // 源码行映射 [start, end]
  block:     true               // 是否块级 token
  hidden:    false              // 是否隐藏 (tight list 用)
  meta:      null               // 插件自定义数据
}
```

### 2.3 Ruler 系统：规则管理核心

`Ruler` 是 markdown-it 可扩展性的基石。每个解析器（Core / Block / Inline）都持有一个 `Ruler` 实例：

```javascript
class Ruler {
  __rules__   // [{ name, enabled, fn, alt[] }]
  __cache__   // 编译后的规则链缓存
}
```

**核心 API**：

- `push(name, fn)` — 末尾追加规则
- `before(name, newName, fn)` — 在指定规则前插入
- `after(name, newName, fn)` — 在指定规则后插入
- `at(name, fn)` — 替换规则
- `enable(list)` / `disable(list)` — 启用/禁用规则
- `enableOnly(list)` — 白名单模式，仅启用指定规则
- `getRules(chain)` — 获取已编译的活跃规则数组

**Alt 链机制**：Block 规则通过 `alt` 属性声明它能被哪些规则 "中断"。例如 `table` 规则声明 `alt: ['paragraph', 'reference']`，意味着段落内可以出现表格。

### 2.4 预设系统 (Presets)

三种预设控制规则组合和选项：

| 预设 | 用途 | maxNesting | HTML | 特殊规则 |
|------|------|------------|------|----------|
| `default` | 类 GFM，所有规则启用 | 100 | false | 全部 block + inline 规则 |
| `commonmark` | 严格 CommonMark | 20 | true | 去掉 table, strikethrough, linkify, smartquotes |
| `zero` | 全部禁用，手动配置 | 20 | false | 仅保留 paragraph + text |

---

## 3. 核心解析流程

### 3.1 主入口 `MarkdownIt.render(src, env)`

```javascript
render(src, env) {
  // 1. 解析 → Token Stream
  const tokens = this.parse(src, env)
  // 2. 渲染 → HTML
  return this.renderer.render(tokens, this.options, env)
}
```

### 3.2 Core Pipeline 执行顺序

```
1. normalize     — 统一换行符为 \n，NULL → \uFFFD
2. block         — 调用 ParserBlock，生成块级 token
3. inline        — 对每个 inline 容器调用 ParserInline
4. linkify       — 自动识别 URL 文本并转为链接
5. replacements  — 语言无关替换 (© → ©, ± → ± 等)
6. smartquotes   — 直引号 → 排版引号 ("" → "")
7. text_join     — 合并 text_special token（转义序列等）
```

### 3.3 Block 解析器工作方式

`ParserBlock.tokenize()` 按行扫描输入：

1. 跳过空行
2. 检查嵌套深度限制 (`maxNesting`)
3. 依次尝试所有规则，首个返回 `true` 的规则胜出
4. 规则负责更新 `state.line`（前进到下一行）和 `state.tokens`
5. 如果没有规则匹配 → 抛出异常（`paragraph` 是兜底规则）

**验证模式 (silent mode)**：规则支持 `silent=true` 参数，仅做前瞻检查不修改 token 流。用于嵌套场景下判断是否可以中断当前块。

### 3.4 Inline 解析器工作方式

`ParserInline` 有两个阶段：

**阶段 1 — tokenize**：逐字符扫描，依次尝试 12 条规则

**阶段 2 — ruler2 post-process**：处理配对标记
1. `balance_pairs` — 平衡嵌套的强调标记
2. `strikethrough.postProcess` — 匹配 `~~` 对
3. `emphasis.postProcess` — 匹配 `*`/`_` 对（区分 em/strong）
4. `fragments_join` — 合并被拆分的文本碎片

**缓存优化**：`skipToken` 方法使用位置缓存 (`state.cache[pos]`)，避免对同一位置重复解析。

### 3.5 Emphasis 匹配算法

markdown-it 的 emphasis 处理是 CommonMark 最复杂的部分之一：

1. **Tokenize 阶段**：遇到 `*` 或 `_` 时，调用 `scanDelims` 判断 `can_open` / `can_close`，将每个标记字符作为独立的 text token 加入 `state.delimiters` 数组
2. **Post-process 阶段**：从后向前扫描 delimiters 数组，找到配对的开闭标记
3. **Strong 检测**：如果两个相邻的同类型标记可以配对（如 `**text**`），合并为 `<strong>`
4. **Token 改写**：将 text token 改写为 `em_open`/`em_close` 或 `strong_open`/`strong_close`

---

## 4. Token 与渲染

### 4.1 Token 类

`Token` 是一个轻量级数据类，提供属性操作方法：

- `attrIndex(name)` / `attrGet(name)` — 属性查找
- `attrSet(name, value)` — 设置属性（覆盖）
- `attrPush([name, value])` — 追加属性
- `attrJoin(name, value)` — 空格拼接属性值（常用于 class）

### 4.2 Renderer 渲染

Renderer 遍历 token 流，对每种 token 类型调用对应的渲染规则：

```javascript
render(tokens, options, env) {
  for (let i = 0; i < tokens.length; i++) {
    const type = tokens[i].type
    if (type === 'inline') {
      result += this.renderInline(tokens[i].children, options, env)
    } else if (typeof rules[type] !== 'undefined') {
      result += rules[type](tokens, i, options, env, this)
    } else {
      result += this.renderToken(tokens, i, options)  // 默认渲染
    }
  }
}
```

**默认渲染规则**：`code_inline`, `code_block`, `fence`, `image`, `hardbreak`, `softbreak`, `text`, `html_block`, `html_inline`

**自定义渲染**：直接覆盖 `md.renderer.rules[type]`，无需修改解析器：

```javascript
md.renderer.rules.link_open = function(tokens, idx, options, env, self) {
  tokens[idx].attrSet('target', '_blank')
  return self.renderToken(tokens, idx, options)
}
```

---

## 5. 安全设计

markdown-it 的安全策略是多层的：

1. **HTML 默认禁用**：`html: false`，源码中的 HTML 标签会被转义而非执行
2. **URL 协议白名单**：`validateLink()` 阻止 `javascript:`, `vbscript:`, `file:`, `data:`（仅允许 `data:image/...` 安全子集）
3. **链接归一化**：`normalizeLink()` 使用 mdurl 编码 + punycode 处理域名
4. **插件隔离**：插件操作 tokenized 内容，不接触原始 HTML
5. **DOM Clobbering 防护**：文档提醒插件不要生成依赖用户输入的 `id` / `name` 属性

---

## 6. 插件系统

### 6.1 插件加载

```javascript
md.use(plugin1)
  .use(plugin2, opts)
```

`use()` 调用 `plugin(md, ...opts)`，插件可自由操作：
- `md.block.ruler` / `md.inline.ruler` / `md.core.ruler` — 增删规则
- `md.renderer.rules` — 覆盖渲染
- `md.options` — 修改选项

### 6.2 官方插件生态

| 插件 | 功能 |
|------|------|
| markdown-it-sub | 下标 |
| markdown-it-sup | 上标 |
| markdown-it-footnote | 脚注 |
| markdown-it-deflist | 定义列表 |
| markdown-it-abbr | 缩写 |
| markdown-it-emoji | emoji |
| markdown-it-container | 自定义容器 |
| markdown-it-ins | 插入线 |
| markdown-it-mark | 高亮标记 |
| markdown-it-for-inline | 遍历 inline token 辅助工具 |

---

## 7. CLI 工具

```
markdown-it [options] file
```

| 参数 | 功能 |
|------|------|
| `--no-html` | 禁用 HTML |
| `-l, --linkify` | 启用自动链接 |
| `-t, --typographer` | 启用排版替换 |
| `--trace` | 显示错误堆栈 |
| `-o, --output` | 输出文件（默认 stdout） |

支持 stdin 输入（`-`）。

---

## 8. 测试体系

| 测试类别 | 内容 |
|----------|------|
| `test:cmspec` | CommonMark spec 全量 conformance 测试（600+ cases） |
| `test:markdown-it` | 自有 fixtures：linkify, normalize, smartquotes, strikethrough, tables, typographer, xss, proto, fatal |
| `test:build` | dist 构建产物完整性验证 |
| `pathological` | 病态输入测试（ReDoS 防护） |
| `ruler.test` | Ruler 规则管理单元测试 |
| `token.test` | Token 属性操作单元测试 |

CI 使用 GitHub Actions (`.github/workflows/ci.yml`)。

---

## 9. 架构亮点与启示

### 9.1 设计优势

- **极端可扩展性**：所有语法都是 Ruler 中的命名规则，可以精确控制每一条
- **Token 流的简洁性**：相比 AST，操作扁平数组更直观，插件编写门槛极低
- **验证模式优化**：silent mode 让块级嵌套判断无需副作用
- **缓存机制**：inline 解析的位置缓存避免重复计算
- **性能可控**：预设系统让用户精确选择功能/性能平衡点

### 9.2 潜在局限

- **无增量解析**：每次渲染都是全量重新解析，无 AST diff 能力
- **无源码 sourcemap**：token 有 `map` 但不精确到字符级
- **单线程设计**：不适合超大文档流式处理（虽然有 `maxNesting` 防护）
- **Token 流非标准**：输出不是 AST，转换为其他格式需要自定义逻辑

### 9.3 对开发者的启示

markdown-it 的架构是 **"规则链 + 状态机 + Token 流"** 模式的经典实现，这种设计模式适用于：
- 任何需要可扩展的文本解析场景
- 需要 "管道式" 数据处理的设计
- 需要精确控制处理步骤顺序的系统

---

## 10. 关键文件索引

```
lib/
├── index.mjs              — MarkdownIt 主类（入口、选项、enable/disable/use）
├── parser_core.mjs        — Core pipeline（7 条 core 规则编排）
├── parser_block.mjs       — Block 解析器（11 条规则 + tokenize 循环）
├── parser_inline.mjs      — Inline 解析器（12 条规则 + ruler2 后处理）
├── ruler.mjs              — 规则管理器（push/before/after/at/enable/disable）
├── renderer.mjs           — HTML 渲染器（默认规则 + renderToken）
├── token.mjs              — Token 数据类
├── presets/
│   ├── default.mjs        — 默认预设（类 GFM）
│   ├── commonmark.mjs     — CommonMark 严格模式
│   └── zero.mjs           — 空预设
├── rules_core/            — Core 规则
│   ├── normalize.mjs      — 换行/NULL 归一化
│   ├── block.mjs          — 委托 ParserBlock
│   ├── inline.mjs         — 委托 ParserInline
│   ├── linkify.mjs        — URL 自动链接
│   ├── replacements.mjs   — 排版替换
│   ├── smartquotes.mjs    — 智能引号
│   ├── text_join.mjs      — 文本合并
│   └── state_core.mjs     — Core 状态对象
├── rules_block/           — 11 个块级规则
│   ├── table.mjs          — GFM 表格
│   ├── code.mjs           — 缩进代码块
│   ├── fence.mjs          — 围栏代码块
│   ├── blockquote.mjs     — 引用块
│   ├── hr.mjs             — 水平分割线
│   ├── list.mjs           — 列表（有序/无序）
│   ├── reference.mjs      — 参考链接定义
│   ├── html_block.mjs     — HTML 块
│   ├── heading.mjs        — ATX 标题
│   ├── lheading.mjs       — Setext 标题
│   ├── paragraph.mjs      — 段落（兜底规则）
│   └── state_block.mjs    — Block 状态对象
├── rules_inline/          — 12 个行内规则
│   ├── text.mjs           — 纯文本
│   ├── linkify.mjs        — 行内 URL 自动链接
│   ├── newline.mjs        — 硬换行/软换行
│   ├── escape.mjs         — 转义字符
│   ├── backticks.mjs      — 行内代码
│   ├── strikethrough.mjs  — 删除线 (~~)
│   ├── emphasis.mjs       — 强调 (* _)
│   ├── link.mjs           — 链接 (inline + reference)
│   ├── image.mjs          — 图片
│   ├── autolink.mjs       — 尖括号自动链接
│   ├── html_inline.mjs    — 行内 HTML
│   ├── entity.mjs         — HTML 实体
│   ├── balance_pairs.mjs  — 嵌套配对平衡
│   ├── fragments_join.mjs — 碎片合并
│   └── state_inline.mjs   — Inline 状态对象
├── helpers/               — 链接解析辅助
│   ├── parse_link_destination.mjs
│   ├── parse_link_label.mjs
│   └── parse_link_title.mjs
└── common/
    ├── utils.mjs          — 工具函数集
    ├── html_blocks.mjs    — HTML 块级标签名表
    └── html_re.mjs        — HTML 正则表达式
```

---

*本文档基于 markdown-it v14.3.0 源码分析生成。*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕