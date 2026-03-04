# 课时 4：多模态能力

[← 上一课：工具使用](./03-tool-use.md) | [返回目录](./README.md) | [下一课：流式传输与高级功能 →](./05-streaming-and-advanced.md)

---

## 概述

Claude 具备强大的多模态能力，可以理解和分析图片、PDF 文档等非文本内容。本课时讲解如何发送图片（Base64 和 URL）、处理 PDF 文档、常见图像分析用例和多模态最佳实践。

---

## 图片输入：Vision 能力

支持的格式：JPEG（`image/jpeg`）、PNG（`image/png`）、GIF（`image/gif`）、WebP（`image/webp`）。

### 方式一：Base64 编码

将本地图片编码后嵌入请求，适合确保图片可用性的场景：

```python
import anthropic
import base64
from pathlib import Path

client = anthropic.Anthropic()

image_data = base64.standard_b64encode(Path("screenshot.png").read_bytes()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/png",
                    "data": image_data
                }
            },
            {"type": "text", "text": "请描述这张图片的内容。"}
        ]
    }]
)
```

### 方式二：URL 引用

直接引用在线图片，更简洁但需确保 URL 可访问：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "url", "url": "https://example.com/chart.png"}},
            {"type": "text", "text": "请分析这张图表中的数据趋势。"}
        ]
    }]
)
```

### 发送多张图片

一个消息中可发送多张图片，Claude 会综合分析：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": before_data}},
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": after_data}},
            {"type": "text", "text": "请对比这两张 UI 设计图的差异。"}
        ]
    }]
)
```

---

## PDF 文档处理

Claude 可以直接处理 PDF 文档，提取其中的文本和视觉内容。

```python
import base64
from pathlib import Path

pdf_data = base64.standard_b64encode(Path("report.pdf").read_bytes()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=4096,
    messages=[{
        "role": "user",
        "content": [
            {
                "type": "document",
                "source": {"type": "base64", "media_type": "application/pdf", "data": pdf_data}
            },
            {
                "type": "text",
                "text": "请阅读这份报告，总结：\n1. 主题和目的\n2. 关键数据\n3. 主要结论"
            }
        ]
    }]
)
```

也支持通过 URL 发送 PDF：

```python
{"type": "document", "source": {"type": "url", "url": "https://example.com/report.pdf"}}
```

**注意事项**：PDF 页数限制取决于模型和页面复杂度；大型 PDF 建议分页发送；扫描件也能处理但效果取决于图片质量。

---

## 图像分析用例

### 用例 1：图表数据提取

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": chart_data}},
            {"type": "text", "text": "以 JSON 格式提取图表数据，包括图表类型、标题、数据点和趋势。"}
        ]
    }]
)
```

### 用例 2：UI 截图审查

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    system="你是资深 UI/UX 设计师。分析视觉层次、色彩搭配、交互设计和无障碍访问。",
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": ui_data}},
            {"type": "text", "text": "请审查这个移动端登录页面的设计。"}
        ]
    }]
)
```

### 用例 3：OCR 文字识别

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/jpeg", "data": receipt_data}},
            {"type": "text", "text": "识别发票中的文字：发票号码、日期、商品明细和合计金额。"}
        ]
    }]
)
```

---

## 通用图片处理函数

建议封装通用函数简化开发：

```python
import base64
from pathlib import Path

def analyze_image(image_source: str, prompt: str, model: str = "claude-sonnet-4-6") -> str:
    """分析图片。image_source 可以是文件路径或 URL。"""
    if image_source.startswith(("http://", "https://")):
        image_content = {"type": "image", "source": {"type": "url", "url": image_source}}
    else:
        path = Path(image_source)
        data = base64.standard_b64encode(path.read_bytes()).decode("utf-8")
        suffix_to_mime = {".jpg": "image/jpeg", ".jpeg": "image/jpeg",
                         ".png": "image/png", ".gif": "image/gif", ".webp": "image/webp"}
        media_type = suffix_to_mime.get(path.suffix.lower(), "image/png")
        image_content = {
            "type": "image",
            "source": {"type": "base64", "media_type": media_type, "data": data}
        }

    message = client.messages.create(
        model=model, max_tokens=2048,
        messages=[{"role": "user", "content": [image_content, {"type": "text", "text": prompt}]}]
    )
    return message.content[0].text

# 使用
result = analyze_image("./chart.png", "描述图表中的数据趋势。")
result = analyze_image("https://example.com/photo.jpg", "这张照片是在哪里拍的？")
```

---

## 多模态最佳实践

### 图片质量与尺寸

- 较大的图片消耗更多 token，建议适当压缩
- 如只需分析局部区域，先裁剪再发送

### 提示词编写

```python
# 不好
"看看这张图"

# 好
"这是一张电商首页截图。请分析：\n1. 产品展示布局\n2. 导航栏设计\n3. CTA 按钮是否突出"
```

### 与工具使用结合

多模态能力可以与工具使用结合，例如分析图片后自动将结果存入数据库。

### 常见限制

| 限制 | 说明 |
|------|------|
| 图片大小 | 建议单张不超过 20MB |
| 文字识别 | 清晰印刷体效果好，手写体准确率较低 |
| 空间推理 | 复杂空间关系判断可能不够准确 |
| 实时性 | 无法判断图片拍摄时间或验证真实性 |

---

## 关键要点

- Claude 支持 JPEG、PNG、GIF、WebP 图片输入，可通过 Base64 或 URL 发送
- PDF 文档通过 `document` 类型直接发送，Claude 能理解文本和视觉内容
- 常见用例：图表数据提取、UI 审查、OCR 识别、代码截图分析
- 具体有针对性的提示词能显著提升多模态分析质量
- 建议封装通用图片处理函数简化开发流程
- 多模态能力可与工具使用结合，构建更强大的自动化工作流

---

[← 上一课：工具使用](./03-tool-use.md) | [返回目录](./README.md) | [下一课：流式传输与高级功能 →](./05-streaming-and-advanced.md)
