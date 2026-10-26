# AGENTS.md — Lessons Learned

> 从本项目的错误中总结的规则，避免重复犯错。

## 编码与字符 (Encoding)

- **永远用 UTF-8 读/写文件**。PowerShell 的 Get-Content / Set-Content 默认可能用 ANSI，必须显式指定 -Encoding UTF8。
- **中文字符串要逐处验证**。修复 HTML 元素中的中文后，还要同步检查 JS 字符串字面量、CSS 注释等所有出现中文的地方。不同上下文中的中文可能被不同方式损坏。
- **使用字节级替换修复损坏的中文**。如果文本已经含 U+FFFD（替换字符），字符串级别的 .Replace() 无法匹配，需要用 [System.IO.File]::ReadAllBytes / WriteAllBytes 做字节级替换。

## 服务器与端口 (Server & Port)

- **启动后台服务器用 Start-Process -WindowStyle Hidden -PassThru**。Start-Job、WScript.Shell.Run、& 运算符等方式的进程会在 exec_command 超时或结束后被杀死。
- **验证服务器是否真在运行用 curl，不要用 Invoke-WebRequest**。沙箱可能阻止 PowerShell 对 localhost 的 HTTP 请求，但 curl 和浏览器能正常访问。
- **修改端口后同步更新 .bat 文件和 start.js**。用户浏览器可能还停留在旧端口。
- **如果原端口被其他进程占用无法杀死，直接换端口**。

## HTML/JS 编辑 (HTML/JS Editing)

- **不要用固定行号修改代码**。在多次编辑过程中行号会变化。应该用字符串匹配（-match）来定位目标。
- **添加 try/catch 和 console.log 到关键路径**。用户点击按钮无反应时，console 里的错误日志是最快的诊断方式。
- **每次非平凡编辑后运行 
ode --check 验证 JS 语法**。
- **媒体文件（.wasm, .tflite, .data）必须确认 MIME 类型正确**，否则浏览器会静默失败。

## 错误排查顺序 (Debugging Priority)

1. 浏览器 Console（F12）— 看有没有 404 或 JS 错误
2. 检查服务器是否在监听正确端口
3. 检查媒体模型文件是否能通过 HTTP 正确下载
4. 检查 JS 字符串是否因编码问题包含 U+FFFD


## 代码清理 (Code Cleanup)

- **清理重复行时，必须用 curl 或浏览器验证 HTML 结构完整性**。删除重复行很容易误删关键元素（如 <div id="dots">），导致 JS 在页面加载时就崩溃，事件监听器根本绑不上。
- **添加 try/catch 还不够，还要加 window.onerror 捕获全局 JS 错误**。按钮点不进去可能是页面加载阶段就报错了，不是点击后的问题。
- **每次编辑后，至少验证三件事**：JS 语法（
ode --check）、HTML 关键元素是否存在（Select-String + Context）、服务器响应是否正确。
- **问题修完后立即更新 AGENTS.md**，不要等用户提醒。


## 批处理文件 (Batch Files)

- **.bat 文件必须无 BOM 保存**（UTF-8 without BOM 或 ANSI），否则 @echo off 前会出现乱码导致命令失败。
- **用 start /B 后台启动服务器**，之后用 start http://... 打开浏览器，不用等服务器进程结束。
- **退出时用 PID 精确杀死子进程**，避免 	askkill /im node.exe 误杀其他 node 进程。
- **	imeout /t 2 给服务器足够时间启动**再打开浏览器。



## 中文乱码修复 (Encoding Corruption Fix)

- **修复中文乱码后，用字节级扫描确认所有 U+FFFD 都已清除**。仅修复 HTML 元素中的中文不够，JS 字符串、CSS 注释等所有上下文中的中文都要检查。
- **使用 [System.IO.File]::ReadAllBytes + .UTF8.GetString 读取文件**，然后循环 IndexOf(0xFFFD) 统计所有乱码位置，按优先级逐个修复。
- **JS 字符串中的中文乱码是功能性 bug**（如气泡消息、提示文本），必须优先修复；CSS 注释和代码注释中的乱码是外观性 bug。
- **修复方法**：找到包含乱码的完整行，用正确的中文字符串替换整行（使用字节级 WriteAllBytes），不能用字符串 .Replace() 因为 U+FFFD 无法被模式匹配。
- **修复后用包含中文的 .Contains() 验证**。PowerShell 的 Select-String 可能因编码问题无法匹配中文，应用 $text.Contains('中文') 直接验证。
- **检查文件名是否正确包含中文**。用户的中文文件名保存后不会自动乱码，但文件内的中文字符串可能会因编辑工具编码问题损坏。用 [System.Text.Encoding]::UTF8.GetString([System.IO.File]::ReadAllBytes()) 可靠读取。
## DOM 与文件编码 (DOM & File Encoding)

- **文件头部不允许出现双 BOM**。两个连续 UTF-8 BOM 字节序列（`EF BB BF EF BB BF`）会导致浏览器以怪异模式解析 HTML，部分元素 ID 无法被 `document.querySelector` 识别。保存文件时必须确保只保留一个或无 BOM。
- **所有引用 DOM 元素的变量都应加 null 保护**。使用 `$(selector)` 或 `document.querySelector` 前，要考虑元素可能未找到的情况，用 `if(el)` 或 `el?el.children:null` 模式避免报错。
- **`startCam`、`stopCam` 等关键函数中的 DOM 操作要使用 `getElementById` 而非缓存变量**。因为停止/启动摄像头后 DOM 状态可能变化，缓存变量可能失效。
- **始终用 `python3` 或明确指定编码的工具写文件**。PowerShell 的 Set-Content -Encoding Ascii 会丢失非 ASCII 字符，必须用 -Encoding UTF8。

## MediaPipe WASM模型 (WASM Model Loading)

- **检查所有模型文件的大小**，不要只确认文件存在。0字节的关键数据文件（如 _simd_wasm_bin.data）会让WASM看似加载成功（initialize() resolve）但实际无法处理帧，onResults永远不会触发。
- **.data 文件（非 .wasm 和 .tflite）被WASM内部引用，必须是有效内容**。如果原始下载不完整或缺失，WASM运行时读取空缓冲区导致静默失败。优先删除0字节的.data文件，让WASM走内嵌二进制路径。
- **onResults 回调不触发时，第一步是在回调里加 console.log**。这样可以区分模型没加载和模型加载了但不产生结果两种情况。
- **loop() 中的 send() 返回 Promise，要用 .catch() 捕获异步错误**。只包 try/catch 只能捕获同步抛出的异常，异步rejection会被忽略。

## 文件行尾与替换 (CRLF vs LF)

- **Windows上的HTML/JS文件默认用CRLF (\r\n) 换行**。Python字符串匹配时必须用 \r\n 而不是 \n，否则replace找不到目标。
- **使用 rb 模式读写字节级替换**，避免文本模式下的行尾自动转换导致匹配失败。

## 编辑策略 (Edit Strategy)

- **不要通过 python3 -c 传递包含特殊字符或多行内容的脚本**。PowerShell会错误解释引号、方括号、中文字符。改为写临时 .py 文件后用 python3 temp.py 执行。
- **每次编辑后验证三个层**：JS语法、关键DOM元素、服务器响应。
- **问题修完后立即更新AGENTS.md**，不要等用户提醒。


## DOM 与脚本执行顺序 (DOM vs Script Order)

- **所有 JS 通过 getElementById 引用的 DOM 元素，其 HTML 必须放在 script 标签之前**。如果元素定义在 script 之后，同步脚本执行时 getElementById 返回 null，后续 classList 等方法直接报 TypeError。
- **添加新 HTML 元素时，先检查它在文件中的位置**。新加的结束画面、弹窗等元素默认放在文件末尾，容易掉到 script 后面。
- **验证方法**：在浏览器 Console 中执行 document.getElementById(xxx)，看返回 null 还是元素对象。也可以查看元素和 script 在源码中的相对顺序。
- **页面完全黑屏 + 无任何 UI 可见 + Console 有 TypeError**，优先怀疑 DOM 元素在脚本执行时尚未解析。

## 文件重复 (File Duplication)

- **反复编辑的HTML文件可能被意外拼接成两份**。同一份代码出现两次（两个 <html>、两组相同ID的DOM元素），浏览器会创建重复节点，getElementById 返回第一个，但CSS和事件可能混乱。
- **检查方法**：搜索文件中有几个 <html lang 或 <!DOCTYPE。多余的表示文件已重复。
- **修复方法**：用字节级搜索找到第二个 <html 标签，保留第一个 <html> 到第一个 </html> 之间的内容，丢弃后面的所有内容。
- **#endOverlay 必须在 #cover 之后（DOM顺序）**。如果结束画面在欢迎画面前面，即使用了z-index（40 vs 50），在特定浏览器中也可能拦截点击事件。同时#endOverlay默认必须有 pointer-events:none，只在.show时改为pointer-events:auto。
- **每次编辑后验证HTML完整性和DOM顺序**：用 Select-String 检查关键元素（cover、endOverlay、canvas、video）的相对顺序是否正确。

## 停止动画 (End Animation)

## z-index 层级 (z-index Stacking)
- **放弃全屏深色覆盖层方案**：使用 ackground:rgba(0,0,0,.4) 半透明 + ackdrop-filter:blur(6px) 毛玻璃效果代替全屏实心深色背景。这样控件即使没有 z-index 也能透过覆盖层看到。
- **停止后控件保持 100% 透明度**：.stopped 的 opacity: .35 改为 opacity: 1，让控件在毛玻璃背景下完全可见。
- **不要隐藏 canvas**：停止时不要 ca.style.display='none'，让最后一帧画面作为毛玻璃的背景。

- **结束覆盖层（z-index:40）会盖住所有 z-index 低于 40 的元素**。即使控件没有被隐藏（display 可见），只要它们的容器 z-index < 40，就会被结束层的深色背景完全遮挡。
- **确保控件容器的 z-index 高于结束覆盖层**。#controls、#info、#btnStop 等容器元素的 z-index 必须 >= 50，不能只看它们内部子元素的 z-index。
- **.stopped 状态和 z-index 都要改**。停止后控件需要：①保持可见（不 display:none），②z-index 高于结束层，③视觉上变淡（opacity:.35）说明已停用。
- **"再玩一次"和"开始体验"都要清除 .stopped 状态**。停止时加上的类，在重新开始时必须主动 remove，否则 opacity:.35 会覆盖 .on 的 opacity:1。


- **停止摄像头时不要立即 kill 一切**。先设置 active=false 停止检测循环，保留最后帧画面作为背景，再显示结束覆盖层。延迟 1.5 秒后再清理摄像头和模型。
- **视频加 .dim 类变成半透明模糊背景**，比黑屏过渡更自然。CSS: filter:brightness(.25) saturate(.3) blur(4px)。
- **结束卡片内的元素依次交错入场**（stagger），用 CSS animation-delay 实现：图标→标题→描述→统计→按钮。
- **"再玩一次"按钮必须同时恢复 canvas 显示**（ca.style.display=''）、移除视频 dim 类。否则回到封面后 canvas 仍隐藏，再次点击"开始体验"可能不可用。
- **Python 写文件比 PowerShell 可靠**

## GSAP 动画 (GSAP Animation)
- **GSAP timeline 代替 CSS transition**：结束动画用 gsap.timeline({paused:true}) 创建，在 onComplete 中清理摄像头/模型。避免 timeline 播放期间出现竞态条件。
- **onComplete 再清理资源**
- **无限循环（repeat:-1）不能放在 timeline 主链上**。gsap.timeline 的 onComplete 只在所有 tween 都完成时触发，包含 
epeat:-1 的 tween 永远不会完成。应把无限动画放在 	l.add(fn) 中，在函数内用独立的 gsap.to(..., {repeat:-1}) 启动。
- **.show 类必须在动画过程中尽早添加**（而不是等 timeline 整体完成）。把 classList.add('show') 放在 	l.to() 的 onComplete 回调中，将其附加到具体某个 tween 上，确保 pointer-events: auto 在交互按钮出现之前就已生效。
：不要立即 kill 摄像头，等 GSAP 动画全部播完（约 1.5-2s）后再停流、关模型。
- **再玩一次先清 GSAP 再恢复**：gsap.set(element, {clearProps:'all'}) 清除所有 GSAP 注入的内联样式，否则元素卡在半路状态。另外显式 reset video filter 和控件 class。
- **JS 代码段用 
（LF），CSS/HTML 用 
（CRLF）**。在同一文件中混合行尾是常见的，脚本替换时必须保持目标区段正确的行尾。
- **验证所有 15 项**（GSAP CDN、timeline、关键时间线步骤、CSS 清理、endBtn 重置）确保动画完整工作。

- **停止后保持控件可见**。不要隐藏（display:none 或 classList.remove('on')）控件栏、信息栏、停止按钮。改为加 .stopped 类，用 opacity:.35 + pointer-events:none 使其视觉上静默但仍占据位置。按钮文字改为"✓ 已结束"并 disabled。
- **检查所有 stop 处理函数**。项目中可能有多个 stop 处理函数（stopCam() 和 _stopBtn click 回调），每个都需要改，不能只改一个。
- **Python脚本直接写入，不用PowerShell heredoc传多行**。PowerShell的@""@ heredoc会引入BOM导致Python报SyntaxError。用Set-Content -Encoding UTF8写临时文件代替。
。涉及多行文本替换、中文、正则时，写临时 .py 文件比在 PowerShell 中处理字符串拼接/转义稳定得多。

## 动画与停止 (Animation & Stop)

- **GSAP 行内样式会覆盖 CSS**。用 GSAP 的 tl.to() 设置 opacity 后，GSAP 会设置行内样式，覆盖 CSS 类的同名属性。如果需要 CSS 类控制最终状态，不要用 GSAP 修改该属性。
- **结束界面必须有 CSS 兜底**。结束卡片等元素的可见性不能完全依赖 GSAP 动画。在 CSS 中添加 .show 状态下的 opacity: 1 !important 兜底规则，确保即使 GSAP 动画失败，元素也可见。
- **GSAP 代码必须包 try/catch**。GSAP CDN 加载失败、版本不兼容等都会导致 JS 运行错误。关键路径上的 GSAP 调用需要 try/catch 保护。
- **停止后的控件应保持可见**。停止摄像头后，控制按钮（停止/开关）应该以非交互状态（pointer-events: none）可见，而不是消失。用户需要知道页面状态。
- **区分 CSS transition 和 GSAP animation**。同一个属性不要同时使用 CSS transition 和 GSAP animation，两者会冲突。GSAP 设置行内样式，CSS transition 需要类切换触发，顺序管理不好的话效果互相干扰。

## 停止崩溃处理 (Stop Crash Hardening)

- **停止按钮 handler 整体必须包 try/catch**。handler 内任何 DOM 操作都可能 throw（如 setD、endFrames.textContent），一旦抛出，后面的 GSAP timeline / CSS fallback 全部不执行，覆盖层永不显示。
- **添加 forceEndOverlay() 兜底函数**：用 el.style.cssText='opacity:1!important;...' 强制显示结束覆盖层，防止 GSAP 动画失败或 CSS 类不生效。在 catch 块和 setTimeout(2000) 中调用。
- **setD()、textContent 赋值必须加 null guard**：el && (el.textContent = val) 模式，不要在 handler 内暴露未保护的 DOM 访问。
- **CSS .show 类加固：加 display:flex!important + end-card 加 visibility:visible!important**。即使 GSAP 设置的行内样式或 CSS 过渡失败，覆盖层和内容依然强制可见。
- **GSAP timeline 添加 eventCallback('onComplete')**：清理摄像头/模型资源的代码应放在 onComplete 中，而非依赖 tl.add 内的 setTimeout（可能因顺序问题被跳过）。
- **GSAP play 同时启动 setTimeout(forceEndOverlay, 2000) 作为最终兜底**：确保即使 GSAP 全程静默失败，覆盖层也一定在 2s 内显示。

## 功能移除 (Feature Removal)

- **移除一项功能时，必须清理所有相关代码**：CSS 规则、HTML 元素、JS 变量声明、函数定义、所有调用点、注释。——残留的变量引用或函数定义会导致 JS 运行时错误（如 ReferenceError）。
- **移除前先全文搜索**：搜索功能相关的所有关键词（变量名、函数名、CSS class/id、HTML id），确保无一遗漏。
- **移除后验证功能性**：检查邻近代码未被误删，关键路径（停止、开始、结束覆盖层等）不受影响。
- **批量行删除要用 Python 脚本，精确匹配**：用 remove_indices = set() + 精确行号标记，比 brace_depth 流式删除更安全。

## 变量声明完整性与帧计数 (Variable Completeness)

- **所有被引用的变量必须有声明和初始化**。项目中 _frameCount 被 stopBtn handler 引用（endFrames.textContent=_frameCount||0），但从未声明。必须检查 var 声明和 ++ 增量的配对齐全。
- **结束画面显示的统计数据必须源自真实计数**。不要在 stop handler 写死 0 或依赖未定义的全局变量。_frameCount 必须在 render() 循环内逐帧递增，_startTime 必须在开始体验时赋值。

## Python 文本替换精度 (Python Text Replacement Precision)

- **替换前用 repr() 打印目标区段的原始字符串**。CRLF/LF 混用、多余空白行会导致字符串匹配失败。先用 Python 的 repr() 查看精确内容再写替换模式。
- **用 \r\n 转义序列匹配 Windows 文件**。PowerShell 的转义与 Python 不同，文件中的真实行尾是 \r\n（CRLF），不是 \n。
- **分步替换，每步验证上次替换成功再继续下一步**。避免因前一步失败导致后续所有步骤错位。
- **替换后立即验证**：用 keyword in text 检查所有目标关键词是否存在于修改后的文件中。

## Canvas X光医学渲染 (X-Ray Medical Rendering)

- **X光医学风格比霓虹科幻风格更受用户认可**。最初实现的是赛博朋克五色手指+扫描线+粒子流，用户评价"丑"。改为X光片风格（暗灰背景+青紫霓虹骨骼+边界框+置信度标签）后用户满意。**风格选择必须贴合应用场景（医学分析 vs 游戏娱乐），不能预设喜好。**
- **joint boundary box 是医学UI的核心视觉元素**。每个关节点用半透明矩形框包围，四角加 corner accent 短线，框上方显示骨骼名称（MCP/PIP/DIP/Distal Phalanx），框下方显示置信度分数（如"0.85"），模拟目标检测AI的标注界面。
- **骨骼绘制用双色分层（青+白）模拟X光片**。外层用青色半透明粗线+shadowBlur 发光，内层用白色细线作为骨骼高密度核心，模拟X光片的骨质密度差异。
- **掌骨（Metacarpal）和指骨（Phalanx）用不同颜色区分**。掌骨（手腕到MCP）用紫色，指骨（MCP到指尖）用青色，anatomical 层次更清晰。
- **每只手标注 LEFT/RIGHT** 文字标签在手的暗色背景区域上方，方便区分左右手。
- **暗色半透明背景垫底**。在每只手区域后方画 `rgba(10,10,20,.85)` 的圆角矩形，创造"医学暗室"的观感，使发光骨骼在摄像头画面上更醒目。
- **所有标签使用等宽字体**（SF Mono / monospace），符合医疗仪器界面的 typography 惯例。

## 大型代码替换 (Large Code Replacement)

- **替换超长代码块时，用临时 Python 脚本代替 edit_file**。edit_file 的 SEARCH 长度有限且对 CRLF/多余空白行极其敏感，超长块（>50行）匹配失败率高。
- **Python 替换步骤**：① `open('file','rb')` 读取字节 ② `.decode('utf-8')` 转文本 ③ `text.find(start_marker)` 定位边界 ④ 拼接新文本 `text[:start] + replacement + text[end:]` ⑤ `.encode('utf-8')` + `write` 写回。这种方案不受行尾和空白行差异影响。
- **替换后立即验证三个层面**：① JS 语法（提取 script 内容后 node --check）② 关键 DOM 元素存在 ③ 已删除功能的关键词确认已移除。
- **提取 HTML 中 script 标签内容做语法检查**：`re.search(r'<script>(.*?)</script>', text, re.DOTALL)` 提取后写入临时 .js 文件，再用 `node --check` 验证。
- **HTML5 的 `</body>` 和 `</html>` 是可选的**。浏览器会自动闭合，缺失不是 bug，验证时无需标记为错误。

## 功能验证 (Feature Verification)

- **删除前先验证功能是否存在**。用户要求删除某项功能时，必须先在代码中全文搜索该功能的所有相关关键词（CSS class/id、HTML id、JS 变量名、函数名、注释关键词）。如果搜索结果显示该功能从未实现（或已在前次编辑中删除），应告知用户实情，不执行不存在的删除操作。
- **空函数体是未实现的占位符**。函数体为空（仅有变量声明或注释）的空函数，表示该功能是预留但未实现的占位符。若用户要求删除对应功能，只需确认占位符的存在而非实际代码，清理时需避免把邻近已实现的代码一起误删。
- **区分"功能不存在"和"功能已存在"**。不要假设用户描述的功能一定存在于代码中。先搜索、再确认、最后操作。如果功能不存在，直接在回复中说明，无需修改代码。