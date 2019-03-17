# 重新创建交换链

## 简介

我们的程序现在成功地画出了一个三角形，但是还有一些没有正确处理的东西。表面有可能发生了改变而交换链没有一同改变导致二者不兼容。导致这一情况的原因之一是窗口大小发生了改变。我们需要捕获这些事件然后重新创建交换链。

## 重新创建交换链

新建一个`recreateSwapChain`函数，在其中调用`createSwapChain`及其他所有依赖于交换链或者窗口大小的对象的创建函数。

```c++
void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

首先，我们调用了[`vkDeviceWaitIdle`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDeviceWaitIdle.html)，因为与上一章类似，我们不应该修改那些可能正在被使用的资源。显然，我们需要做的第一件事情是重新创建交换链本身。图像视图（image view）需要被重新创建，因为它们是直接基于交换链图像的。渲染过程（render pass）需要被重新创建，因为它们依赖于叫交换链图像的格式。尽管类似于改变窗口大小这种操作很少会导致交换链图像的格式发生变化，但是还是应该处理一下。视口和裁剪矩形的大小是在图形渲染管线创建时指定的，因此图形渲染管线也需要重新建立。如果为视口和裁剪矩形启用动态设置（dynamic state）的话可以避免重建整个图形渲染管线。最后，帧缓冲和命令缓冲也是直接基于交换链图像的。

为保证在重新创建这些对象之前其旧版本已经被清除，我们需要把一部分清除（cleanup）代码挪出来，写成一个单独的函数，这样我们就可以在`recreateSwapChain`函数里面调用了。给新函数起个名字叫`cleanupSwapChain`：

```c++
void cleanupSwapChain() {

}

void recreateSwapChain() {
    vkDeviceWaitIdle(device);

    cleanupSwapChain();

    createSwapChain();
    createImageViews();
    createRenderPass();
    createGraphicsPipeline();
    createFramebuffers();
    createCommandBuffers();
}
```

我们会把重新创建交换链时涉及的所有需要重新创建的对象的清除代码从`cleanup`挪到`cleanupSwapChain`里：

```c++
void cleanupSwapChain() {
    for (size_t i = 0; i < swapChainFramebuffers.size(); i++) {
        vkDestroyFramebuffer(device, swapChainFramebuffers[i], nullptr);
    }

    vkFreeCommandBuffers(device, commandPool, static_cast<uint32_t>(commandBuffers.size()), commandBuffers.data());

    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);

    for (size_t i = 0; i < swapChainImageViews.size(); i++) {
        vkDestroyImageView(device, swapChainImageViews[i], nullptr);
    }

    vkDestroySwapchainKHR(device, swapChain, nullptr);
}

void cleanup() {
    cleanupSwapChain();

    for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
        vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
        vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
        vkDestroyFence(device, inFlightFences[i], nullptr);
    }

    vkDestroyCommandPool(device, commandPool, nullptr);

    vkDestroyDevice(device, nullptr);

    if (enableValidationLayers) {
        DestroyDebugReportCallbackEXT(instance, callback, nullptr);
    }

    vkDestroySurfaceKHR(instance, surface, nullptr);
    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

我们可以从头开始重新创建命令池，但是这样相当浪费（资源）。作为替代，我选择了使用[`vkFreeCommandBuffers`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkFreeCommandBuffers.html)函数清除现有的命令缓冲。这样我们就可以利用已经创建的命令池来分配新的命令缓冲。

为了正确处理窗口大小的改变，我们还需要来查询帧缓冲现在的大小来保证交换链图像的大小是正确的（新的）。为此我们修改`chooseSwapExtent`函数来考虑实际大小：

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max()) {
        return capabilities.currentExtent;
    } else {
        int width, height;
        glfwGetFramebufferSize(window, &width, &height);

        VkExtent2D actualExtent = {
            static_cast<uint32_t>(width),
            static_cast<uint32_t>(height)
        };

        ...
    }
}
```

这就是重新创建交换链所需要的全部步骤！然而这种方式的缺点在于我们需要停止所有渲染操作直到新交换链创建完成。在创建新交换链的同时，可以让旧交换链上的绘制命令继续进行。你需要通过`VkSwapchainCreateInfoKHR`结构体的`oldSwapChain`字段来传递之前的交换链，然后在你用完旧交换链的时候销毁它。

## 未经优化的或过期的交换链

现在我们只需要找出什么时候需要重新创建交换链，然后调用我们新的`recreateSwapChain`函数就行了。幸运的是，Vulkan通常会在显示过程中告知我们交换链已经不再适用。`vkAcquireNextImageKHR`和`vkQueuePresentKHR`函数可以返回如下的特殊值来表明这一点：

* `VK_ERROR_OUT_OF_DATE_KHR`：交换链已经与表面不兼容且无法再用于渲染。通常会在窗口大小改变后发生。
* `VK_SUBOPTIMAL_KHR`：交换链可以继续用于渲染，但是其与表面属性不再完全匹配。

```c++
VkResult result = vkAcquireNextImageKHR(device, swapChain, std::numeric_limits<uint64_t>::max(), imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);

if (result == VK_ERROR_OUT_OF_DATE_KHR) {
    recreateSwapChain();
    return;
} else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR) {
    throw std::runtime_error("failed to acquire swap chain image!");
}
```

如果交换链在试图获取一个新图像的时候过期，那么它将无法再用于显示。因此我们应该立即重建交换链并且在下一次调用`drawFrame`函数时重试。

然而，如果我们在这时放弃了绘制，那么屏障（fence）就不会通过[`vkQueueSubmit`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkQueueSubmit.html)函数进行提交，在稍后我们尝试等待它的时候其可能处于一个意外的状态。我们可以把重新创建屏障作为重新创建交换链的一个部分，不过移动[`vkResetFences`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkResetFences.html)的调用更简单：

```c++
vkResetFences(device, 1, &inFlightFences[currentFrame]);

if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

你也可以在交换链不再适用的时候这么做，但是我选择在这时继续渲染，因为我们已经请求到了一个图像。`VK_SUCCESS`和`VK_SUBOPTIMAL_KHR`都可以认为是“成功”的返回值。

```c++
result = vkQueuePresentKHR(presentQueue, &presentInfo);

if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR) {
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    throw std::runtime_error("failed to present swap chain image!");
}

currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHT;
```

`vkQueuePresentKHR`函数返回同样的值，意思也是一样的。在这种情况下如果交换链变得不再适用我们也会重新创建交换链，因为我们想尽可能获得好的结果。

## 显式处理大小的改变

尽管在窗口大小改变之后，大多数驱动和平台都会自动触发`VK_ERROR_OUT_OF_DATE_KHR`，但这种行为是不受保证的。这也就是我们要添加一些额外的代码来显式处理大小的改变的理由。首先添加一个新的成员变量来表示大小是否被改变：

```c++
std::vector<VkFence> inFlightFences;
size_t currentFrame = 0;

bool framebufferResized = false;
```

`drawFrame`函数需要被修改以检查此标志：

```c++
if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized) {
    framebufferResized = false;
    recreateSwapChain();
} else if (result != VK_SUCCESS) {
    ...
}
```

在`vkQueuePresentKHR`之后再执行这一操作非常重要，这样可以确保信号量处于一致的状态，否则可能永远无法正确等待一个改变了信号的信号量。现在为了检测大小的改变，我们可以使用GLFW框架中的`glfwSetFramebufferSizeCallback`函数来设置一个回调函数：

```c++
void initWindow() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);

    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
    glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
}

static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {

}
```

之所以创建了一个`static`函数是因为GLFW不知道该如何使用正确的、指向我们的`HelloTriangleApplication`实例的`this`指针来正确地调用成员函数。

不过，我们在回调函数得到了一个`GLFWwindow`的引用，并且有另一个函数允许你向它里面保存一个任意类型的指针：`glfwSetWindowUserPointer`：

```c++
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
glfwSetWindowUserPointer(window, this);
glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

这个值现在可以在回调函数里通过`glfwGetWindowUserPointer`取回，然后正确地设置标志：

```c++
static void framebufferResizeCallback(GLFWwindow* window, int width, int height) {
    auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
    app->framebufferResized = true;
}
```

现在尝试运行程序并且调整窗口大小来看看帧缓冲的大小是否的确随着窗口大小改变了。

## 处理最小化

还有一种情况会导致交换链过期，并且这也是一种特殊的窗口大小改变：窗口最小化。这种情况特殊在它会导致帧缓冲的大小变为`0`。在这篇教程中我们会以暂停渲染直到窗口回到前台的方式处理最小化，这需要扩展`recreateSwapChain`函数：

```c++
void recreateSwapChain() {
    int width = 0, height = 0;
    while (width == 0 || height == 0) {
        glfwGetFramebufferSize(window, &width, &height);
        glfwWaitEvents();
    }

    vkDeviceWaitIdle(device);

    ...
}
```

恭喜，你现在完成了你的第一个表现良好的Vulkan程序！在下一章里我们会摆脱硬编码在顶点着色器里的顶点，然后开始使用顶点缓冲。

[C++代码](https://vulkan-tutorial.com/code/16_swap_chain_recreation.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)