---
title: 深入web浏览器内部
date: 2025-03-11 22:38:48
tags:
- 前端
- 浏览器
---

# 第一部分
## 浏览器架构（Chrome）

浏览器顶部是 浏览器进程，负责与应用程序不同部分的其他进程进行协调。 对于渲染进程，将创建多个进程并将其分配给每个选项卡。

| 进程 | 控制内容 |
| ----------- | ----------- | 
| 浏览器进程 | 控制应用程序的“chrome”部分，包括地址栏、书签、返回和 前进按钮。还包括了不可见部分，例如网络请求和文件访问 |
| 渲染进程 | 控制选项卡网站显示的内容内容 |
| 插件进程 | 控制网站使用的插件，比如Flash |
| GPU进程 | 与其他进程隔离处理 GPU 任务。它被分成不同的进程，因为 GPU 处理来自多个应用程序的请求并在同一中绘制它们。 |
![浏览器UI不同部分的不同进程](chrome-processes.png)

## 多进程架构优势
如果一个选项卡变得无响应，则可以关闭无响应的选项卡并继续作，同时保持其他选项卡处于活动状态。如果所有选项卡都在一个进程上运行，则当一个选项卡无响应时，所有选项卡都无响应。这很可悲。

## 节省更多内容
当 Chrome 在强大的硬件上运行时，它可能会将每个服务拆分为不同的进程，从而提高稳定性，但如果它在资源受限的设备上，Chrome 会将服务整合到一个进程中，从而节省内存占用。在此更改之前，Android 等平台上已经使用了类似的合并进程以减少内存使用的方法。


# 第二部分
## 导航会发生什么
简单用例：在浏览器中键入 URL，然后浏览器从 Internet 获取数据并显示一个页面

一切从**浏览器进程**开始，选项卡之外的内容都有**浏览器进程**处理。**浏览器进程**具有线程，比如绘制浏览器按钮和输入字段的**UI线程**、处理网络堆栈以从 Internet 接收数据的****网络线程****、控制文件访问的**存储线程**等等。当从地址栏输入URL时，输入有**浏览器进程**的**UI线程**处理

## 简单的导航
1. 处理输入
当从地址栏输入内容，**UI线程**会处理这是“搜索内容还是站点URL”？。**UI线程**需要解析决定是发送到搜索引擎还是发送到请求的站点
2. 开始导航
当用户按Enter键，UI线程将启动网络调用以获取站点内容。**网络线程**会通过适当的协议，例如 DNS 查找和为请求建立 TLS 连接。
此时，**网络线程**可能会收到类似 HTTP 301 的服务器重定向标头。在这种情况下，**网络线程**会与服务器请求重定向的 UI 线程通信。然后，将启动另一个 URL 请求。
3. 读取响应
一旦正文响应开始进入，MIME类型检索会在这里完成。
如果响应的是HTML文件，则下一步会把数据传递给渲染进程。
同时，这里也是 SafeBrowsing 和 Cross Origin Read Block进程检查的地方。
SafeBrowsing：检查访问的展示是否在网络黑名单上
Cross Origin Read Block：跨源读取阻止，一种由浏览器实现的安全机制，旨在阻止恶意网站从其他源加载某些类型的跨域资源，从而减少跨站信息泄露的风险。

4. 查找**渲染器进程**
当**UI线程**想**网络线程**发送URL 请求时，它已经知道它们正在导航到哪个站点。**UI线程**尝试主动查找或启动与网络请求并行的**渲染器进程**。这样，如果一切按预期进行，则当**网络线程**收到数据时，**渲染器进程**已经处于待机状态。如果导航跨站点重定向，则可能不会使用此备用进程，在这种情况下，可能需要其他进程。

5. 提交导航
数据和**渲染器进程**已准备就绪，IPC 将从**浏览器进程**发送到**渲染器进程**以提交导航。它还传递数据流，以便**渲染器进程**可以继续接收 HTML 数据。一旦**浏览器进程**听到提交已在**渲染器进程**中发生的确认，导航即完成，文档加载阶段开始。
此时，地址栏会更新，安全指示器和站点设置 UI 会反映新页面的站点信息。选项卡的会话历史记录将更新，因此后退/前进按钮将逐步浏览刚刚导航到的站点。为了在关闭选项卡或窗口时便于恢复选项卡/会话，会话历史记录存储在磁盘上。

6. 额外步骤
提交导航后，**渲染器进程将**继续加载资源并渲染页面。**渲染器进程**“完成”渲染后，它会通过IPC发送回**浏览器进程**

## 导航到其他站点
如果用户将不同URL放在地址栏，**浏览器进程**会执行同样的步骤，但在此之前会先询问**渲染进程**是否关心`beforeunload`事件
如果导航是从**渲染器进程**启动的（如用户单击链接或客户端 JavaScript 已运行 window.location = "https://newsite.com" ），则**渲染器进程**首先检查 `beforeunload` 处理程序。然后，它将经历与**浏览器进程**启动的导航相同的过程。唯一的区别是导航请求是从**渲染器进程**启动到**浏览器进程**的。

![导航到别的页面](new-navigation-unload.png)

## Service Worker 用例
在注册Service Worker时，Service Worker 的范围将保留为引用。当导航发生时，**网络线程**会根据已注册的 Service Worker 范围检查域，如果为该 URL 注册了 Service Worker，则 **UI线程**会查找**渲染器进程**以执行 Service Worker 代码
![serviceworker作用域查询](service-worker-scope-look.png)

浏览器进程中的 UI 线程启动渲染器进程以处理 Service Worker;然后，渲染器进程中的工作线程从网络请求数据
![serviceworker导航](serviceworker-navigation.png)

## Navigation Preload 导航预加载
Navigation Preload是一种通过Service Worker启动时并行获取网络资源的机制

# 第三部分
**渲染器进程**的内部工作原理
## 渲染器处理web内容
**渲染器进程**负责选项卡内发生的所有事情。在**渲染器进程**中，主线程处理您发送给用户的大部分代码。有时，如果您使用 Web Worker 或 Service Worker，则部分 JavaScript 由 worker 线程处理。**合成器线程**和**光栅线程**也在**渲染器进程**内运行，以高效、流畅地渲染页面。

## Parsing
当**渲染器进程**收到导航的提交消息并开始接收 HTML 数据时，主线程开始解析文本字符串 （HTML） 并将其转换为 DOM。

## 子资源加载
主线程可以在解析以构建 DOM 时找到它们时逐个请求它们，但为了加快速度，“preload scanner” 是并发运行的。如果 HTML 文档中存在 <img> 或 <link> 之类的内容，则预加载扫描程序会查看 HTML 解析器生成的令牌，并将请求发送到浏览器进程中的网络线程。

## JavaScript会阻止Parsing
当 HTML 解析器找到 `<script>` 标记时，它会暂停 HTML 文档的解析，并且必须加载、解析和执行 JavaScript 代码

## 如何提示浏览器如何加载资源
1. 像`<script>`添加`defer`、`async`属性
2. `<script>`添加`type='module'`
3. `<link rel=“preload”>`是一种通知浏览器当前导航肯定需要该资源并且您希望尽快下载的方法

## 样式计算
主线程会解析CSS并构建CSS tree

## 布局
主线程遍历 DOM 和计算样式，并创建布局树。布局树节点包含了 x、y 坐标和边界框大小等信息。

## 绘制
主线程遍历布局树以创建 paint 记录。Paint record 为 绘制过程的注释，如 “Background first， then text， then rectangle”。
![主线程遍历布局树并生成绘制记录](paint-records.png)

## 更新渲染管道的成本很高
从 `构建布局树` 到 `布局` 到 `绘制`，每一步都依赖之前的结果

## 合成
合成是一种将页面的各个部分分成图层、分别栅格化它们，并在称为**合成器线程**的单独线程中合成为页面的技术。如果发生滚动，由于图层已经栅格化，因此只需合成一个新帧即可。可以通过移动图层和合成新帧以相同的方式实现动画。

栅栏化会将图层转换成像素点，它们就变成了预备好的“图片”。
1. 滚动：当用户滚动网页时，实际上只是在改变“观看窗口”的位置，也就是在“移动”这些已经栅格化好的图层。  合成线程只需要 重新组合 这些已经栅格化好的图层，生成新的画面帧，而 不需要重新进行耗时的“栅格化”。  这就大大加快了滚动的速度，让滚动变得流畅。
2. 动画：动画也是类似的原理。 动画通常是通过 移动、旋转、缩放  这些图层来实现的。  合成线程只需要根据动画效果， 调整图层的位置、大小等属性，然后重新合成新的画面帧。  同样，因为图层已经预先栅格化，所以动画的渲染也会非常高效流畅。

想象：栅栏化就是制作积木的过程，合成就是一个移动积木搭积木的过程。制作积木比较耗时，移动积木搭积木很快

## 主线程的栅栏和合成
创建层树并确定绘制顺序后，主线程会将该信息提交到**合成器线程**。然后，**合成器线程**栅格化每个图层。图层可能很大，就像页面的整个长度一样，因此**合成器线程**会将它们划分为多个平铺，并将每个平铺发送到栅格线程。栅格线程栅格化每个切片并将其存储在 GPU 内存中。

**合成器线程**可以首先对视区内的进行栅栏化。
栅格化平铺后，**合成器线程**会收集称为 Draw quads，以创建 Compositor frame。

| Draw quads | 包含图块在内存中的位置以及在页面中绘制图块的位置（同时考虑页面合成）等信息。 |
| Compositor frame | A collection of draw quads that represents a frame of a page. |

**渲染进程**通过IPC将 Compositor frame发送给**浏览器进程**， 同时**浏览器进程**的**UI线程**会添加另外的 Compositor进行扩展。之后这些 Compositor frame会发送到GPU，以在屏幕上显示它。如果出现滚动，则**合成器线程**将创建另一个要发送到 GPU 的合成器帧。


> 总的流程： parsing -> 构建dom tree、css tree 合成 render tree -> 布局 -> 绘制（构建paint记录） -> 栅栏化每个图层（转成像素） -> 合成每个栅栏化后的图层为 Compositor frame -> 发送回浏览器线程进行 frame 扩展 -> 发送到GPU进行绘制


