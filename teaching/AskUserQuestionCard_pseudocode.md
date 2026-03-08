// @AskUserQuestionCard.tsx 核心逻辑伪代码

// 本组件用于渲染 Agent 向用户发起提问的卡片界面。
// 支持单选、多选、自定义输入 ("Other"选项)，并处理多问题分页与提交。

// ==========================================
// 1. 组件定义与状态初始化
// ==========================================

Component AskUserQuestionCard(toolResponse):
    // 获取权限请求状态 (Pending Check)
    request = GetPendingPermission(toolResponse.toolCallId)
    isPending = (toolResponse.status == 'pending' AND request Exists)

    // 1.1 数据解析 (Data Parsing)
    // 优先使用请求中的 input，否则回退到 toolResponse 的 arguments
    source = isPending ? request.input : toolResponse.arguments
    { questions, answers } = ParseInput(source)

    // 1.2 状态定义 (State Definitions)
    state currentIndex = 0                           // 当前显示第几个问题
    state selectedAnswers = {}                       // 记录每个问题的选中项 (Map<Index, List<String>>)
    state customInputs = {}                          // 记录"其他"选项的输入内容 (Map<Index, String>)
    state showCustomInput = {}                       // 记录是否显示"其他"输入框 (Map<Index, Boolean>)
    state submittedAnswers = {}                      // 本地保存提交后的答案

    // 决定当前展示的答案 (优先显示已有答案，否则显示刚提交的)
    displayAnswers = IsNotEmpty(answers) ? answers : submittedAnswers

    // 1.3 辅助变量
    currentQuestion = questions[currentIndex]
    totalQuestions = questions.length
    isLastQuestion = (currentIndex == totalQuestions - 1)

// ==========================================
// 2. 交互逻辑处理 (Interaction Handlers)
// ==========================================

    // 处理选项选择
    Function HandleSelect(questionIndex, label, checked):
        isMulti = questions[questionIndex].multiSelect

        // 情况 A: 选择了 "Other" (其他)
        If label == "__other__":
            showCustomInput[questionIndex] = checked OR True // 切换显示状态
            If NOT showCustomInput[questionIndex]:
                ClearCustomInput(questionIndex)              // 隐藏时清空内容
            
            // 如果是单选模式，选择"其他"时需清空常规选项
            If NOT isMulti:
                selectedAnswers[questionIndex] = []
            Return

        // 情况 B: 常规选项 (多选模式)
        If isMulti:
            If checked:
                Add(selectedAnswers[questionIndex], label)
            Else:
                Remove(selectedAnswers[questionIndex], label)
        
        // 情况 C: 常规选项 (单选模式)
        Else:
            showCustomInput[questionIndex] = False    // 隐藏"其他"输入框
            selectedAnswers[questionIndex] = [label]  // 设置选中项 (互斥)
            ClearCustomInput(questionIndex)           // 清空自定义输入

    // 处理提交
    Function HandleSubmit():
        If request is Null: Return

        // 收集所有答案
        collectedAnswers = {}
        For Each (question, idx) In questions:
            selected = selectedAnswers[idx]
            custom = customInputs[idx]

            // 组合答案逻辑：
            // 1. 仅自定义: "User custom text"
            // 2. 仅选项: "Option A"
            // 3. 多选混合: "Option A, User custom text"
            answerText = FormatAnswer(selected, custom)
            collectedAnswers[question.text] = answerText

        // 更新状态并发送给主进程
        SetSubmittedAnswers(collectedAnswers)
        Dispatch(SubmissionSent Action)

        Try:
            // 调用 API 响应权限请求
            Result = CallBackendAPI('respondToPermission', {
                requestId: request.requestId,
                behavior: 'allow',
                updatedInput: { answers: collectedAnswers } // 将答案回填
            })
            If Result.success is False: Throw Error
        Catch:
            ShowToast("Submission Failed")
            Dispatch(SubmissionFailed Action)

// ==========================================
// 3. 渲染逻辑 (Render Logic)
// ==========================================

    // 主渲染流
    Render:
        Container:
            // 3.1 头部信息 (Header)
            Header(
                Title: "My Agent Question",
                Progress: "${currentIndex + 1} / ${totalQuestions}"
            )

            // 3.2 内容区域 (Content)
            If isPending:
                // 待回答状态
                RenderPendingContent(
                    OptionList: 
                        RenderOptions(currentQuestion.options, mode=Single/Multi)
                        RenderOtherOption() // 特殊渲染"其他"选项
                    CustomInput: 
                        If showCustomInput[currentIndex]: RenderTextInput()
                )
            Else:
                // 已完成状态
                RenderCompletedContent(
                    QuestionText: currentQuestion.text,
                    AnswerTags: displayAnswers[currentQuestion.text]
                )

            // 3.3 底部导航 (Navigation)
            Footer:
                If totalQuestions > 1:
                    Button("Previous") -> onClick: currentIndex--
                
                If isPending:
                    If isLastQuestion:
                        Button("Submit") -> onClick: HandleSubmit()
                    Else:
                        Button("Next") -> onClick: currentIndex++
                Else:
                    // 非 Pending 状态只允许查看
                    Button("Next") -> onClick: currentIndex++ (Disabled if last)
