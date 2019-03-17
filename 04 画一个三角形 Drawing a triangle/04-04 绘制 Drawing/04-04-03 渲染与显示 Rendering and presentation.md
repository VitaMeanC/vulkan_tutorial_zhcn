# 渲染与显示

## 建立

这是要把所有东西都组合到一起的一章。我们要编写`drawFrame`函数，并且在主循环中调用它来把三角形显示到屏幕上。创建这个函数并且在`mainLoop`中调用它：

```c++
oid mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }
}

...

void drawFrame() {

}
```

## 同步

`drawFrame`函数将执行以下操作：

* 从交换链中请求一个图像
* 执行命令缓冲，并将请求到的图像作为帧缓冲的附件
* 把图片返回到交换链中

这些事件中的每一个都由单个的函数调用来启动，但是它们是异步执行的。函数调用会在操作执行真正完成之前就返回，执行的顺序也是不确定的。很不幸。每一项操作都依赖于前一项操作的完成。

有两种方法来同步交换链事件：屏障（fence）和信号量。这两种对象都可以用作条件操作：一种操作是改变信号，另一种操作是等待一个屏障或一个信号量来从无信号状态跳转到有信号状态。

*（译者注：原文有点别扭，不过搞过多线程编程的人应该都能看懂这一段是什么意思）*

这二者之间的不同在于，屏障可以通过在你的程序中调用类似[`vkWaitForFences`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkWaitForFences.html)的函数来访问，而信号量不能。屏障主要设计为用来在渲染操作中同步你自己的应用程序，而信号量则用来同步命令队列内或者跨命令队列的操作。我们想要同步绘制命令与显示的队列操作，这里非常适合使用信号量。

## 信号量

我们需要一个信号量来表示一个图像已被取回并准备好渲染，还需要另外一个信号量来表示渲染已经完成、可以绘制。新建两个成员变量来保存这两个信号量对象：

```c++
VkSemaphore imageAvailableSemaphore;
VkSemaphore renderFinishedSemaphore;
```

为创建信号量，我们来为这部分教程添加最后一个`create`（创建）函数：`createSemaphores`：

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandPool();
    createCommandBuffers();
    createSemaphores();
}

...

void createSemaphores() {

}
```

创建信号量需要填充[`VkSemaphoreCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSemaphoreCreateInfo.html)，不过在现版本的API中它没有任何需要填充的成员，除了`sType`：

```c++
void createSemaphores() {
    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;
}
```

在Vulkan API的后续版本或者扩展中有可能为`flags`和`pNext`参数添加功能，就像其它的结构体那样。信号量以这种很熟悉的形式调用[`vkCreateSemaphore`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateSemaphore.html)函数来创建：

```c++
if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS ||
    vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS) {

    throw std::runtime_error("failed to create semaphores!");
}
```

信号量需要在程序的最后，当所有命令全部完成、没有需要同步的操作时被清除掉：

```c++
void cleanup() {
    vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
    vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
```

## 从交换链中获取图像

就像之前提过的一样，我们需要在`drawFrame`函数中做的第一件事是从交换链中获取一个图像。重申一次，交换链是一个扩展功能，因此我们必须使用以`vk*KHR`格式命名的函数：

```c++
void drawFrame() {
    uint32_t imageIndex;
    vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
}
```

`vkAcquireNextImageKHR`的头两个参数是逻辑设备和我们希望从中获取图像的交换链。第三个参数指定了请求图像的超时时间，以纳秒（nanosecond）为单位。使用64位无符号整数的最大值来禁用超时。

接下来的两个参数指定了当显示引擎使用图像结束后要改变哪个同步对象的信号。信号改变的时间点就是我们可以开始对其进行绘制的时间点。可以给它指定一个信号量、屏障、或者二者都指定。在这里我们用我们的`imageAvailableSemaphore`来应对这种情况。

*（译者注：`VK_NULL_HANDLE`的位置就是可以指定屏障的位置）*

最后一个参数指定了一个变量，用来输出可用的交换链图像的索引（index）。这个索引代表了我们的`swapChainImages`数组中的一个[`VkImage`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImage.html)。我们会用这个索引来选择正确的命令缓冲。

## 提交命令缓冲

队列的提交和同步通过[`VkSubmitInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSubmitInfo.html)结构体进行配置。

```c++
VkSubmitInfo submitInfo = {};
submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

VkSemaphore waitSemaphores[] = {imageAvailableSemaphore};
VkPipelineStageFlags waitStages[] = {VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT};
submitInfo.waitSemaphoreCount = 1;
submitInfo.pWaitSemaphores = waitSemaphores;
submitInfo.pWaitDstStageMask = waitStages;
```

前三个参数指定了在处理开始之前哪些信号量要保持等待，以及要等待图形渲染管线的哪个或哪些阶段。我们想要等待到图像可用为止以便向其写入颜色，因此我们指定了图形渲染管线中向颜色附件进行写入的阶段。这意味着从理论上讲，这个实现能够在我们的图像尚未可用的时候就开始处理顶点着色器。`waitStages`数组中的每个入口点都对应着`pWaitSemaphores`中与其相同索引的信号量。

```c++
submitInfo.commandBufferCount = 1;
submitInfo.pCommandBuffers = &commandBuffers[imageIndex];
```

接下来的两个参数指定了哪些命令缓冲将会被实际地提交并执行。就像之前提过的那样，我们应该提交那个绑定了刚刚取回的交换链图像作为颜色附件的命令缓冲。

```c++
VkSemaphore signalSemaphores[] = {renderFinishedSemaphore};
submitInfo.signalSemaphoreCount = 1;
submitInfo.pSignalSemaphores = signalSemaphores;
```

`signalSemaphoreCount`和`pSignalSemaphores`参数指定了当命令缓冲执行结束之后要改变哪些信号量的信号。在我们这里使用了`renderFinishedSemaphore`来应对这种情况。

```c++
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

现在我们可以使用[`vkQueueSubmit`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueSubmit.html)向图形队列提交命令缓冲了。这个函数接受一个[`VkSubmitInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSubmitInfo.html)结构体作为参数，以在工作量很大的时候保持效率。最后一个参数引用了一个可选的屏障，当命令缓冲执行完成后，这个屏障的信号会被改变。我们使用信号量进行同步，因此这里只传递一个`VK_NULL_HANDLE`。

## 子过程依赖

记住，一个渲染过程中的子过程会自动处理图像布局过渡。这些过渡由“子过程依赖”控制，它指定了子通道之间内存和执行的依赖关系。现在我们只有一个子过程，但是在这个子过程之前和之后的操作也会被算作隐式的“子过程”。

有两个内建的依赖处理渲染过程开始和结束时的过渡，但是前者不会发生在正确的时间。它假设过渡发生在图形渲染管线的开始，但是这个时候我们还没获得图像！有两种方法来解决这个问题。我们可以把`imageAvailableSemaphore`的`waitStages`改成`VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT`来确保图像可用之后才启动渲染过程，或者我们可以让渲染过程在`VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT`阶段等待。我决定采用第二种方法，因为这样可以很好地观察子过程依赖及其工作方式。

子过程依赖定义在[`VkSubpassDependency`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSubpassDependency.html)结构体中。跳转到`createRenderPass`函数并且添加一个结构体：

```c++
VkSubpassDependency dependency = {};
dependency.srcSubpass = VK_SUBPASS_EXTERNAL;
dependency.dstSubpass = 0;
```

头两个字段指定了依赖和被依赖的子过程的索引。特殊值`VK_SUBPASS_EXTERNAL`指的是渲染过程开始之前或者结束之后的那个隐式的子过程，到底指哪个取决于它定义在了`srcSubpass`还是`dstSubpass`。索引值`0`代表了我们的子过程，它是第一个，也是唯一一个子过程。`dstSubpass`必须永远比`srcSubpass`高，以避免循环依赖。

```c++
dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.srcAccessMask = 0;
```

接下来的两个字段指定了要等待的操作以及这些操作发生的阶段。在我们能够访问图像之前，我们需要等待到交换链完成对图像的读取为止。这可以通过等待颜色附件输出阶段本身来完成。

```c++
dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_READ_BIT | VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;
```

应该等待的操作是颜色附件阶段的读取和写入操作。这些设置可以避免过渡在不需要（且不应该）它的时候发生：当我们正在向颜色附件写入颜色的时候。

```c++
renderPassInfo.dependencyCount = 1;
renderPassInfo.pDependencies = &dependency;
```

[`VkRenderPassCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkRenderPassCreateInfo.html)结构体有两个数组来指定一个存放依赖的数组。

## 显示

绘制一帧的最后一步就是把结果返回到交换链中，使其最终可以被显示到屏幕上。显示由`VkPresentInfoKHR`结构体在`drawFrame`函数的末尾进行配置。

```c++
VkPresentInfoKHR presentInfo = {};
presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;

presentInfo.waitSemaphoreCount = 1;
presentInfo.pWaitSemaphores = signalSemaphores;
```

头两个参数指定了在可以显示之前需要等待哪个（些）信号量，就像[`VkSubmitInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSubmitInfo.html)一样。

```c++
VkSwapchainKHR swapChains[] = {swapChain};
presentInfo.swapchainCount = 1;
presentInfo.pSwapchains = swapChains;
presentInfo.pImageIndices = &imageIndex;
```

接下来的两个参数指定了要显示图像的交换链，以及每个交换链中图像的索引。这里应该始终只有一个交换链。

```c++
presentInfo.pResults = nullptr; // Optional
```

这是一个叫做`pResults`的可选参数。它允许你指定一个[`VkResult`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkResult.html)的数组来检查每个交换链的显示是否成功。当你只使用一个交换链的时候没有必要用这个，因为你可以简单地使用显示函数的返回值。

```c++
vkQueuePresentKHR(presentQueue, &presentInfo);
```

`vkQueuePresentKHR`函数将提交一个把一个图像显示到交换链的请求，我们将在下一章对`vkAcquireNextImageKHR`和`vkQueuePresentKHR`函数添加错误处理，因为与我们迄今为止看到的那些函数不同，它们的失败不代表整个程序应该退出。

如果目前为止你做对了所有事情，那么当运行程序时，你应该看到类似下面的东西：

![](https://vulkan-tutorial.com/images/triangle.png)

耶！不幸的是，你会看到当验证层开启的时候，程序会在你关闭它的时候崩溃。`debugCallback`打印到终端上的信息告诉了我们原因：

![](https://vulkan-tutorial.com/images/semaphore_in_use.png)

记住，所有`drawFrame`中的操作都是异步的。这意味着当我们退出`mainLoop`中的循环时，渲染和显示操作依然有可能会继续。在这些操作正在进行的时候清除所有资源不是什么好主意。

为了解决这个问题，我们应该在退出`mainLoop`和销毁窗口之前等待逻辑设备执行完所有操作：

```c++
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
        glfwPollEvents();
        drawFrame();
    }

    vkDeviceWaitIdle(device);
}
```

你也可以使用[`vkQueueWaitIdle`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueWaitIdle.html)来等待某一特定的命令队列执行完成。这些函数是执行同步操作的基本方法。现在在关闭窗口的时候你会看到程序会退出而不产生错误了。

## 处理中的帧

如果你在启用了验证层的情况下运行你的程序，并且观察程序的内存占用量，你或许会发现它在缓慢增长。原因是程序在`drawFrame`函数中飞快地进行提交操作，但是没有检查这些操作是否完成了。如果CPU的提交操作太快导致GPU跟不上的话，队列就会缓慢地被这些操作占用。更糟的是，我们在重复使用`imageAvailableSemaphore`和`renderFinishedSemaphore`同时处理多个帧。

解决这个问题的简单方法是在提交之后等待提交操作完成，比如使用[`vkQueueWaitIdle`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueWaitIdle.html)：

```c++
void drawFrame() {
    ...

    vkQueuePresentKHR(presentQueue, &presentInfo);

    vkQueueWaitIdle(presentQueue);
}
```

然而，这种方式可能使我们无法最佳化利用GPU，因为现在整个图形渲染管线在同一时间只处理一个帧。这些阶段只能在当前帧处理结束、处于空闲状态时才能处理下一个帧。我们现在会扩展我们的程序来允许多个帧进入“正在处理”（in-flight）模式，同时仍然限制累积起来的操作总量。

首先，在程序的最顶端添加一个常量来定义有多少帧需要被同时处理：

```c++
const int MAX_FRAMES_IN_FLIGHT = 2;
```

每个帧都应该有自己的一组信号量：

```c++
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
```

`createSemaphores`函数应该被修改，以创建所有的这些信号量：

```c++
void createSemaphores() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create semaphores for a frame!");
        }
}
```

类似地，它们也应该全部被清除：

```c++
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
    }

    ...
}
```

为了使用正确的信号量对，我们需要追踪当前帧。我们会使用一个帧索引来应对这种情况：

```c++
size_t currentFrame = 0;
```

`drawFrame`函数现在也需要被修改来使用正确的对象：

```c++
void drawFrame() {
    vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

    ...

    VkSemaphore waitSemaphores[] = {imageAvailableSemaphores[currentFrame]};

    ...

    VkSemaphore signalSemaphores[] = {renderFinishedSemaphores[currentFrame]};

    ...
}
```

当然，不要忘了每次都要进入下一帧：

```c++
void drawFrame() {
    ...

    currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
}
```

通过使用取余（%）运算符，我们可以确保帧索引的循环处于每`MAX_FRAMES_IN_FLIGHT`个已排队帧的后面。

尽管我们已经设置好了同时处理多个帧所必需的对象，现在还是无法避免比`MAX_FRAMES_IN_FLIGHT`更多的帧被提交。现在这里只有GPU-GPU的同步，而没有CPU-CPU的同步来继续跟踪提交操作的后续情况。有可能出现需要使用#0（第0个）帧，但#0帧仍处于正在处理模式的情况。

为实现CPU-CPU的同步，Vulkan提供了第二种同步原语（synchronization primitive），称为屏障（fence）。在某种意义上，屏障类似于信号量：它们都可以被改变信号和等待。但是这次我们实际上是在我们自己的代码上面等待屏障。首先先来为每个帧创建一个屏障：

```c++
std::vector<VkSemaphore> imageAvailableSemaphores;
std::vector<VkSemaphore> renderFinishedSemaphores;
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;
```

我决定把创建屏障和创建信号量放在一起，然后把`createSemaphores`重命名为`createSyncObjects`：

```c++
void createSyncObjects() {
    imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);
    inFlightFences.resize(MAX_FRAMES_IN_FLIGHT);

    VkSemaphoreCreateInfo semaphoreInfo = {};
    semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

    VkFenceCreateInfo fenceInfo = {};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS ||
            vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS ||
            vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS) {

            throw std::runtime_error("failed to create synchronization objects for a frame!");
        }
    }
}
```

创建屏障（[`VkFence`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkFence.html)）与创建信号量非常相似。还要确保清除了屏障：

```c++
void cleanup() {
    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    ...
}
```

现在我们来更改`drawFrame`以使用屏障进行同步。[`vkQueueSubmit`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueSubmit.html)函数调用包含了一个可选的参数来传递一个在命令缓冲完成执行之后应该被改变信号的屏障。我们可以用这个来表示一个帧已经使用完成。

```c++
void drawFrame() {
    ...

    if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
        throw std::runtime_error("failed to submit draw command buffer!");
    }
    ...
}
```

现在只剩下更改`drawFrame`的开头部分来等待帧使用完成了：

```c++
void drawFrame() {
    vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, std::numeric_limits<uint64_t>::max());
    vkResetFences(device, 1, &inFlightFences[currentFrame]);

    ...
}
```

[`vkWaitForFences`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkWaitForFences.html)函数接受一个屏障数组并且在返回前等待其中的一个或所有屏障改变了信号。我们在这里传递的`VK_TRUE`代表了我们想要等待所有的屏障，但是在只有一个屏障的情况下这个选项造不成什么影响。就像`vkAcquireNextImageKHR`一样，这个函数也需要一个超时时间。与信号量不同的是，我们需要通过调用[`vkResetFences`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkResetFences.html)函数来手动把屏障复位到无信号（unsignaled）状态。

如果你现在运行程序，你会发现有些奇怪。这个程序看起来没有绘制任何东西。开启了验证层的话，你会看到如下消息：

![](https://vulkan-tutorial.com/images/unsubmitted_fence.png)

这意味着我们正在等待一个没有被提交的屏障。出现这个问题是因为，在默认情况下，新建的屏障是处于无信号状态的。这意味着如果我们之前没有使用过这个屏障的话，[`vkWaitForFences`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkWaitForFences.html)会一直等待下去。为解决这个问题，我们可以更改屏障的创建过程，来把新建的屏障初始化为有信号状态的，就好像我们已经渲染了一个初始帧：

```c++
void createSyncObjects() {
    ...

    VkFenceCreateInfo fenceInfo = {};
    fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
    fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

    ...
}
```

现在这个程序应该正确运行，而内存泄漏也应该消失了。我们现在已经实现了所有必需的同步操作来保证同一时间内不会有超过2个帧在队列里。注意，对于代码的其他部分，比如最后的清除操作，使用更加粗略的同步操作，比如[`vkDeviceWaitIdle`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDeviceWaitIdle.html)，是完全可以的。你应该基于性能要求来决定采用哪种同步方法。

要通过例子来了解更多同步操作的话，请参阅Khronos的[这份广泛的概述](https://github.com/KhronosGroup/Vulkan-Docs/wiki/Synchronization-Examples#swapchain-image-acquire-and-present)。

## 小结

在写了900行多一点的程序之后，我们终于看到了有东西显示在了屏幕上！固然，创建一个Vulkan程序需要大量工作，但是这种清晰的创建过程也能带来强大的控制能力。我推荐你花些时间重新阅读这些代码，然后在脑海里构建一个包含程序中所有Vulkan对象的模型，以及这些对象之间是如何关联起来的。从现在开始我们将基于这些知识来扩展程序的功能。

在下一章我们会处理一个表现良好的Vulkan程序所必需的一点小问题。

[C++代码](https://vulkan-tutorial.com/code/15_hello_triangle.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)