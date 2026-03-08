# 前端工具渲染 (Agent Tools Rendering) 实习生开发指南

本指南旨在指导开发人员（特别是实习生）如何在 Cherry Studio 前端实现和优化 Agent 工具（特别是文件系统工具）的渲染逻辑。

## 1. 整体架构与流程

Agent 工具的渲染遵循从 **原始 JSON 数据** 到 **结构化 UI 组件** 的转换流程：

1.  **数据接收**: 后端通过流式或全量方式发送工具调用信息（`tool_use`）和结果（`tool_result`）。
2.  **分发逻辑**: `MessageAgentTools/index.tsx` 作为入口，根据 `toolName` 将数据分发给对应的渲染组件。
3.  **包装容器**: 每个工具渲染器通常返回一个 `Ant Design` 的 `Collapse` 项，包含 `label` (标题栏) 和 `children` (内容区)。

---

## 2. 核心公共组件 (`GenericTools.tsx`)

在开发新工具渲染前，必须熟悉以下基础组件以保持 UI 一致性：

- **`ToolHeader`**: 统一的标题栏。支持显示工具名称、运行参数（如文件名）及右侧的状态指示器。
- **`SkeletonValue`**: 流式渲染利器。当工具还在运行时（`isStreaming`），显示骨架屏；数据到达后自动切换为真实值。
- **`ToolStatusIndicator`**: 状态灯。根据工具状态（`invoking`, `done`, `error`, `waiting`）显示不同的图标和颜色。
- **`TruncatedIndicator`**: 内容截断提示。用于文件内容过大被后端截断时，提示用户“还有更多内容”。

---

## 3. 文件系统工具渲染实例

### 3.1 `ReadTool` (读文件)
- **参数解析**: 从路径中提取文件名展示在标题栏，计算行数和文件大小显示为统计信息。
- **内容处理**: 
    - 使用 `ReactMarkdown` 渲染文件内容（支持代码高亮）。
    - 调用 `truncateOutput` 处理超长文本，避免撑爆主线程渲染。
    - 清理标签：调用 `removeSystemReminderTags` 过滤掉内部使用的提示标记。

### 3.2 `EditTool` (修改文件)
- **差异对比 (Diff View)**: 不同于简单的文本显示，`EditTool` 需要展示修改前后的对比。
- **UI 实现**:
    - `variant="old"`: 红色背景 + 删除线，前面带 `-` 符号。
    - `variant="new"`: 绿色背景，前面带 `+` 符号。
    - 均使用等宽字体 (`font-mono`) 和 行号展示。

### 3.3 `WriteTool` & `GlobTool`
- 相对简单，侧重于展示目标路径和操作结果的预览。

---

## 4. 开发一个新工具渲染器的步骤 (Checklist)

1.  **定义类型**: 在 `types.ts` 中添加工具的 `Input` 和 `Output` 类型定义。
2.  **注册渲染器**: 在 `MessageAgentTools/index.tsx` 的 `toolRenderers` 映射中添加对应条目。
3.  **编写组件**:
    - 使用 `useTranslation` 处理国际化。
    - 标题栏 (`label`)：使用 `ToolHeader`，参数部分使用 `SkeletonValue` 包装。
    - 内容区 (`children`)：根据工具特性选择展示方式（代码块、列表、Markdown 等）。
4.  **处理流式状态**: 确保在工具运行时（Status 为 `pending` 或 `invoking`）组件能优雅地展示（如骨架屏或 Loading 状态），而不是显示一片空白。

---

## 5. 注意事项与最佳实践

- **性能优化**: 对于大数据量的输出（如 `grep` 结果），必须实现分页或截断显示。
- **安全性**: 不要直接使用 `dangerouslySetInnerHTML`。优先使用 `ReactMarkdown` 或 文本转义。
- **用户反馈**: `ToolStatusIndicator` 必须准确反映后端的 `status`，特别是当工具报错时，要能通过 UI 清楚地告知用户。
- **代码重用**: 复杂的代码展示逻辑应提取到 `GenericTools` 或 `shared` 目录下。

---

## 6. 参考文件目录

- 逻辑分发: `@renderer/pages/home/Messages/Tools/MessageAgentTools/index.tsx`
- 公共组件: `@renderer/pages/home/Messages/Tools/MessageAgentTools/GenericTools.tsx`
- 读文件示例: `@renderer/pages/home/Messages/Tools/MessageAgentTools/ReadTool.tsx`
- 修改文件示例: `@renderer/pages/home/Messages/Tools/MessageAgentTools/EditTool.tsx`
