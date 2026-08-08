# 05-08 Collector 文档处理

> **所属章节**: [05-核心代码讲解](./05-核心代码讲解.md)  
> **核心文件**: `collector/index.js`、`collector/processSingleFile/index.js`、`collector/processSingleFile/convert/*`、`collector/processLink/index.js`、`collector/processRawText/index.js`、`collector/utils/*`

---

## 1. 架构概述

Collector 是 AnythingLLM 的 **文档采集与处理子服务**，作为独立 Node.js 进程运行，通过内部 HTTP（默认端口 8888）与 API Server 通信。

### 1.1 设计理由

| 决策 | 理由 |
|------|------|
| 独立进程 | 文档处理（OCR、Puppeteer、大文件解析）是 CPU 密集操作，避免阻塞 API Server 事件循环 |
| HTTP 通信 | 语言无关，未来可用其他语言重写 Collector |
| 内部端口 | 不暴露到外部网络，安全隔离 |

### 1.2 核心端点

| 端点 | 方法 | 功能 |
|------|------|------|
| `/process` | POST | 单文件处理（需签名验证） |
| `/parse` | POST | 文件解析（不保存） |
| `/link` | POST | 网页链接抓取 |
| `/raw-text` | POST | 原始文本处理 |
| `/get-file` | GET | 获取文件内容 |
| `/audio` | POST | 音频转写 |
| `/extensions/*` | POST | 扩展端点 |

---

## 2. 单文件处理管线

### 2.1 入口函数

```javascript
// collector/processSingleFile/index.js
async function processSingleFile(targetFilename, options = {}, metadata = {}) {
  // 1. 路径规范化与校验
  const fullFilePath = normalizePath(
    options.absolutePath || path.resolve(WATCH_DIRECTORY, targetFilename)
  );

  // 2. 安全检查
  if (!options.absolutePath && !isWithin(path.resolve(WATCH_DIRECTORY), fullFilePath))
    return { success: false, reason: "Filename is a not a valid path to process." };

  if (RESERVED_FILES.includes(targetFilename))
    return { success: false, reason: "Filename is a reserved filename and cannot be processed." };

  if (!fs.existsSync(fullFilePath))
    return { success: false, reason: "File does not exist in upload directory." };

  // 3. 扩展名解析
  const fileExtension = path.extname(fullFilePath).toLowerCase();
  if (fullFilePath.includes(".") && !fileExtension)
    return { success: false, reason: "No file extension found." };

  // 4. 处理器选择
  let processFileAs = fileExtension;
  if (!SUPPORTED_FILETYPE_CONVERTERS.hasOwnProperty(fileExtension)) {
    if (isTextType(fullFilePath)) {
      processFileAs = ".txt";
    } else {
      if (!options.absolutePath) trashFile(fullFilePath);
      return { success: false, reason: `File extension ${fileExtension} not supported.` };
    }
  }

  // 5. 动态加载处理器
  const FileTypeProcessor = require(SUPPORTED_FILETYPE_CONVERTERS[processFileAs]);
  return await FileTypeProcessor({ fullFilePath, filename: targetFilename, options, metadata });
}
```

### 2.2 处理器映射表

```javascript
// collector/utils/constants.js
const SUPPORTED_FILETYPE_CONVERTERS = {
  ".pdf": "./convert/asPDF/index.js",
  ".docx": "./convert/asDocx.js",
  ".doc": "./convert/asOfficeMime.js",
  ".xlsx": "./convert/asXlsx.js",
  ".xls": "./convert/asXlsx.js",
  ".pptx": "./convert/asOfficeMime.js",
  ".html": "./convert/asTxt.js",
  ".htm": "./convert/asTxt.js",
  ".txt": "./convert/asTxt.js",
  ".md": "./convert/asTxt.js",
  ".csv": "./convert/asTxt.js",
  ".epub": "./convert/asEPub.js",
  ".mbox": "./convert/asMbox.js",
  ".wav": "./convert/asAudio.js",
  ".mp3": "./convert/asAudio.js",
  ".mp4": "./convert/asAudio.js",
  ".png": "./convert/asImage.js",
  ".jpg": "./convert/asImage.js",
  // ...
};
```

### 2.3 处理器返回格式

所有处理器返回统一格式：

```javascript
{
  success: boolean,     // 是否成功
  reason: string,       // 失败原因（若失败）
  documents: [          // 文档块列表
    {
      pageContent: "...",  // 文本内容
    }
  ],
  metadata: {           // 文档元数据
    title: "...",
    author: "...",
    // ...
  }
}
```

---

## 3. 各格式处理器详解

### 3.1 PDF 处理器（asPDF）

PDF 处理是 **最复杂的处理器**，支持多种解析策略：

1. **原生解析**：使用 `@mintplex-labs/mdpdf` 或 `pdf-parse` 提取文本。
2. **OCR 回退**：若原生解析结果为空（扫描件），使用 OCR 引擎识别文字。
3. **图片提取**：将每页转为图片（用于多模态模型）。

```javascript
// collector/processSingleFile/convert/asPDF/index.js（简化）
module.exports = async function asPDF({ fullFilePath, options, metadata }) {
  // 1. 尝试原生解析
  const nativeResult = await parsePdfNative(fullFilePath);
  if (nativeResult.text.length > 100) {
    return { success: true, documents: [{ pageContent: nativeResult.text }] };
  }

  // 2. 原生解析失败，尝试 OCR
  const ocrLoader = new OCRLoader();
  const ocrResult = await ocrLoader.parse(fullFilePath);
  if (ocrResult.success) {
    return { success: true, documents: ocrResult.documents };
  }

  // 3. 都失败
  return { success: false, reason: "PDF parsing failed (native + OCR)" };
};
```

### 3.2 DOCX 处理器

```javascript
// collector/processSingleFile/convert/asDocx.js
module.exports = async function asDocx({ fullFilePath }) {
  const mammoth = require("mammoth");
  const result = await mammoth.extractRawText({ path: fullFilePath });
  return {
    success: true,
    documents: [{ pageContent: result.value }],
  };
};
```

### 3.3 XLSX 处理器

```javascript
// collector/processSingleFile/convert/asXlsx.js
module.exports = async function asXlsx({ fullFilePath }) {
  const XLSX = require("node-xlsx");
  const sheets = XLSX.parse(fullFilePath);
  const documents = sheets.map(sheet => ({
    pageContent: XLSX.utils.sheet_to_csv(sheet.data),
  }));
  return { success: true, documents };
};
```

### 3.4 音频处理器

音频文件（MP3、MP4、WAV）需要 **转写为文本**：

1. **格式转换**：`convertAudioToWav/index.js` 将各种音频格式转为 WAV。
2. **语音转写**：使用 Whisper（本地或 API）将 WAV 转为文字。

```javascript
// collector/convertAudioToWav/index.js
module.exports = async function convertAudioToWav({ fullFilePath }) {
  const ffmpeg = require("fluent-ffmpeg");
  const wavPath = fullFilePath.replace(path.extname(fullFilePath), ".wav");
  await new Promise((resolve, reject) => {
    ffmpeg(fullFilePath)
      .toFormat("wav")
      .on("end", resolve)
      .on("error", reject)
      .save(wavPath);
  });
  return wavPath;
};
```

### 3.5 图片处理器

图片文件使用 **OCR 或 多模态模型** 提取内容：

- **OCR 模式**：使用 Tesseract.js 识别文字。
- **多模态模式**：将图片发送给支持视觉的 LLM（如 GPT-4V）进行描述。

---

## 4. 网页链接处理

### 4.1 流程

```javascript
// collector/processLink/index.js
async function processLink(url, options = {}) {
  // 1. 使用 Puppeteer 抓取页面
  const browser = await puppeteer.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto(url, { waitUntil: "networkidle2" });

  // 2. 提取正文内容
  const content = await page.evaluate(() => {
    // 移除脚本、样式、导航等
    const scripts = document.querySelectorAll("script, style, nav, footer");
    scripts.forEach(el => el.remove());
    return document.body.innerText;
  });

  await browser.close();

  // 3. 文本分块
  const chunks = textSplitter.split(content);
  return { success: true, documents: chunks };
}
```

### 4.2 安全限制

- **超时控制**：Puppeteer 页面加载超时（默认 30 秒）。
- **资源限制**：禁用图片/CSS 加载（可选），减少带宽消耗。
- **域名白名单**：可配置允许抓取的域名列表。

---

## 5. 安全机制

### 5.1 完整性校验

```javascript
// collector/middleware/verifyIntegrity.js
function verifyPayloadIntegrity(req, res, next) {
  const signature = req.headers["x-signature"];
  const payload = JSON.stringify(req.body);
  const expected = comKey.sign(payload);

  if (signature !== expected) {
    return res.status(403).json({ error: "Invalid signature" });
  }
  next();
}
```

**功能**：使用 HMAC 签名验证请求来自合法的 API Server，防止未授权访问。

### 5.2 路径遍历防护

```javascript
// collector/utils/files/index.js
function isWithin(parent, child) {
  const relative = path.relative(parent, child);
  return relative && !relative.startsWith("..") && !path.isAbsolute(relative);
}
```

**功能**：确保处理的文件路径在允许的目录内，防止 `../../../etc/passwd` 攻击。

---

## 6. 潜在问题与改进建议

| 问题 | 影响 | 建议 |
|------|------|------|
| 无处理队列 | 大文件阻塞后续请求 | 添加任务队列（Bull/BullMQ） |
| OCR 质量不稳定 | 扫描件识别率低 | 支持多种 OCR 引擎回退 |
| Puppeteer 内存泄漏 | 长时间运行内存增长 | 定期重启浏览器实例 |
| 无进度反馈 | 大文件处理用户等待焦虑 | 通过 WebSocket 推送进度 |
| 音频转写慢 | 长音频处理耗时 | 支持流式转写 |

---

**返回 → [05-核心代码讲解.md](./05-核心代码讲解.md)** | **上一子文件 ← [05-07-MCP集成.md](./05-07-MCP集成.md)** | **下一子文件 → [05-09-前端React架构.md](./05-09-前端React架构.md)**

---

☕️ 制作不易，请我喝咖啡☕️关注我➕