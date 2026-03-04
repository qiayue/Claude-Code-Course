# 课时 2：提示工程

[← 上一课：API 基础](./01-api-fundamentals.md) | [返回目录](./README.md) | [下一课：工具使用 →](./03-tool-use.md)

---

## 概述

提示工程（Prompt Engineering）是与 Claude 高效交互的核心技能。本课时涵盖系统提示词与用户提示词的区别、多轮对话管理、XML 标签结构化、思维链推理、少样本学习和提示缓存。

---

## 系统提示词与用户提示词

### 系统提示词（System Prompt）

定义 Claude 在整个对话中的角色、风格和行为约束：

```python
import anthropic

client = anthropic.Anthropic()

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="你是一名资深的数据库工程师，专注于 PostgreSQL。回答时：\n"
           "1. 提供 SQL 代码示例\n2. 解释性能影响\n3. 提醒安全风险",
    messages=[{"role": "user", "content": "如何创建一个用户表？"}]
)
```

### 用户提示词（User Prompt）

好的用户提示词应该**明确具体**、**提供上下文**、**指定格式**：

```python
# 模糊的提示词
bad_prompt = "写代码"

# 明确的提示词
good_prompt = """请用 Python 编写一个函数，实现以下功能：
- 输入：一个包含学生姓名和成绩的字典列表
- 输出：按成绩从高到低排序的列表
- 要求：包含类型注解和文档字符串"""
```

### 结构化系统提示词最佳实践

```python
system_prompt = """你是一个代码审查助手。

<role>资深软件工程师，擅长 Python 和代码质量分析。</role>

<guidelines>
- 关注可读性、可维护性和性能
- 指出潜在的 bug 和安全漏洞
- 提供改进建议和修改后的代码
</guidelines>

<output_format>
1. 问题描述  2. 严重程度（高/中/低）  3. 改进建议  4. 修改后的代码
</output_format>"""
```

---

## 多轮对话

将完整对话历史作为 `messages` 数组传递，Claude 基于完整上下文生成回复：

```python
conversation = []

def chat(user_message: str) -> str:
    conversation.append({"role": "user", "content": user_message})

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system="你是一个友好的编程导师。",
        messages=conversation
    )

    assistant_message = response.content[0].text
    conversation.append({"role": "assistant", "content": assistant_message})
    return assistant_message

print(chat("什么是列表推导式？"))
print(chat("能给我一个更复杂的例子吗？"))
```

**消息格式规则**：消息必须以 `user` 开始，`user` 和 `assistant` 角色交替出现。可通过末尾添加 `assistant` 消息进行预填充：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "列出 Python 的三大优势，以 JSON 格式返回。"},
        {"role": "assistant", "content": "{"}  # 预填充，引导输出 JSON
    ]
)
print("{" + message.content[0].text)
```

---

## 使用 XML 标签组织信息

Claude 天然理解 XML 标签语义，能够准确区分不同部分的内容：

```python
prompt = """请分析以下代码并提供改进建议。

<code>
def calculate_average(numbers):
    total = 0
    for n in numbers:
        total += n
    return total / len(numbers)
</code>

<requirements>
- 处理空列表的情况
- 添加类型注解
- 支持浮点数精度控制
</requirements>

请按以下格式回答：
<analysis>代码分析</analysis>
<improved_code>改进后的代码</improved_code>
<explanation>改进说明</explanation>"""
```

常用标签模式：`<document>` 包裹文档、`<instructions>` 包裹指令、`<context>` 提供背景、`<output_format>` 指定输出格式。

---

## 思维链推理

思维链（Chain of Thought, CoT）要求 Claude 先展示推理过程再给出答案，对复杂逻辑和数学任务特别有效：

```python
prompt = """请一步一步思考，然后回答：

一个水池有两个进水管和一个出水管。
- 进水管 A 单独注满需要 6 小时
- 进水管 B 单独注满需要 8 小时
- 出水管 C 单独排空需要 12 小时

三个管同时打开，需要多少小时注满水池？

在 <thinking> 中展示推理过程，在 <answer> 中给出最终答案。"""
```

对于代码调试，可以要求逐步分析：先理解预期功能，再逐行跟踪执行，找出 bug 并提供修复方案。

---

## 少样本学习

通过提供输入-输出示例，帮助 Claude 理解期望的格式和行为：

```python
prompt = """将产品描述转换为结构化数据。

<example>
输入：苹果 MacBook Pro 16英寸，M3 Pro 芯片，36GB 内存，512GB 存储，售价 19999 元
输出：{"brand": "苹果", "product": "MacBook Pro", "screen": "16英寸", "chip": "M3 Pro", "memory": "36GB", "storage": "512GB", "price": 19999}
</example>

<example>
输入：联想 ThinkPad X1 Carbon，14英寸，i7-1365U，16GB 内存，1TB SSD，售价 12999 元
输出：{"brand": "联想", "product": "ThinkPad X1 Carbon", "screen": "14英寸", "chip": "i7-1365U", "memory": "16GB", "storage": "1TB SSD", "price": 12999}
</example>

现在处理：华为 MateBook X Pro，14.2英寸，Ultra 9，32GB 内存，2TB SSD，售价 14999 元"""
```

也可以通过多轮消息历史提供示例：

```python
messages = [
    {"role": "user", "content": "情感分析：这家餐厅太棒了，食物美味！"},
    {"role": "assistant", "content": "正面"},
    {"role": "user", "content": "情感分析：等了一个小时才上菜，味道一般。"},
    {"role": "assistant", "content": "负面"},
    {"role": "user", "content": "情感分析：东西还行，价格有点贵，环境不错。"},
]
```

---

## 提示缓存

提示缓存允许将频繁使用的提示词前缀缓存在服务器上，降低成本并减少延迟：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": "大量上下文内容..." * 200,  # 长且稳定的内容
            "cache_control": {"type": "ephemeral"}  # 标记为可缓存
        }
    ],
    messages=[{"role": "user", "content": "基于以上内容，总结关键点。"}]
)

# 检查缓存使用情况
print(f"缓存创建 tokens: {message.usage.cache_creation_input_tokens}")
print(f"缓存命中 tokens: {message.usage.cache_read_input_tokens}")
```

**缓存费用**：写入为基准价 x 1.25，读取为基准价 x 0.1。多次使用时可节省高达 **90%** 的输入成本。最小缓存长度：1024 tokens（Sonnet/Opus）或 2048 tokens（Haiku）。

---

## 提示词技巧总结

| 技巧 | 适用场景 | 效果 |
|------|----------|------|
| 系统提示词 | 定义角色和全局行为 | 保持一致的输出风格 |
| XML 标签 | 结构化输入和输出 | 提高指令理解准确度 |
| 思维链 | 复杂推理和分析 | 提高答案准确性 |
| 少样本学习 | 特定格式的输出 | 精确控制输出格式 |
| 预填充 | 引导输出格式 | 确保输出以特定格式开始 |
| 提示缓存 | 大量重复上下文 | 降低成本和延迟 |

---

## 关键要点

- 系统提示词定义 Claude 的角色和行为，对整个对话持续生效
- 多轮对话需传递完整消息历史，角色必须交替出现
- XML 标签是组织复杂提示词的利器，Claude 能准确理解标签语义
- 思维链提示显著提升 Claude 在复杂推理任务中的表现
- 少样本学习通过示例教会 Claude 期望的输出格式
- 提示缓存可节省高达 90% 的输入成本

---

[← 上一课：API 基础](./01-api-fundamentals.md) | [返回目录](./README.md) | [下一课：工具使用 →](./03-tool-use.md)
