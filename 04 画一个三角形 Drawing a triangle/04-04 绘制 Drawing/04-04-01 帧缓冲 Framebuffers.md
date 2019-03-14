# 帧缓冲

在前几章我们已经提到了帧缓冲好几次，并且我们已经创建了一个接受一个与交换链图片相同格式的帧缓冲的渲染过程（render pass），不过实际上我们还没有创建帧缓冲。

在渲染过程创建时指定的附件（attachment）会被包裹到[`VkFramebuffer`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkFramebuffer.html)对象中进行绑定。一个帧缓冲对象会引用所有代表附件的[`VkImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageView.html)对象。我们这里只有一个附件：颜色附件。然而，我们要用做附件的图像取决于当我们取回一个图像用于显示时交换链会返回哪个。这意味着我们需要为交换链中的每一个图像创建一个帧缓冲，并且在绘制的时候使用对应着取回的图像的那个。

为此，创建另一个`std::vector`成员来保存帧缓冲：

```c++
std::vector<VkFramebuffer> swapChainFramebuffers;
```

我们会在一个新函数，`createFramebuffers`里为这个数组创建对象，并且在`initVulkan`里、创建了图形渲染管线之后调用它：

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
}

...

void createFramebuffers() {

}
```

首先，resize容器的大小以保存所有帧缓冲：

```c++
void createFramebuffers() {
    swapChainFramebuffers.resize(swapChainImageViews.size());
}
```

然后，遍历所有图像视图来为它们创建帧缓冲：

```c++
for (size_t i = 0; i < swapChainImageViews.size(); i++) {
    VkImageView attachments[] = {
        swapChainImageViews[i]
    };

    VkFramebufferCreateInfo framebufferInfo = {};
    framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
    framebufferInfo.renderPass = renderPass;
    framebufferInfo.attachmentCount = 1;
    framebufferInfo.pAttachments = attachments;
    framebufferInfo.width = swapChainExtent.width;
    framebufferInfo.height = swapChainExtent.height;
    framebufferInfo.layers = 1;

    if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create framebuffer!");
    }
}
```

如你所见，帧缓冲的创建非常简单明了。我们首先需要指定哪个`renderPass`是帧缓冲需要兼容的。你只能使用与渲染过程兼容的帧缓冲，“兼容”大致上意味着它们拥有相同数量和类型的附件。

`attachmentCount`和`pAttachments`参数指定了需要被绑定到各自的附件描述的[`VkImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageView.html)对象，这些附件描述就是渲染过程中`pAttachment`数组中的那些。

*（译者注：参考《渲染过程》一章中“渲染过程”一节，`pAttachment`数组指的应该是`VkRenderPassCreateInfo`的`pAttachments`成员）*

`width`和`height`参数是自注释的，`layers`是指图像数组中图层的个数。我们的交换链图像都是单个图像，因此图层的数量是`1`。

我们需要在删除图像视图及其依赖的渲染过程之前删除帧缓冲，不过只能在我们完成渲染之后：

```c++
void cleanup() {
    for (auto framebuffer : swapChainFramebuffers) {
        vkDestroyFramebuffer(device, framebuffer, nullptr);
    }

    ...
}
```

我们现在已经抵达了一个里程碑——我们拥有了渲染所需的所有对象。在下一章里，我们会开始写实际的绘制命令。

[C++代码](https://vulkan-tutorial.com/code/13_framebuffers.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)