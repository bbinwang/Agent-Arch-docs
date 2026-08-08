# 07-02 — 工具调用 Schema

> **本章内容**: 工具调用的 JSON Schema 定义
> **预估字数**: ~6,000 字

---

## 1. Action Schema

```json
{
  "type": "object",
  "properties": {
    "action": {
      "anyOf": [
        {"$ref": "#/$defs/ToolCallAction"},
        {"$ref": "#/$defs/GoDeeperAction"},
        {"$ref": "#/$defs/StopAction"},
        {"$ref": "#/$defs/AskHumanAction"}
      ],
      "description": "Action specification for the next step"
    },
    "reason": {
      "type": "string",
      "description": "Reason for taking this specific action"
    }
  },
  "required": ["action", "reason"]
}
```

---

## 2. ToolCallAction Schema

```json
{
  "type": "object",
  "properties": {
    "tool_name": {
      "type": "string",
      "enum": ["read", "grep", "glob", "check_api_key", "parse_file"],
      "description": "Chosen tool"
    },
    "tool_input": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "parameter_name": {"type": "string"},
          "parameter_value": {}
        },
        "required": ["parameter_name", "parameter_value"]
      },
      "description": "Input to call the tool with"
    }
  },
  "required": ["tool_name", "tool_input"]
}
```

---

## 3. 工具参数定义

### 3.1 read

```json
{
  "file_path": {"type": "string", "description": "Path to the text file"}
}
```

### 3.2 grep

```json
{
  "file_path": {"type": "string", "description": "Path to the file"},
  "pattern": {"type": "string", "description": "Regular expression pattern"}
}
```

### 3.3 glob

```json
{
  "directory": {"type": "string", "description": "Directory to search in"},
  "pattern": {"type": "string", "description": "File name pattern"}
}
```

### 3.4 check_api_key

```json
{}
# 无参数
```

### 3.5 parse_file

```json
{
  "file_path": {"type": "string", "description": "Path to the unstructured file"}
}
```

---

## 4. 请求示例

### 4.1 read 工具调用

```json
{
  "action": {
    "tool_name": "read",
    "tool_input": [
      {"parameter_name": "file_path", "parameter_value": "data/testfile.txt"}
    ]
  },
  "reason": "I need to check the content of this file"
}
```

### 4.2 grep 工具调用

```json
{
  "action": {
    "tool_name": "grep",
    "tool_input": [
      {"parameter_name": "file_path", "parameter_value": "data/report.txt"},
      {"parameter_name": "pattern", "parameter_value": "error|warning"}
    ]
  },
  "reason": "Searching for error messages in the report"
}
```

### 4.3 GoDeeper

```json
{
  "action": {
    "directory": "data/benchmark"
  },
  "reason": "The target file is likely in this subdirectory"
}
```

### 4.4 Stop

```json
{
  "action": {
    "final_result": "The answer is 42."
  },
  "reason": "I have found the answer to the user's question"
}
```

### 4.5 AskHuman

```json
{
  "action": {
    "question": "Which file should I summarize?"
  },
  "reason": "There are multiple relevant files and I need clarification"
}
```

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)