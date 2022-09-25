# 在浏览器中运行 GDB

使用在浏览器中运行的 Web 程序集的 x86 虚拟机中的 Linux 中的 GDB。疯狂的？

GDB 是一个非常强大的调试工具。最近，我通过实现 gdbserver 协议为[Wokwi Arduino Simulator](https://wokwi.com/)添加了 GDB 支持。

然而，在 Wokwi 中使用 GDB 并不容易：您必须在计算机上下载并安装 avr-gdb 的副本，然后还下载并设置[一个特殊的代理](https://github.com/wokwi/wokwi-gdbserver)，将在浏览器中运行的模拟器与 GDB 连接起来。

简而言之 - 用户体验远非理想。Wokwi 模拟器在浏览器中运行，因此只有尝试让 GDB 也在浏览器中运行才有意义，让我们的用户可以神奇地点击链接并直接进入 GDB 体验。

在这篇博文中，我将与您分享我尝试过的三种不同方法以及令人惊讶的解决方案：使用在浏览器中的虚拟机 (VM) 中运行的微型 Linux 系统运行 GDB。让我们开始吧！

# 第一次尝试：WebAssembly

我的第一个想法是：GDB 是用 C/C++ 编写的，所以我可以将它编译成 WebAssembly 二进制文件，它可以在任何现代浏览器中运行。这似乎是最直接的解决方案，我也希望它能够提供不错的性能。

我用 Emscripten（一个 WebAssembly 编译器工具链）设置了一个 Docker 容器，下载了[最新的 GDB 源代码存档](https://ftp.gnu.org/gnu/gdb/gdb-10.1.tar.xz)，然后……第二天晚上，我和我的朋友 Benny Meisels 进行了很长时间的通话。

我们一一解决了所有的编译器问题，[7个补丁文件](https://github.com/wokwi/wasm-avr-gdb/tree/main/patches)之后，我们就编译好了！

然而，事实证明，让 GDB 编译只是冰山一角：

首先，我们得到的二进制文件很大——大约 90MB！
其次，它需要启用实验性的 WebAssembly 线程支持。
它将无法在 Chrome 中加载，并且在 Node.js 中运行时，它会立即退出：

<iframe id="twitter-widget-0" scrolling="no" frameborder="0" allowtransparency="true" allowfullscreen="true" class="" title="推特推文" src="https://platform.twitter.com/embed/Tweet.html?dnt=false&amp;embedId=twitter-widget-0&amp;features=eyJ0ZndfdGltZWxpbmVfbGlzdCI6eyJidWNrZXQiOlsibGlua3RyLmVlIiwidHIuZWUiLCJ0ZXJyYS5jb20uYnIiLCJ3d3cubGlua3RyLmVlIiwid3d3LnRyLmVlIiwid3d3LnRlcnJhLmNvbS5iciJdLCJ2ZXJzaW9uIjpudWxsfSwidGZ3X2hvcml6b25fdGltZWxpbmVfMTIwMzQiOnsiYnVja2V0IjoidHJlYXRtZW50IiwidmVyc2lvbiI6bnVsbH0sInRmd190d2VldF9lZGl0X2JhY2tlbmQiOnsiYnVja2V0Ijoib24iLCJ2ZXJzaW9uIjpudWxsfSwidGZ3X3JlZnNyY19zZXNzaW9uIjp7ImJ1Y2tldCI6Im9uIiwidmVyc2lvbiI6bnVsbH0sInRmd19jaGluX3BpbGxzXzE0NzQxIjp7ImJ1Y2tldCI6ImNvbG9yX2ljb25zIiwidmVyc2lvbiI6bnVsbH0sInRmd190d2VldF9yZXN1bHRfbWlncmF0aW9uXzEzOTc5Ijp7ImJ1Y2tldCI6InR3ZWV0X3Jlc3VsdCIsInZlcnNpb24iOm51bGx9LCJ0Zndfc2Vuc2l0aXZlX21lZGlhX2ludGVyc3RpdGlhbF8xMzk2MyI6eyJidWNrZXQiOiJpbnRlcnN0aXRpYWwiLCJ2ZXJzaW9uIjpudWxsfSwidGZ3X2V4cGVyaW1lbnRzX2Nvb2tpZV9leHBpcmF0aW9uIjp7ImJ1Y2tldCI6MTIwOTYwMCwidmVyc2lvbiI6bnVsbH0sInRmd19kdXBsaWNhdGVfc2NyaWJlc190b19zZXR0aW5ncyI6eyJidWNrZXQiOiJvbiIsInZlcnNpb24iOm51bGx9LCJ0ZndfdHdlZXRfZWRpdF9mcm9udGVuZCI6eyJidWNrZXQiOiJvZmYiLCJ2ZXJzaW9uIjpudWxsfX0%3D&amp;frame=false&amp;hideCard=false&amp;hideThread=false&amp;id=1352562450269409284&amp;lang=en&amp;origin=https%3A%2F%2Fblog.wokwi.com%2Frunning-gdb-in-the-browser%2F&amp;sessionId=a1046360a541a7bac38cf3771e0d97d1960ecae0&amp;siteScreenName=WokwiMakes&amp;theme=light&amp;widgetsVersion=1bfeb5c3714e8%3A1661975971032&amp;width=550px" data-tweet-id="1352562450269409284"></iframe>

虽然我对最初的成功感到高兴，但如果我想让它在浏览器中工作，似乎还有很长的路要走。

但请随时证明我错了。我在 GitHub 上分享了这个项目，所以你可以继续破解它：[https ://github.com/wokwi/wasm-avr-gdb/](https://github.com/wokwi/wasm-avr-gdb/)

# 第二次尝试：云 GDB

然后我想：如果我不能在浏览器中运行 GDB，第二个最好的办法是在云中的容器内与代理服务器一起运行它，并创建一个小型前端应用程序，将各个部分粘合在一起并让用户与云 GDB 实例交互。

这种方法存在一些挑战：

1. 我必须为每个用户配置一个容器，因为每个用户都需要一个新的 GDB 副本。只要用户连接，这个容器就必须存在，因此容器的数量会快速增长并消耗大量 RAM。
2. GDB 可以执行 shell 命令，所以我必须考虑保护这些容器并确保它们完全隔离。

幸运的是，我不必自己处理这些问题——我找到了一个已经可以满足我需求的服务。[Gitpod](https://www.gitpod.io/)是一项服务，可让您在云中运行开发环境，并且该过程完全可以编写脚本。

从用户的角度来看，使用 Gitpod 非常简单：只需将浏览器导航到 gitpod.io/#repo-url-here，等待几秒钟，您的浏览器中就会拥有一个功能齐全的开发环境。

我设置了一个安装 Node.js 代理和 avr-gdb 的[Gitpod 配置](https://github.com/wokwi/wokwi-gdbserver/blob/gidpod/.gitpod.yml)，然后打开一个最小网页，其中包含如何将其与 Wokwi 连接的说明：

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/image-1.png)Node.js 代理说明页面，在 Gitpod 上运行有趣的部分——我没有为 GDB 编写自己的前端，而是使用了[GoTTY](https://github.com/yudai/gotty)，这是一个将 CLI 工具（例如 GDB）转换为 Web 应用程序的程序。它通过设置一个 Web 套接字来实现其魔力，该套接字将 CLI 程序的输出传输到浏览器，并将浏览器的输入传输到 CLI 程序。

与我之前的尝试不同，我在几个小时内让 Gitpod 设置工作。但是，它有一些主要限制：

1. Gitpod 要求您先使用 GitHub 帐户登录，然后才能使用它。
2. 工作区的启动时间为 10-30 秒。用户每次启动新的 GDB 会话时都必须等待。
3. 如果 Gitpod 将来发生更改，这可能会中断。如果它可以破裂，它就会破裂。

所以在我看来，这仍然不是一个足够好的解决方案，但至少它是工作和可用的。

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/image-6.png)GitPod 中的 GDB 使用 GoTTY如果您想亲自体验，请[点击此链接](https://gitpod.io/#https://github.com/wokwi/wokwi-gdbserver/tree/gidpod)。我可以花更多时间完善体验（这样您就不必手动将 WebSocket 链接复制到 Wokwi 中，也不必将调试符号加载到 GDB），但我决定继续探索另一种方法：

# 获胜者：浏览器中的 Linux VM！

将 GDB 编译为 Web Assembly 被证明是一项具有挑战性的任务，但如果我能以某种方式在浏览器中运行 Linux GDB 二进制文件会怎样？

这将允许我使用任何 GDB 版本，而无需解决编译问题或使代码适应浏览器环境。只需获取现有的二进制文件并运行它而无需任何修改！

然后我想起[了 JSLinux](https://bellard.org/jslinux/)。顾名思义，它是在浏览器中运行的 Linux 版本。大约 10 年前，当我第一次看到它时，我完全被吓到了。

我开始使用[Alpine X86](https://bellard.org/jslinux/vm.html?url=alpine-x86.cfg&mem=192) VM，性能似乎可以接受。GDB 通常不做繁重的计算，所以我很乐观，我可以从这个设置中得到一些有用的东西。

唯一的问题是：JSLinux 没有真正的文档，没有关于如何构建虚拟机的信息，而且我找到的唯一[源代码](https://github.com/levskaya/jslinux-deobfuscated)是 9 年前的。所以我去寻找替代品。

## 然后我找到了：v86

v86 是一个 32 位 x86 模拟器，用 WebAssembly 编写。事实上，它甚至将 x86 机器码翻译成 WebAssembly，从而使其运行得更快。

它是[开源](https://github.com/copy/v86)的、有据可查的，并且有几个[在线演示 VM](https://copy.sh/v86/)可供您使用。它还支持快照，这意味着您可以在机器加载后保存机器的状态，而不必每次都等待 Linux 启动（参见[Arch Linux 演示](https://copy.sh/v86/?profile=archlinux)）。

我玩了一些演示，甚至上传了[GDB 的静态构建](https://github.com/hugsy/gdb-static)，效果很好。但是，我仍然需要弄清楚如何使用支持 AVR 的更新版本的 GDB 构建我自己的 Linux 映像。

v86 自述文件将我指向[browservm](https://github.com/humphd/browser-vm)，这是一个 docker 容器，可自动为 v86 创建 Linux 映像。它使用 Buildroot，一种可以为嵌入式系统生成紧凑 Linux 映像的工具。

我不熟悉 Buildroot，但幸运的是，[bro​​wservm 文档](https://github.com/humphd/browser-vm#running-via-docker)证明非常有用，并解释了如何自定义构建。事实证明 buildroot 带有一个内置的 GDB 包，您可以通过他们的`menuconfig`系统启用：

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/image-2.png)添加 GDB 就像选中一个框一样简单！那天晚上，我设法获得[了在浏览器中运行的 avr-gdb 的第一个工作版本](https://github.com/wokwi/web-avr-gdb/tree/ed798c0977b9a32a71af50a22d36ddfdb25d7c44)！🥳

为了让交易更甜蜜，整个 Linux + GDB 映像只有 6.25MB，即使加上 v86 的大小（约 2.3MB），这仍然是一个合理的下载。提醒您，WebAssembly 二进制文件是 90MB！

但工作还没有完成：

1. GDB版本是7.11.1，快5年了
2. 它无法与 Wokwi 模拟器通信
3. 虚拟机的启动时间为 15-30 秒

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/Gdb-avr-browser-rc.gif)初步成功！旧 GDB，启动缓慢## 捆绑松散的末端

第二天，我将 Buildroot 升级到最新版本，找出构建最新 GDB (10.1) 的配置，找到与 v86 正常工作的内核版本，并[修补](https://github.com/wokwi/browser-vm-gdb/blob/master/gdb-avr.patch)Buildroot以构建具有 AVR 目标（而不是 x86）的 GDB。

然后我让 GDB 和 Wokwi 对话。浏览器 VM 通过模拟串行端口 ( ) 与用户交互，该串行端口使用出色的Term.js库`/dev/ttyS0`连接到虚拟终端。[](https://xtermjs.org/)

我添加了第二个串行端口和一些[JavaScript 胶水，通过](https://github.com/wokwi/web-avr-gdb/blob/9ef057a9b35c914ca9ac806af5ba71f6cfb1eaf5/src/worker.js#L237)[MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)端口将 gdbserver 消息传输到 Wokwi 。这就是您可以要求 GDB 通过串行端口进行调试的方式：`gdb -ex "target remote /dev/ttyS1"`.

当您拥有调试符号和正在调试的程序的源代码时，GDB 会更加有用。但是如何让它们进入虚拟机呢？

幸运的是，v86 支持[9P2000](https://en.wikipedia.org/wiki/9P_(protocol))，这是一个简单的远程文件系统协议，允许 VM 读取 JS 代码提供的文件。Wokwi 通过同一个 MessageChannel 发送带有符号和程序源的二进制 ELF 文件，[另一段胶水代码](https://github.com/wokwi/web-avr-gdb/blob/9ef057a9b35c914ca9ac806af5ba71f6cfb1eaf5/src/worker.js#L131-L139)将它们上传到 VM。

此时，仍然存在一个主要的可用性问题：VM 需要大约 15-30 秒才能启动！网络用户被宠坏了。他们希望他们的网络应用程序在几秒钟内加载，而不是半分钟。

v86 可以从快照启动 VM，跳过整个启动过程。它有一个缺点：如果您下载 Linux 映像并启动它，则需要下载 ~20MB 而不是仅 ~6.25MB。因此，您可以节省启动时间，但会花更多时间下载快照。

我最终结合了这两种方法：当您第一次启动 Web GDB 时，它会启动 Linux，然后拍摄快照，将其存储到浏览器缓存中。然后，下一次，快照从缓存中加载，虚拟机几乎立即启动。

# 万岁！浏览器中的 GDB

我们做到了。GDB在浏览器中加载运行，性能足够好，甚至还有TUI（Text User Interface）[支持](https://github.com/wokwi/web-avr-gdb/commit/3da7f18f9409c07a2d296b04df3cff9cd5b72958)！

现在轮到你检查了：打开 Wokwi 上的任何项目（例如这个[Simon 游戏](http://localhost:3002/arduino/libraries/demo/simon-game)），点击代码编辑器，然后按 F1。在打开的提示中，输入“GDB”：

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/image-8.png)选择“调试构建”选项（发布构建更难调试，但如果您的程序使用 FastLED 库，它会很有用）。Web GDB 将在新的浏览器选项卡中加载（您必须有点耐心），您应该会看到熟悉的 GDB 提示：

```
0x00000000 in __vectors ()
(gdb)
```

此时，您可以编写`continue`程序来启动程序，或者更好 - 查看[Arduino/AVR GDB 备忘](https://blog.wokwi.com/gdb-avr-arduino-cheatsheet/)单，了解 GDB 可以为您做的所有事情！

![](https://cdn.getmidnight.com/84f7b02a8128f5f5775611244c24b941/2021/02/image-7.png)Web GDB + TUI 调试[Simon](https://wokwi.com/arduino/libraries/demo/simon-game)### 资源

* [Web AVR GDB 存储库](https://github.com/wokwi/web-avr-gdb)
* [avr-gdb Buildroot 配置（浏览器虚拟机）存储库](https://github.com/wokwi/browser-vm-gdb)
* v86：[源代码](https://github.com/copy/v86)，[在线演示](https://copy.sh/v86/)
* [Wokwi Arduino 模拟器](https://wokwi.com/)
* [Arduino/AVR GDB 备忘单](https://blog.wokwi.com/gdb-avr-arduino-cheatsheet/)

