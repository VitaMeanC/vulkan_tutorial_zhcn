# 简介

## 关于此教程

此教程将会教你使用基础的[Vulkan](https://www.khronos.org/vulkan/)图像及计算API。由[Khronos group](https://www.khronos.org/)（因OpenGL而闻名）开发的Vulkan是一套提供了对现代显卡更好的抽象的全新API。这套新接口允许你更好地描述你的应用程序想要做的事情，因此它拥有比现有的图形API，比如[OpenGL](https://zh.wikipedia.org/wiki/OpenGL)和[Direct3D](https://zh.wikipedia.org/wiki/Direct3D)，更好的性能以及更少的意外行为。Vulkan背后的思想比较接近[Direct3D 12](https://zh.wikipedia.org/wiki/Direct3D#Direct3D_12)和[Metal](https://zh.wikipedia.org/wiki/Metal_(API))的，但是Vulkan拥有跨平台的优点，也就是说，你可以同时为Windows、Linux和Android平台开发应用程序。

然而，拥有这些优点所要付出的代价是，你必须使用这些非常详细的API。图形API中的每一个细节都需要你在使用前从零开始设置好，包括帧缓冲创建前的初始化，或者像是缓冲或者贴图这类对象的内存管理。显卡驱动少了很多手把手帮你设置的动作，这意味着你需要在你的应用程序中做更多工作来保证行为正确。

丑话先说在前头，Vulkan并不适合每一个人。它针对的是那些钟情于高性能的计算机图形，并且愿意为其做贡献的程序员们。如果你对游戏开发更感兴趣而不是计算机图形学的话，你可能会愿意继续使用OpenGL或者Direct3D而不是抛弃它们换用Vulkan。另一个可供选择的选项是使用[虚幻4](https://zh.wikipedia.org/wiki/%E8%99%9A%E5%B9%BB%E5%BC%95%E6%93%8E#%E8%99%9A%E5%B9%BB%E5%BC%95%E6%93%8E4)或者[Unity](https://zh.wikipedia.org/wiki/Unity_(%E6%B8%B8%E6%88%8F%E5%BC%95%E6%93%8E))之类的游戏引擎，它们可以在使用Vulkan的同时暴露更高级的API给你。

来看看在接下来的教程中有什么事前准备需要做：
* 支持Vulkan的显卡及驱动([NVIDIA](https://developer.nvidia.com/vulkan-driver), [AMD](http://www.amd.com/en-us/innovations/software-technologies/technologies-gaming/vulkan), [Intel](https://software.intel.com/en-us/blogs/2016/03/14/new-intel-vulkan-beta-1540204404-graphics-driver-for-windows-78110-1540))
* C++使用经验（熟悉RAII和初始化列表）
* 支持C++11的编译器（Visual Studio 2013及以上版本，GCC 4.8及以上版本）
* 对3D计算机图形学已有一些经验

此教程假设你不懂OpenGL或Direct3D，不过你的确需要知道一点3D计算机图形学的基本概念，比如此教程不会解释透视投影背后的数学原理。你可以读读[这本电子书](https://paroj.github.io/gltut/)，它很好地介绍了计算机图形学的概念。这是一些其它的关于计算机图形学的很棒的资源：
* [Ray tracing in one weekend](https://github.com/petershirley/raytracinginoneweekend)（《一个周末就能看懂的光线追踪》）
* [Physically Based Rendering book](http://www.pbr-book.org/)（《基于物理的渲染》）
* Vulkan在真正的游戏引擎中的实践，以开源版本的[Quake](https://github.com/Novum/vkQuake)和[DOOM 3](https://github.com/DustinHLand/vkDOOM3)为例

如果你想的话，你可以用C代替C++。不过你得用一个其它的线性代数库，并且代码结构也得用你自己的。我们将会使用一些C++的特性，比如类和RAII，来组织代码逻辑并管理资源的生命周期。这也有为Rust开发者写的此教程的[另外一个版本](https://github.com/bwasty/vulkan-tutorial-rs)。

为了让那些使用其他语言的开发者们能够跟上我们的教程，也为了能够积攒一点使用基础API的经验，我们将会使用Vulkan的原生C API。不过如果你在使用C++，或许你会更喜欢这个新一点的[Vulkan-Hpp](https://github.com/KhronosGroup/Vulkan-Hpp)绑定，它对一些脏活累活做了抽象，并且有助于防止某些类型的错误的发生。

## 电子书
如果你更喜欢以电子书的形式阅读此教程，你可以在此下载EPUB或者PDF格式的版本（英文原文）：
* [EPUB](https://raw.githubusercontent.com/Overv/VulkanTutorial/master/ebook/Vulkan%20Tutorial.epub)
* [PDF](https://raw.githubusercontent.com/Overv/VulkanTutorial/master/ebook/Vulkan%20Tutorial.pdf)

## 此教程的结构

我们将会从一个介绍Vulkan如何工作的概述，以及为了在屏幕上画出第一个三角形需要做的所有工作开始。其目的在于，当你理解了每一个小步骤在整张图片中所扮演的基本角色之后，你会更好地理解它们的作用。下一步，我们会用[Vulkan SDK](https://lunarg.com/vulkan-sdk/)、做线性代数操作的[GLM](http://glm.g-truc.net/)库和创建窗口的[GLFW](http://www.glfw.org/)库建立一个开发环境。此教程将会展示如何在Windows+Visual Studio和Ubuntu Linux+GCC上设置这些库。

然后，我们将会实现所有必要的Vulkan程序组件，用来渲染你的第一个三角形。每一章都会大致遵循以下结构：
* 介绍一个新概念以及它的作用
* 调用所有相关的API来把它整合进你的程序里
* 它在帮助函数中的抽象部分

虽然此教程的每一章都与上一章相连，但是你也可以把这些章节看作是独立的，每一章讲解某个Vulkan的概念。这意味着，这个网站也可以作为有用的参考资料。所有Vulkan函数和类型都被链接到了它们的规范上，你可以点击它们来获取更多内容。Vulkan是一套很新的API，所以规范本身也有可能出现某些短板，我们鼓励你向[这个Khronos仓库](https://github.com/KhronosGroup/Vulkan-Docs)提交反馈。

正如之前所说过的，Vulkan API是一个有许多参数的、相当啰嗦的API，旨在能让你最大限度地控制显卡。这就导致了哪怕是基础操作，例如创建一个贴图，都要经过很多步骤，并且每次都要重复这些步骤。因此我们会在教程的每一步创建我们自己的帮助函数合集。

每一章都会以一个到该知识点为止的完整程序源代码的链接作结。如果你对代码的结构有疑问，或者你在debug时需要一个参考对象，你都可以来参考这个源代码。所有代码都经过了多家供应商的不同显卡来测试其正确性。在每一章的底部都有一个评论区，你可以在这问关于本章的具体问题。提问时请说明你的平台、驱动版本、源代码、你期望的行为和实际发生的行为来让我们帮助你。

这个教程旨在结成一个社区。Vulkan现在仍然是非常新的一套API，它的最佳实践现在还没有形成。如果你对此教程或此网站有任何反馈，不要迟疑，提交一个issue或者一个pull request到[这个GitHub仓库](https://github.com/Overv/VulkanTutorial)。你可以watch这个仓库以获取此教程的更新提示。

当你经历了在屏幕上渲染第一个三角形的仪式之后，我们会扩展这个程序，将线性变换、贴图和3D模型加入进去。

如果你在这之前玩过图形API，你就会知道在屏幕上渲染出几何图形之前有许多要做的步骤。这种初始化步骤在Vulkan中有很多，不过你会发现每个具体步骤都很容易理解，并且也不会感到冗长。还有一件很重要的事情，那就是要记住，当你看腻了三角形之后，绘制有贴图的3D模型不需要做更多的额外工作，并且向前迈进的每一步都会让你感到无比自豪。

如果你在跟着教程操作的时候遇到了问题，首先请查看“常见问题”部分是否已经列出并解决了你的问题。如果没有，请放轻松并且在最相关的那一章的评论区提问。

准备好投身于高性能图形API的未来了吗？让我们开始吧！