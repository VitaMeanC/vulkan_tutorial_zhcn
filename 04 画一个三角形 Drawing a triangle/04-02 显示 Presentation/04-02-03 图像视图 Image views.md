# 图像视图

要在渲染管线中使用[`VkImage`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImage.html)，包括交换链中的，我们就需要创建一个[`VkImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageView.html)对象。一个图像视图（image view）实际上就是一个图像的一个视图。它描述了如何访问图像，以及访问图像的哪一部分，例如这个图像应该被视为一个2D深度纹理，并且不创建多级渐远纹理。

在这一章我们会写一个`createImageViews`函数来为交换链中的每个图像创建一个基本的图像视图，这样一会儿我们就能把它们用作颜色目标了。

首先添加一个成员函数来保存图像视图：

```c++
std::vector<VkImageView> swapChainImageViews;
```

创建`createImageViews`函数，并且在创建交换链之后调用。

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
    createImageViews();
}

void createImageViews() {

}
```

首先我们需要resize这个列表，使其与我们要创建的图像视图的数量相吻合：

```c++
void createImageViews() {
    swapChainImageViews.resize(swapChainImages.size());

}
```

接下来，写一个循环来遍历交换链中的图像。

```c++
for (size_t i = 0; i < swapChainImages.size(); i++) {

}
```

创建图像视图的参数由[`VkImageViewCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImageViewCreateInfo.html)结构体指定。头几个参数非常简单明了。

```c++
VkImageViewCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
createInfo.image = swapChainImages[i];
```

`viewType`和`format`参数指定了应该如何解释图像数据。`viewType`参数允许你将图像看作1D纹理、2D纹理、3D纹理或立方体贴图。

```c++
createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
createInfo.format = swapChainImageFormat;
```

`components`参数允许你映射颜色通道。例如，你可以把所有通道映射到红色通道上以获得一个单色纹理。你也可以给一个通道映射为`0`或`1`的常量值。在这里我们使用默认映射。

```c++
createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
```

`subresourceRange`参数描述了图像的目标以及应该访问图像的哪一部分。我们的图像会被用作颜色目标，并且没有任何多级渐远纹理或是多个图层。

```c++
createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
createInfo.subresourceRange.baseMipLevel = 0;
createInfo.subresourceRange.levelCount = 1;
createInfo.subresourceRange.baseArrayLayer = 0;
createInfo.subresourceRange.layerCount = 1;
```

如果你写的是一个3D程序，你应该创建一个支持多图层的交换链。这样你可以为每个图像创建多个图像视图，通过访问不同的图层来分别代表左眼和右眼。

现在，调用[`vkCreateImageView`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateImageView.html)来创建图像视图：

```c++
if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image views!");
}
```

不同于图像，图像视图是我们现实创建的，因此我们需要一个循环来在程序末尾销毁它们：

```c++
void cleanup() {
    for (auto imageView : swapChainImageViews) {
        vkDestroyImageView(device, imageView, nullptr);
    }

    ...
}
```

图像视图已经足够把一个图像用作纹理了，但是现在还不能用作渲染目标。它还需要一个间接的步骤，这个步骤叫做帧缓冲（framebuffer）。不过首先我们需要建立图形渲染管线。

[C++代码](https://vulkan-tutorial.com/code/07_image_views.cpp)