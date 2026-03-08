// @edit.ts 核心逻辑伪代码

// 1. 输入验证
// 接收参数:
// - file_path: 文件路径 (字符串)
// - old_string: 要替换的旧文本 (字符串)
// - new_string: 替换后的新文本 (字符串)
// - replace_all: 是否替换所有匹配项 (布尔值, 默认为 false)

Function HandleEditTool(args, baseDir):
    // 解析并验证参数
    parsedArgs = Validate(args, EditToolSchema)
    filePath = parsedArgs.file_path
    oldString = parsedArgs.old_string
    newString = parsedArgs.new_string
    replaceAll = parsedArgs.replace_all

    // 2. 路径处理
    // 将相对路径转换为绝对路径，并验证安全性
    fullPath = ResolvePath(filePath, baseDir)

    // 3. 文件检查与创建逻辑
    If FileExists(fullPath) returns False:
        // 如果文件不存在，且明确指定旧文本为空字符串，则视为新建文件
        If oldString is Empty:
            CreateDirectory(ParentOf(fullPath)) // 确保父目录存在
            WriteFile(fullPath, newString)       // 创建文件并写入内容
            Log("File created at " + fullPath)
            Return "Created new file: " + RelativePath(fullPath)
        Else:
            // 如果文件不存在但试图替换内容，抛出错误
            Throw Error("File not found: " + filePath)

    // 4. 读取现有文件内容
    content = ReadFile(fullPath)

    // 5. 覆盖写逻辑
    // 如果旧文本为空字符串，直接用新内容覆盖整个文件
    If oldString is Empty:
        WriteFile(fullPath, newString)
        Log("File overwritten at " + fullPath)
        Return "Overwrote file: " + RelativePath(fullPath)

    // 6. 内容查找与替换
    // 使用模糊匹配算法查找旧文本并替换为新文本
    // 注意：实际实现中可能包含缩进处理、多行匹配等复杂逻辑
    newContent = ReplaceWithFuzzyMatch(content, oldString, newString, replaceAll)

    // 7. 写入修改后的内容
    WriteFile(fullPath, newContent)

    // 8. 生成差异摘要
    lineDiff = CountLines(newContent) - CountLines(content)
    diffSummary = "Edited: " + RelativePath(fullPath)
    If lineDiff > 0:
        diffSummary += " (+" + lineDiff + " lines)"
    Else If lineDiff < 0:
        diffSummary += " (" + lineDiff + " lines)"

    Log("File edited at " + fullPath)
    Return diffSummary
