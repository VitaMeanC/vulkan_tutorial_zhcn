# 概述

这一章开始于对Vulkan以及它所解决的问题的介绍。然后我们将会看看为了画出第一个三角形所需要的步骤。这可以让你纵观全局并且理清后续每一章在整个过程中的位置。我们将会以展示Vulkan API的结构以及它们的一般使用模式作结。

## Vulkan的起源

和之前的那些图形API一样，Vulkan也被设计成了跨GPU平台抽象的。之前的那些图形API的问题在于，那个时代的API都被设计成了与显卡紧密相关的，并且被限制在了固定管线上。程序员们只能用什么“标准格式的顶点数据”，并且在显卡厂商的怜悯之下进行光照和着色操作。

随着显卡架构的不断发展，他们开始提供越来越多的可编程功能。这些新功能都是以某种方式利用已有的API被集成进去的。这就造成了不理想的抽象，以及在显卡将程序员的意图映射到现代图形架构时产生的许多猜测。这就是驱动要经常更新来为游戏提供更好的显示性能的原因，当然有的时候也是为了自己的利益。出于这种复杂性，应用开发者们也得处理各种供应商的显卡之间的不一致性，比如[着色器](https://zh.wikipedia.org/wiki/%E7%9D%80%E8%89%B2%E5%99%A8)的格式。除了这些新功能之外，在过去的十年中也能看到许多拥有强劲显卡的移动设备。这些移动设备的GPU根据功耗和空间需求的不同有着不同的架构。一个有代表性的例子就是[基于图块渲染](https://zh.wikipedia.org/wiki/%E5%9F%BA%E4%BA%8E%E5%9B%BE%E5%9D%97%E6%B8%B2%E6%9F%93)，通过给予程序员更大的控制权来获得更好的性能。之前的图形API的另一个问题在于不支持多线程，这往往是造成CPU端性能瓶颈的原因。

Vulkan通过根据现代图形架构从头重新设计的方式解决了这个问题。它通过让程序员使用更多的API来明确地声明自己意图的方式降低了驱动的开销，并且支持多线程，使得命令可以被并行地创建和提交。它通过将着色器程序通过单独的编译器编译成字节码的方式减少了编译着色器时的不一致性。最后，它承认现代显卡的计算能力，并且将计算和图形功能合并到了同一个API中。

## 画一个三角形需要做什么

我们现在来大致看看在一个行为良好的Vulkan程序中渲染一个三角形所需要的所有步骤。此处介绍的所有概念在接下来的章节中会被详细介绍。此处只是为了让你对每个具体组件之间的关系有一个整体上的了解。

### 第一步 实例和选择物理设备

一个Vulkan应用程序以通过一个[`VkInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstance.html)（实例）设置Vulkan API开始。创建一个实例则需要你描述你的应用程序并且设置你想使用的API扩展。创建实例之后，你可以查询支持Vulkan的硬件设备并且选择一个或多个[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)（物理设备）来使用。你可以查询设备的属性（比如显存大小）和能力来选择一个你想要的设备，比如你想使用独立显卡。

### 第二步 逻辑设备和队列家族

在选择了要使用的正确的硬件设备之后，你需要创建一个[`VkDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDevice.html)（逻辑设备），你可以在其中更具体地描述你想使用哪些[`VkPhysicalDeviceFeatures`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDeviceFeatures.html)（物理设备特性），比如多视口渲染以及64位浮点数。同时，你也需要指明你想使用的队列家族（queue families）。绝大多数Vulkan操作，比如绘图命令和内存操作，都是通过提交到[`VkQueue`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkQueue.html)（队列）来异步执行的。队列从队列家族中分配，每一个队列家族支持一组特定的操作。例如，可能存在不同的队列家族进行图形、计算和内存传递操作。队列家族的可用性也可以成为在选择物理显卡时的一个影响因素。Vulkan有可能会支持一些完全不具备任何图形功能的设备，然而如今所有支持Vulkan操作的显卡都可以完成我们感兴趣的所有队列操作。

### 第三步 窗口、表面和交换链

除非你只想离屏渲染，你将会需要一个窗口来显示渲染过的图像。窗口可以使用原生API或者像是[GLFW](http://www.glfw.org/)以及[SDL](https://www.libsdl.org/)之类的图形库来创建。在此教程中，我们选用GLFW，这个问题在下一章会详细讲解。

我们还需要另外两个组件来把图像渲染到窗口上：一个表面（`VkSurfaceKHR`, surface）和一个交换链（`VkSwapchainKHR`, swap chain）。注意一下，这个`KHR`后缀说明这些对象是Vulkan扩展（extension）的一部分。Vulkan API本身是完全平台无关的，因此我们必须使用标准化的WSI（Window System Interface，窗口系统接口）扩展来与窗口管理器进行交互。表面是一个跨平台的、针对被渲染的窗口的一个抽象，它通常需要传入一个原生窗口的句柄，比如Windows上的`HWND`进行实例化。幸运的是，GLFW库自带了一个函数来帮我们处理不同平台上的具体差异。

交换链是渲染目标的集合。它最基本的作用就是确保现在正在渲染的图像与现在显示在屏幕上的图像不是同一个。确保只有渲染完成的图像才会被显示这一点十分重要。每当我们想要绘制一个帧的时候，我们必须向交换链请求一个图像来进行渲染。当我们绘制完这一帧之后，再把这个图像返回到交换链中以便在某个时间点显示。渲染对象的数量以及将渲染好的图像显示到屏幕上的条件由显示模式（present mode）决定。常见的渲染模式有双缓冲（垂直同步）和三缓冲。在交换链那一章我们再详细讨论这些问题。

### 第四步 图像视图和帧缓冲

为了在从交换链中请求到的图像上进行绘制，我们需要用[`VkImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageView.html)（图像视图）和[`VkFramebuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkFramebuffer.html)（帧缓冲）把它包裹起来。一个图像视图引用一个图像中要被使用的特定部分，一个帧缓冲则引用一些图像视图并把它们当作颜色、深度和模板目标使用。因为在一个交换链中可能有多个不同的图像，所以我们提前为每一个图像创建一个图像视图和一个帧缓冲，并且在绘制时选择正确的那个。

### 第五步 渲染过程

Vulkan中的渲染过程（render passes）描述了在渲染操作时要使用的图像类型、图像的使用方式以及处理图像的内容的方式。在我们最初的这个绘制三角形的应用中，我们将会告诉Vulkan，我们将会使用一个图像作为颜色目标，并且我们想要在绘制操作中用纯色填充它。然而一个渲染过程只描述图像的类型，[`VkFramebuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkFramebuffer.html)（帧缓冲）才会把具体的图像绑定到这些选项上。

### 第六步 图形渲染管线

Vulkan中的图形渲染管线由[`VkPipeline`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipeline.html)（管线）来建立。它描述了显卡的一些可配置部分（不可编程部分），比如视口大小以及深度缓冲操作等，而可编程部分则使用[`VkShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModule.html)（着色器模块）对象来描述。[`VkShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModule.html)（着色器模块）对象使用着色器的字节码来创建。驱动还需要知道管线中的哪些渲染对象会被使用，而这些渲染对象就是我们在渲染过程中通过引用指定的图像。

Vulkan与现有的其它API之间最明显的区别就是，图形渲染管线的几乎所有配置项都需要在创建管线之前设置好。这意味着如果你想切换到另一个着色器或者稍微改变一下顶点数据的布局，你都需要重新创建整个图形渲染管线。这意味着，你或许需要针对你在渲染操作时的具体情况提前创建许多[`VkPipeline`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipeline.html)（管线）对象。只有很少的一些基础配置项可以动态更改，比如视口大小和清屏颜色等。你还必须明确地描述管线中的所有配置项，比如说没有默认的颜色混合选项。

好消息是，就像提前编译（AOT）与即时编译（JIT）那样，驱动有更多的优化机会，并且运行时性能将会更加具有可预测性，因为像是切换不同的图形渲染管线那样的大量的状态更改是非常明确的。

### 第七步 命令池和命令缓冲

正如刚才所说，在Vulkan中，许多我们想要执行的操作，比如绘制操作，都需要被提交到一个队列中去。这些操作在提交之前需要先被记录到一个[`VkCommandBuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandBuffer.html)（命令缓冲）中去。命令缓冲由[`VkCommandPool`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandPool.html)（命令池）分配，每个命令池与一个特定的队列家族相关联。为了画出一个三角形，我们需要在命令缓冲里记录下列操作：
* 开始渲染过程
* 绑定图形渲染管线
* 画三个顶点
* 结束渲染过程

因为帧缓冲中的图像取决于交换链具体会给我们哪一个，我们需要为每一个可能使用的图像记录一个命令缓冲，然后在绘制的时候选择正确的那个。另外一种可行的方法是每一帧都重新记录一次命令缓冲，但是这种方法效率不高。

### 第八步 主循环

现在绘制命令已经被包裹在了命令缓冲里，那主循环就非常直截了当了。首先我们通过`vkAcquireNextImageKHR`来从交换链中获得一个图像，之后为这个图像选择合适的命令缓冲并且用[`vkQueueSubmit`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueSubmit.html)来执行。最后，我们用`vkQueuePresentKHR`把这个图像返回到交换链中以供显示。

提交到队列中的操作会被异步执行。因此我们必须使用一个类似信号量这样的同步对象来保证执行顺序正确。绘图命令缓存必须被设置为等到获取图像之后再执行，否则可能导致我们开始渲染的那个图像还在被读取并在屏幕上显示。`vkQueuePresentKHR`函数反过来又需要等待渲染完成，为此我们需要第二个信号量，并且在渲染完成后发出信号。

### 总结

以上这些简单的讲解应该让你对绘制一个三角形所需要做的工作有了一个基本的认识。然而我们真正要做的程序包含了更多的步骤，比如分配顶点缓存，创建uniform缓存以及上传纹理图像等，这些将会在后面的章节讲解。我们先从一个简单的例子开始，因为Vulkan的学习曲线非常陡峭。我们作了一点小弊，把顶点坐标硬编码到了顶点着色器里而不是使用顶点缓存，因为管理顶点缓存需要先对命令缓存有一定的了解。

所以长话短说，画出第一个三角形需要：
* 创建一个[`VkInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstance.html)（实例）
* 选择一个受支持的显卡（[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)（物理设备））
* 为绘制和显示创建一个[`VkDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDevice.html)（逻辑设备）和[`VkQueue`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkQueue.html)（队列）
* 创建一个窗口、表面和交换链
* 把交换链里的图像包裹到[`VkImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageView.html)（图像视图）里面
* 创建一个渲染过程来指定渲染目标和渲染目标的使用方法
* 为渲染过程创建帧缓冲
* 设置图形渲染管线
* 为交换链中每个可用的图像分配并用绘制命令记录命令缓冲
* 通过获取图像绘制帧，提交正确的那个渲染命令缓存并把图像返回到交换链中

步骤有很多，但是在接下来的章节中，每一步的目标都会变得非常明确而简单。如果你对某一步在整个程序中的作用有疑惑，你应该回来参考本章。

## API概念

本章将简要概述Vulkan API在更低的级别上的结构。

### 代码约定

Vulkan中所有的函数、枚举类型和结构体都定义在了`vulkan.h`头文件中，这个文件在LunarG开发的[Vulkan SDK](https://lunarg.com/vulkan-sdk/)里。下一章我们将会探讨如何安装这个SDK。

函数以小写的`vk`开头，枚举类型和结构体这样的类型以`Vk`开头，枚举值则以`VK_`开头。这套API严重依赖结构体作为函数参数。举个例子，对象通常以这种形式创建：
```c++
VkXXXCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_XXX_CREATE_INFO;
createInfo.pNext = nullptr;
createInfo.foo = ...;
createInfo.bar = ...;

VkXXX object;
if (vkCreateXXX(&createInfo, nullptr, &object) != VK_SUCCESS) {
    std::cerr << "failed to create object" << std::endl;
    return false;
}
```

Vulkan中的许多结构体要求你在`sType`成员中明确指定结构体类型。`pNext`成员可以是一个指向扩展结构的指针，在此教程中它将被永远置为`nullptr`。创建或销毁一个对象的函数会有一个[`VkAllocationCallbacks`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkAllocationCallbacks.html)参数，允许你为启动内存使用一个自定义的分配器，它在此教程中也将永远被置为`nullptr`。

几乎所有函数的返回值都是一个[`VkResult`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkResult.html)的枚举类型，它要么是`VK_SUCCESS`（成功），要么是一个错误代码。Vulkan规范说明了每个函数会返回什么错误代码以及它们的含义。

### 验证层

就像之前说过的，Vulkan被设计为一个高性能低负载的API。因此它默认的错误检查和调试能力受到很多限制。当你做错了什么事情时，驱动常常是直接崩溃而不是返回一个错误代码——或者更糟糕，在你的显卡上跑得起来，在别的显卡上就完全不行了。

Vulkan允许你启用一个扩展的错误检查功能，也就是“验证层”（validation layers）。验证层是一些可以被插入到API与显卡驱动之间的代码片段，可以用来运行额外的函数参数检查或者追踪内存管理问题。它的优点是你可以在开发的时候启用验证层，然后在发行版本中彻底关闭它以避免性能开销。每个人都可以编写自己的验证层，不过Vulkan SDK提供了一些标准的验证层，我们在教程中用到的就是它们。为了从验证层接收调试信息，你需要注册一个回调函数。

Vulkan中每个操作都非常明确，并且还有如此灵活的验证层可用，其实事实上Vulkan比OpenGL和Direct3D更容易找到错误原因。

在我们开始写代之前还有一步要做，那就是配置开发环境。