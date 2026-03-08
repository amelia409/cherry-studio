# 模型主动提问 (AskUserQuestion) 功能实现指南

本文档详细讲解了如何实现 "模型向用户提问" (AskUserQuestion) 的完整链路，包括后端工具拦截、前端卡片渲染以及用户反馈的通信流程。

## 1. 核心流程概述

1.  **模型发起工具调用**: Agent 决定需要用户输入，调用 `AskUserQuestion` 工具。
2.  **后端拦截与挂起**: 主进程 (`Main`) 拦截该工具调用，将其作为"权限请求"挂起，并向前端发送 IPC 消息。
3.  **前端渲染卡片**: 渲染进程 (`Renderer`) 收到请求，展示 `AskUserQuestionCard` 组件，等待用户操作。
4.  **用户提交回复**: 用户选择选项或输入文本后点击提交，前端通过 IPC 发送回复。
5.  **后端恢复执行**: 主进程收到回复，将其作为工具执行结果返回给 Agent，Agent 继续后续逻辑。

---

## 2. 后端实现 (Main Process)

后端主要负责工具的**权限管理**与**挂起机制**。

### 2.1 工具定义 (Schema)

虽然此工具通常内置于 Agent SDK，但其逻辑定式如下：

```typescript
// Tool Definition
const AskUserQuestionTool = {
  name: "AskUserQuestion",
  description: "Ask the user a question to get more information or confirmation.",
  inputSchema: {
    type: "object",
    properties: {
      questions: {
        type: "array",
        items: {
          type: "object",
          properties: {
            question: { type: "string" },
            header: { type: "string" }, // 短标签
            options: {                   // 预设选项
              type: "array",
              items: {
                type: "object",
                properties: { label: { type: "string" }, description: { type: "string" } }
              }
            },
            multiSelect: { type: "boolean" }
          }
        }
      }
    }
  }
}
```

### 2.2 拦截与挂起逻辑 (Pseudocode)

参考 `src/main/services/agents/services/claudecode/tool-permissions.ts`：

```typescript
// 当 Agent 调用工具时触发
function promptForToolApproval(toolName, input, options) {
  // 1. 创建唯一的请求 ID
  const requestId = randomUUID()
  
  // 2. 将请求存入 pendingRequests Map 中挂起
  pendingRequests.set(requestId, {
     fulfill: resolvePromise, // 用于后续恢复执行
     toolName,
     originalInput: input
  })

  // 3. 向前端发送 IPC 消息，通知UI渲染
  broadcastToRenderer('agent:tool-permission:request', {
     requestId,
     toolName,
     input,
     requiresPermissions: true
  })

  // 4. 返回 Promise，使当前 Agent 执行流暂停
  return new Promise((resolve) => {
     // 等待前端响应...
  }) 
}
```

### 2.3 处理前端回复

```typescript
// 监听前端的回复 IPC
ipcMain.handle('agent:tool-permission:response', (event, payload) => {
  const { requestId, behavior, updatedInput } = payload
  
  // 1. 查找挂起的请求
  const pending = pendingRequests.get(requestId)
  if (!pending) return { success: false }

  // 2. 恢复 Promise 执行
  if (behavior === 'allow') {
     // 将用户的回答 (updatedInput) 作为工具结果返回给 Agent
     pending.fulfill({ 
       behavior: 'allow', 
       updatedInput: updatedInput 
     })
  } else {
     pending.fulfill({ behavior: 'deny' })
  }

  // 3. 清理状态
  pendingRequests.delete(requestId)
  return { success: true }
})
```

---

## 3. 前端实现 (Renderer Process)

前端负责展示交互卡片，并收集用户输入。

### 3.1 卡片组件逻辑 (`AskUserQuestionCard.tsx`)

该组件处理两种状态：`Pending` (等待输入) 和 `Completed` (已显示结果)。

**核心逻辑参考：**

```typescript
export function AskUserQuestionCard({ toolResponse }) {
  // 1. 获取当前挂起的权限请求
  const request = useAppSelector(selectPendingPermission(toolResponse.toolCallId))
  const isPending = toolResponse.status === 'pending' && !!request

  // 2. 解析问题数据
  // 如果是 pending 状态，数据来自 request.input
  // 如果是 completed 状态，数据来自 toolResponse.arguments
  const source = isPending ? request.input : toolResponse.arguments
  const questions = source.questions

  // 3. 处理用户输入状态
  const [answers, setAnswers] = useState({}) 

  // 4. 提交逻辑
  const handleSubmit = async () => {
    // 构造回复数据
    const responsePayload = {
      requestId: request.requestId,
      behavior: 'allow',
      updatedInput: { 
        ...request.input, 
        answers: answers // 将用户填写的答案注入
      }
    }

    // 调用 IPC 发送给后端
    await window.api.agentTools.respondToPermission(responsePayload)
  }

  // 5. 渲染 UI
  return (
    <Card>
      {questions.map(q => (
        <div key={q.question}>
           <h3>{q.question}</h3>
           {/* 渲染选项或输入框 */}
           <OptionsList 
             options={q.options} 
             onSelect={(val) => setAnswers({...answers, [q.question]: val})} 
           />
        </div>
      ))}
      {isPending && <Button onClick={handleSubmit}>Submit</Button>}
    </Card>
  )
}
```

### 3.2 关键交互点

*   **多选与单选**: 根据 `multiSelect` 属性决定渲染 Radio 还是 Checkbox。
*   **"Other" 选项**: 允许用户输入自定义文本，通常作为补充选项。
*   **分页**: 如果有多个问题，通常使用分页向导形式 (Step-by-step) 展示。

---

## 4. 总结：数据流向图

```text
[Agent] 
   |
   +-- 调用 ask_user_question({"question": "Confirm deploy?"})
         |
      [Main Process] (工具拦截)
         |
         +-- 挂起执行 (Promise Pending)
         +-- 发送 IPC: agent:tool-permission:request
              |
           [Renderer Process]
              |
              +-- 显示 <AskUserQuestionCard />
              +-- 用户点击 "Yes"
              +-- 发送 IPC: agent:tool-permission:response
                  (带上 payload: { answers: { "Confirm deploy?": "Yes" } })
              |
      [Main Process]
         |
         +-- 接收响应
         +-- 恢复 Promise (Resolve with updatedInput)
         |
   +-- 收到工具结果: { "result": "Yes" }
   |
[Agent] 继续执行部署...
```
