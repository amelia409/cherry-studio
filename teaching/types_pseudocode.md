// @types.ts 核心逻辑伪代码

// 本文件包含文件系统操作的核心工具函数，主要包括路径安全验证、模糊匹配替换策略及二进制文件检测。

// ==========================================
// 1. 路径安全验证 (Security Validation)
// ==========================================
// 目的：确保访问的路径在允许的范围内，防御路径遍历攻击。

Function ValidatePath(requestedPath, baseDir):
    // 1. 展开波浪号 (~) 为用户主目录
    expandedPath = ExpandHome(requestedPath)

    // 2. 确定基准目录 (默认为当前工作目录)
    root = baseDir OR CurrentWorkingDirectory()

    // 3. 解析为绝对路径
    If IsAbsolute(expandedPath):
        absolutePath = Resolve(expandedPath)
    Else:
        absolutePath = Resolve(root, expandedPath)

    // 4. 解析符号链接 (Symlinks) 并规范化
    Try:
        realPath = GetRealPath(absolutePath)
        Return NormalizePath(realPath)
    Catch Error:
        // 如果文件不存在（例如新建文件），则验证其父目录
        parentDir = GetParentDir(absolutePath)
        Try:
            realParent = GetRealPath(parentDir)
            Return NormalizePath(absolutePath) // 允许路径，只要父目录有效
        Catch:
            Return NormalizePath(absolutePath) // 兜底返回

// ==========================================
// 2. 模糊匹配替换逻辑 (Fuzzy Matching Logic)
// ==========================================
// 目的：在文件内容中查找 old_string，即使存在格式差异（如缩进、空白）也能匹配。
// 策略：按优先级尝试多种匹配算法，直到找到匹配项。

// 定义所有匹配策略（按优先级排序）
ReplacerStrategies = [
    SimpleReplacer,               // 1. 精确匹配
    LineTrimmedReplacer,          // 2. 忽略行首尾空白
    BlockAnchorReplacer,          // 3. 基于首尾行的块匹配 (容忍中间内容的差异)
    WhitespaceNormalizedReplacer, // 4. 归一化所有空白 (连续空格视为一个空格)
    IndentationFlexibleReplacer,  // 5. 灵活缩进匹配
    EscapeNormalizedReplacer,     // 6. 归一化转义字符
    TrimmedBoundaryReplacer,      // 7. 去除整体首尾空白
    ContextAwareReplacer,         // 8. 上下文感知匹配 (基于相似度阈值)
    MultiOccurrenceReplacer       // 9. 多处出现匹配 (用于 replace_all)
]

Function ReplaceWithFuzzyMatch(content, oldString, newString, replaceAll):
    // 前置检查
    If oldString EQUALS newString:
        Throw Error("old_string and new_string must be different")

    notFound = True

    // 遍历每种匹配策略
    For Each strategy In ReplacerStrategies:
        // 获取该策略下的所有候选匹配项
        matches = strategy(content, oldString)

        For Each matchStr In matches:
            index = IndexOf(content, matchStr)
            If index == -1 Continue

            notFound = False

            // 情况 A: 替换所有匹配项
            If replaceAll Is True:
                Return ReplaceAll(content, matchStr, newString)

            // 情况 B: 替换单个匹配项 (必须保证唯一性)
            lastIndex = LastIndexOf(content, matchStr)
            If index != lastIndex:
                // 找到多个匹配项，不安全，跳过当前策略，尝试下一个或报错
                Continue 
            
            // 找到唯一的匹配项，执行替换
            Return ReplaceSingle(content, index, matchStr, newString)

    // 结果处理
    If notFound Is True:
        Throw Error("old_string not found in content")
    Else:
        // 曾找到匹配项，但因为非唯一性 (Unique) 检查失败而未执行替换
        Throw Error("Found multiple matches. Please provide more context.")

// ------------------------------------------
// 核心策略示例：BlockAnchorReplacer (块锚点匹配)
// ------------------------------------------
// 适用于代码块匹配，只要第一行和最后一行匹配，且中间内容相似度足够高即可。
Function BlockAnchorReplacer(content, find):
    searchLines = SplitLines(find)
    If searchLines.Length < 3: Return // 行数太少不适用此策略

    firstLine = searchLines.First
    lastLine = searchLines.Last
    
    // 1. 在内容中查找所有可能的起始和结束行组合
    candidates = FindCandidates(content, firstLine, lastLine)

    // 2. 验证中间内容的相似度 (使用 Levenshtein 编辑距离)
    bestMatch = Null
    maxSimilarity = -1

    For Each candidate In candidates:
        similarity = CalculateBlockSimilarity(candidate, searchLines)
        
        // 单个候选者阈值宽松，多个候选者阈值严格
        threshold = (candidates.Length == 1) ? 0.0 : 0.3

        If similarity >= threshold AND similarity > maxSimilarity:
            maxSimilarity = similarity
            bestMatch = candidate

    If bestMatch Is Not Null:
        Yield ExtractString(content, bestMatch)

// ==========================================
// 3. 二进制文件检测 (Binary Detection)
// ==========================================
// 目的：读取文件头，通过字节分析判断是否为文本文件。

Function IsBinaryFile(filePath):
    buffer = ReadFirst4096Bytes(filePath)
    if BytesRead == 0 Returns False // 空文件视为文本

    // 统计各类字节特征
    zeroBytes = Count(0x00)
    nonPrintable = Count(NonPrintableASCII)
    
    // 特征 A: 包含过多 NULL 字节
    If zeroBytes / TotalBytes > 0.05:
        // 排除 UTF-16 编码 (UTF-16 通常每隔一个字节是 0)
        If IsUTF16Pattern(buffer): Return False
        Return True

    // 特征 B: 非打印字符比例过高
    If nonPrintable / TotalBytes > 0.3:
        Return True

    Return False
