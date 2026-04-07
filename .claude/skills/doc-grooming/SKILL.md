---
name: doc-grooming-orchestrator
description: 基于 generalist 容器与文件交接范式的文档自动化流。通过注入 Persona 配置文件，实现子智能体的角色扮演与循环博弈。当用户要求“生成文档”、“更新文档”、“分析代码并写文档”时，或者当涉及到复杂的、需要审查的系统架构、核心机制或公共组件的文档编写时使用此技能。
---

# 🏗️ 文档自动化与质量监控工作流 (Container & File Handoff SOP)

**系统指令**：你是调度中枢（Orchestrator）。根据系统底层架构，你必须使用 `generalist` 作为物理执行容器，并通过向其注入（Inject）特定的系统提示词（Persona），来让其扮演特定的专家角色。

## 📦 输入参数 (Input Parameters)
- **Target_Path**: 需要分析的代码文件或目录路径。
- **Doc_Type**: 文档类型（如：架构文档、API手册、机制说明等）。
- **Extra_Context**: (可选) 用户补充的特定要求或背景信息。

## 📂 核心资源路径 (Shared Assets)
- **写手人设**：`.gemini/agents/doc-writer.md`
- **审计人设**：`.gemini/agents/doc-validator.md`
- **共享草稿**：`.temp_doc_draft.md` （所有输出必须写入此文件，禁止在主会话框大量输出）

## ⚡️ 强制执行阶段 (Execution Phases)

### 🔴 阶段 0：前置校验 (Pre-check)
**动作**：确保基础环境就绪。
- **操作规范**：
  1. 检查需要分析的源文件或目录是否存在。
  2. 确保 `.gemini/agents/` 下的两个 Persona 配置文件存在且可读。
  3. 如果 `.temp_doc_draft.md` 存在历史内容，视情况进行覆盖或向用户确认是否清空。

### 🔴 阶段 1：注入 Writer 角色并生成文件
**动作**：调度 `generalist` 容器。
- **操作规范**：
  1. 请先读取 `.gemini/agents/doc-writer.md` 获取角色要求。
  2. 向 `generalist` 发送委派指令：“*你现在的身份是 doc-writer（参考已读取的规则）。请分析指定的代码路径，绝对不要在对话框中输出文档，必须使用工具将完整的带 Mermaid 的 Markdown 文档写入 `.temp_doc_draft.md`。*” （需附带对应的路径与上下文）

*(等待 generalist 回复写入完成后，进入阶段 2)*

### 🔴 阶段 2：注入 Validator 角色并审计
**动作**：再次调度一个新的 `generalist` 容器。
- **操作规范**：
  1. 请先读取 `.gemini/agents/doc-validator.md` 获取角色要求。
  2. 向 `generalist` 发送委派指令：“*当前进度 Round {{Current_Round}}。你现在的身份是 doc-validator（参考已读取的规则）。请调用工具读取 `.temp_doc_draft.md` 的内容，并针对其内容输出严苛的【审计报告】与【修改 Action Items】。若完全符合规范，仅回复 [Pass]。*”

### 🔴 阶段 3：异步迭代博弈 (Iterative Handoff)
- **如果结论是 [Pass]**：跳出循环，进入阶段 4。
- **如果结论包含修改意见**：
  - 动作：再次调度 `generalist`（重新注入 Writer 角色）。
  - **指令**：“*身份切换为 doc-writer。这是审计员的反馈：[粘贴意见]。请读取 `.temp_doc_draft.md`，根据意见修改后，将最新内容覆盖保存回该文件。*”
  - *(最高循环 3 次，超过则强制熔断)*

### 🔴 阶段 4：交付与归档 (Delivery & Archiving)
- 终止循环。告知用户文档已定稿于 `.temp_doc_draft.md`，并输出最终的审计结论摘要。
- 询问用户是否需要将该草稿移动并重命名至正式的文档归档目录（如 `Assets/Doc/` 或根目录的 `README` 等）。

## 🛡️ 异常处理与熔断机制 (Exception Handling & Fallback)
1. **容器执行失败**：如果在任何阶段 `generalist` 执行报错，调度中枢需记录错误，向用户简要报告，并可尝试重试一次当前阶段。
2. **死循环熔断**：如果 Validator 连续多次提出相同或无法解决的修改意见，或总循环达到 3 次，必须强制停止迭代，并将当前的草稿及未解决的 Action Items 提交给用户进行人工判断。
3. **内容超限**：提醒 Writer 在处理大型代码库时，优先采用分模块输出策略（如先输出大纲框架，再逐步填充特定章节）。
