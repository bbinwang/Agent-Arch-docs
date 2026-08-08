# 05-16 — eval_framework/_templating.py 代码讲解

> **文件**: `packages/eval-framework/src/eval_framework/_templating.py`
> **行数**: ~51 行
> **职责**: 轻量模板引擎
> **预估字数**: ~4,000 字

---

## Template 类

```python
import re

PATTERN = re.compile(r"\{\{([^\}]+)\}\}")

class TemplateValidationError(Exception):
    """Raised when the arguments to render a template fail to validate"""

class Template:
    def __init__(self, content: str):
        self.content = content
        self._to_render = PATTERN.findall(content)

    def _validate(self, args: dict[str, str]) -> bool:
        return all(el in args for el in self._to_render) and all(
            isinstance(args[k], str) for k in args
        )

    def render(self, args: dict[str, str]) -> str:
        if self._validate(args):
            content = self.content
            for word in self._to_render:
                content = content.replace("{{" + word + "}}", args[word])
            return content
        else:
            if (ls := list(set(self._to_render) - set(list(args.keys())))) != []:
                raise TemplateValidationError(f"Missing the following arguments: {', '.join(ls)}")
            else:
                raise TemplateValidationError("You should provide a dictionary with only string values.")
```

**设计要点**:
- 使用正则 `{{([^\}]+)}}` 匹配模板变量
- 严格验证：所有变量必须提供，所有值必须是字符串
- 明确的错误提示：区分缺少参数和类型错误

---

# ❤️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../../res/promotion.jpg)