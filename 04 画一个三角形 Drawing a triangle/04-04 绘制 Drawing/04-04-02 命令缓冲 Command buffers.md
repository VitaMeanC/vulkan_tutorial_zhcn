# 命令缓冲

Vulkan中的命令，比如绘制操作和内存传递，不是由函数调用直接执行的。你需要在命令缓冲对象中记录下你想要进行的所有操作。这么做的优点是，可以提前完成设置图形命令中的所有困难的工作，并且可以使用多线程。在这之后，你只需要告知Vulkan在主循环中处理这些命令就可以了。

## 命令池 

在创建命令缓冲之前，我们需要创建一个命令池。命令池管理那些用于存储缓冲的内存，命令缓冲就从这些内存中分配。添加一个新的成员变量来保存[`VkCommandPool`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandPool.html)：

```c++
VkCommandPool commandPool;
```

然后创建一个新的函数：`createCommandPool`，并且在`initVulkan`中、创建了帧缓冲之后调用它。

```c++
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
}

...

void createCommandPool() {

}
```

创建命令池只需要两个参数：

```c++
QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

VkCommandPoolCreateInfo poolInfo = {};
poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO;
poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value();
poolInfo.flags = 0; // Optional
```

命令缓冲通过被提交到一个设备队列的方式执行，比如我们取回的图形和显示队列。每个命令池只能分配被提交到特定的一种队列的命令缓冲。我们接下来要记录绘制命令，这就是我们选择了图形队列家族的原因。

对于命令池有两种可用的标志（flag）：

* `VK_COMMAND_POOL_CREATE_TRANSIENT_BIT`：表示命令缓冲会经常重新记录新命令（有可能改变内存分配的行为）
* `VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT`：允许命令缓冲单独地被重新记录，没有这个标志的话所有命令缓冲需要被一起重置

我们只会在程序的开头记录命令缓冲，然后在主循环中多次执行它们，所以我们用不到上面的那些标志。

```c++
if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS) {
    throw std::runtime_error("failed to create command pool!");
}
```

最后使用[`vkCreateCommandPool`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateCommandPool.html)函数创建命令池。这个函数没有任何特殊的参数。命令会在整个程序中被使用，以把东西绘制到屏幕上，所以命令池只应该在程序末尾被销毁：

```c++
void cleanup() {
    vkDestroyCommandPool(device, commandPool, nullptr);

    ...
}
```

## 分配命令缓冲

我们现在可以开始分配命令缓冲并且在其中记录绘制命令了。因为每个绘制命令都需要绑定到正确的[`VkFramebuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkFramebuffer.html)上，我们实际上需要为每个交换链中的每个图像都记录一次命令缓冲。为此，新建一个[`VkCommandBuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandBuffer.html)的列表作为成员变量。命令缓冲会在其所属的命令池销毁后自动被释放，因此不需要显式清除它。

```c++
std::vector<VkCommandBuffer> commandBuffers;
```

我们现在开始在`createCommandBuffers`函数里写代码，来为每个交换链图像分配并记录命令。

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
}

...

void createCommandBuffers() {
    commandBuffers.resize(swapChainFramebuffers.size());
}
```

命令缓冲由[`vkAllocateCommandBuffers`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkAllocateCommandBuffers.html)函数分配，这个函数需要一个[`VkCommandBufferAllocateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandBufferAllocateInfo.html)结构体作为参数，这个结构体指定了命令池以及需要分配的缓冲数量：

```c++
VkCommandBufferAllocateInfo allocInfo = {};
allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
allocInfo.commandPool = commandPool;
allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
allocInfo.commandBufferCount = (uint32_t) commandBuffers.size();

if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS) {
    throw std::runtime_error("failed to allocate command buffers!");
}
```

`level`参数指定了分配的命令缓冲是主要的（primary）还是次要的（secondary）命令缓冲。

* `VK_COMMAND_BUFFER_LEVEL_PRIMARY`（主要）：可以被提交到一个队列去执行，但是不能被其他命令缓冲调用。
* `VK_COMMAND_BUFFER_LEVEL_SECONDARY`（次要）：不能直接被执行，但是可以被主要的命令缓冲调用。

在这里我们不会用到次要的命令缓冲，但是你可以想象它对复用公共操作的帮助有多大。

## 开始记录命令缓冲

我们通过调用[`vkBeginCommandBuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkBeginCommandBuffer.html)函数并使用一个小[`VkCommandBufferBeginInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkCommandBufferBeginInfo.html) 结构体——用来指定有关于这个命令缓冲的使用方法的一些细节——作为参数来开始记录命令缓冲。

```c++
for (size_t i = 0; i < commandBuffers.size(); i++) {
    VkCommandBufferBeginInfo beginInfo = {};
    beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
    beginInfo.flags = VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT;
    beginInfo.pInheritanceInfo = nullptr; // Optional

    if (vkBeginCommandBuffer(commandBuffers[i], &beginInfo) != VK_SUCCESS) {
        throw std::runtime_error("failed to begin recording command buffer!");
    }
}
```

`flags`参数指定了我们将要如何使用这个命令缓冲。有以下值可用：

* `VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT`：这个命令缓冲在执行一次之后会被重新记录。
* `VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT`：这是一个次要的命令缓冲，它会完全处于一个单独的渲染过程中。
* `VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT`：命令缓冲可以被重新提交，尽管其已经在等待执行。

我们使用了最后一种标志（flag），因为有可能提交了下一帧的绘制命令时当前帧还没有绘制完毕。`pInheritanceInfo`参数只与次要的命令缓冲有关。它指定了要从调用它的主要的命令缓冲继承的状态（state）。

如果一个命令缓冲已经记录过一次了，那么调用[`vkBeginCommandBuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkBeginCommandBuffer.html)会隐式地将其重置。对一个命令缓冲追加命令是不可能的。

## 启动渲染过程

绘制开始于使用[`vkCmdBeginRenderPass`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCmdBeginRenderPass.html)启动渲染过程。渲染过程由[`VkRenderPassBeginInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkRenderPassBeginInfo.html)结构体的一些参数配置。

```c++
VkRenderPassBeginInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
renderPassInfo.renderPass = renderPass;
renderPassInfo.framebuffer = swapChainFramebuffers[i];
```

头两个参数是渲染过程本身以及绑定的附件。我们为每个交换链图像创建了一个帧缓冲并且将其指定为颜色附件。

```c++
renderPassInfo.renderArea.offset = {0, 0};
renderPassInfo.renderArea.extent = swapChainExtent;
```

接下来的两个参数定义了要渲染的区域的大小。渲染区域定义了着色器加载和存储的位置。位于这个区域之外的像素会被设为未定义（undefined）的值。它应该与附件的大小相匹配以获得最佳性能。

```c++
VkClearValue clearColor = {0.0f, 0.0f, 0.0f, 1.0f};
renderPassInfo.clearValueCount = 1;
renderPassInfo.pClearValues = &clearColor;
```

最后两个参数定义了用于`VK_ATTACHMENT_LOAD_OP_CLEAR`的清除值，我们将其用作颜色附件的加载操作。我把清除颜色（clear color）简单地定义为了100%不透明度的黑色。

```c++
vkCmdBeginRenderPass(commandBuffers[i], &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
```

现在，渲染过程可以启动了。所有记录命令的函数都可以通过它们的前缀[`vkCmd`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCmd.html)认出来。它们的返回类型都是`void`，因此直到完成记录之前都无法进行错误处理。

每个命令的第一个参数都是要记录该命令的命令缓冲。第二个参数指定了我们刚才提供的渲染过程中的一些细节。最后一个参数用于控制渲染过程中的命令缓冲将如何被提供。它可以被设为如下两个值之一：

* `VK_SUBPASS_CONTENTS_INLINE`：渲染过程中的命令会被嵌入到主要的命令缓冲中，次要的命令缓冲将不会被执行。
* `VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS`：渲染过程中的命令将从次要的命令缓冲中执行。

我们不使用次要的命令缓冲，因此选择第一个选项。

## 基础绘图命令

我们现在可以绑定图形渲染管线了：

```c++
vkCmdBindPipeline(commandBuffers[i], VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
```

第二个参数指定了这个管线是图形管线还是计算管线。我们现在已经告诉过Vulkan哪些操作是要在图形渲染管线中处理的，以及哪些附件将在片段着色器中被使用，所以现在只剩下告诉Vulkan画一个三角形了：

```c++
vkCmdDraw(commandBuffers[i], 3, 1, 0, 0);
```

实际的[`vkCmdDraw`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCmdDraw.html)函数看上去有点虎头蛇尾，但是因为我们之前已经指定了所有的信息，所以这个函数看起来才如此简单。除了命令缓冲之外，它还具有以下参数：

* `vertexCount`：尽管我们没有顶点缓冲，技术上来讲我们仍然有3个顶点要绘制。
* `instanceCount`：用于实例化渲染（instanced rendering），如果不需要的话就设为`1`。
* `firstVertex`：用作顶点缓冲的偏移量（offset），定义`gl_VertexIndex`的最小值。
* `firstInstance`：用作实例化渲染的偏移量，定义`gl_InstanceIndex`的最大值。

## 完成

渲染过程现在可以结束了：

```c++
vkCmdEndRenderPass(commandBuffers[i]);
```

并且我们也完成了命令缓冲的记录：

```c++
if (vkEndCommandBuffer(commandBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to record command buffer!");
}
```

在下一章里我们会写主循环里的代码：从交换链中获取一个图像，处理正确的命令缓冲然后把完成的图像返回给交换链。

[C++代码](https://vulkan-tutorial.com/code/14_command_buffers.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)