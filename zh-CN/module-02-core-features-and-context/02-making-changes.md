# 课时 5：进行代码修改

[← 上一课：添加上下文](./01-adding-context.md) | [返回目录](../README.md) | [下一课：控制上下文 →](./03-controlling-context.md)

---

## 核心概念

Claude Code 最强大的能力之一就是直接在你的代码库中进行修改。本课时将介绍如何高效地使用 Claude Code 进行文件编辑、代码生成和多文件操作。

## 基本修改工作流

当你要求 Claude Code 修改代码时，它会遵循一个标准流程：

```
理解需求 → 阅读相关文件 → 规划变更 → 执行修改 → 验证结果
```

### 示例：修复一个 Bug

```
> 用户报告了一个问题：当购物车为空时点击结算按钮会报错。帮我修复。
```

Claude Code 的执行过程：

1. **搜索相关代码**：使用 Glob 和 Grep 找到购物车和结算相关的文件
2. **阅读源代码**：使用 Read 工具查看具体实现
3. **定位问题**：分析代码逻辑，找到缺少空值检查的地方
4. **修复代码**：使用 Edit 工具添加检查逻辑
5. **验证修复**：如果有测试，运行测试确认修复有效

## Edit 工具：精确编辑

Claude Code 使用 Edit 工具进行**精确的字符串替换**，而不是重写整个文件。这意味着：

- 只修改需要变更的部分
- 保留文件其他内容不变
- 变更可以清晰地 diff 审查

```
> 把 calculatePrice 函数从同步改成异步的

Claude Code 会：
- 读取包含 calculatePrice 的文件
- 找到函数定义
- 用 Edit 工具将 function calculatePrice() 替换为 async function calculatePrice()
- 更新函数体中的相关调用
- 更新所有调用点以添加 await
```

## 多文件操作

Claude Code 可以跨多个文件进行协调的变更：

### 场景：重命名一个组件

```
> 把 UserCard 组件重命名为 ProfileCard，包括文件名、导入语句和所有引用
```

Claude Code 会自动：
1. 重命名组件文件
2. 更新组件内部的名称
3. 搜索所有导入该组件的文件
4. 更新所有导入语句和使用处

### 场景：添加一个新功能

```
> 在用户设置页面添加一个深色模式切换开关，使用现有的 ThemeContext
```

Claude Code 会：
1. 读取 ThemeContext 了解现有的主题实现
2. 读取设置页面了解布局和样式约定
3. 创建或修改必要的组件
4. 更新设置页面以包含新的切换开关
5. 确保与现有样式和模式一致

## 代码生成

Claude Code 不仅可以修改现有代码，还擅长生成新代码：

### 生成测试

```
> 为 src/utils/validation.ts 中的所有函数编写单元测试
```

Claude Code 会：
1. 读取源文件，理解每个函数的签名和逻辑
2. 了解项目的测试框架（Jest、Vitest 等）
3. 创建测试文件，包含正常用例和边界用例
4. 运行测试确认通过

### 生成 API 端点

```
> 为用户管理创建 CRUD API，包含列表、详情、创建、更新和删除
```

Claude Code 会根据你的项目结构和约定，生成完整的 API 实现。

## 使用 Git 进行变更管理

Claude Code 可以直接使用 Git 工具：

```
> 把我的修改提交一下，用有描述性的 commit message
```

Claude Code 会：
1. 查看 `git diff` 了解变更内容
2. 暂存相关文件
3. 编写描述性的 commit message
4. 创建 commit

更复杂的 Git 操作：

```
> 为这个功能创建一个新分支，提交代码，然后创建 PR
```

## 审查变更

Claude Code 执行修改时，你可以在多个层面进行审查：

### 1. 实时确认

对于可能有风险的操作，Claude Code 会请求确认：

```
Claude wants to edit: src/database/schema.ts
Allow? (y/n/always)
```

### 2. 使用 VS Code/JetBrains 集成

在 IDE 中使用 Claude Code 时，你可以看到**内联 diff**，直观地看到每一处变更。

### 3. 事后审查

修改完成后，使用 Git 审查所有变更：

```
> 显示一下这次所有的变更摘要

或者直接：
git diff
```

## 高效修改的技巧

### 描述意图而非步骤

**好的方式：**
```
> 这个函数太慢了，在处理大数据集时会卡顿。帮我优化它。
```

**不太好的方式：**
```
> 打开 src/utils/data.ts，在第 42 行添加一个缓存变量，
  然后在第 50 行添加一个检查...
```

让 Claude Code 发挥它的能力来确定最佳实现方案。

### 提供错误信息

当修复 Bug 时，直接粘贴错误信息非常有效：

```
> 运行测试时报了这个错误，帮我修复：

TypeError: Cannot read properties of undefined (reading 'map')
  at UserList (src/components/UserList.tsx:15:23)
  at renderWithHooks (node_modules/react-dom/...)
```

### 指定范围

当你知道修改范围时，明确指出：

```
> 只修改 src/api/ 目录下的文件，不要动前端代码
```

### 请求解释

如果不确定修改是否正确：

```
> 在实际修改之前，先解释一下你打算怎么改，以及为什么
```

## 实用技巧

1. **先让 Claude 分析，再让它修改**：对于复杂问题，先问"你觉得问题出在哪里"
2. **小步迭代**：一次处理一个明确的任务，而不是一口气做很多事
3. **利用 Git**：在大的修改前创建分支，方便回滚
4. **检查测试**：修改后让 Claude 运行测试确认没有引入新问题
5. **审查 diff**：养成检查 Claude 生成的变更的习惯

---

## 关键要点

- Claude Code 通过 Edit 工具进行精确的字符串替换，保留文件其他内容
- 支持跨多个文件的协调变更（重命名、重构等）
- 能够生成新代码（测试、API、组件等）
- 可以直接操作 Git 进行提交、创建分支等
- 最佳实践：描述意图、提供上下文、小步迭代

---

[← 上一课：添加上下文](./01-adding-context.md) | [返回目录](../README.md) | [下一课：控制上下文 →](./03-controlling-context.md)
