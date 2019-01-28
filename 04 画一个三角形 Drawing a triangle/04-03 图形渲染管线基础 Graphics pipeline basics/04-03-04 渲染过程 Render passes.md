# 渲染过程

## 建立

在我们完成图形渲染管线的创建之前，我们需要告知Vulkan在渲染时要使用的帧缓冲的附件（attachment）。我们需要指定颜色缓冲和深度缓冲的数量、每个缓冲中有多少个样本（sample）以及样本中的内容在整个渲染操作中会被怎样处理。所有这些信息都会被“渲染过程”（render pass）对象所包裹，为此我们会新建一个`createRenderPass`函数。在`initVulkan`函数中的`createGraphicsPipeline`后面调用这个函数。

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
}

...

void createRenderPass() {

}
```

## 附件描述

在我们的程序中，我们只有一个简单的颜色缓冲附件，它由交换链中的一个图像表示。

```c++
void createRenderPass() {
    VkAttachmentDescription colorAttachment = {};
    colorAttachment.format = swapChainImageFormat;
    colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
}
```

颜色附件的`format`（格式）应该与交换链图像的格式一致，并且由于我们目前不想进行多重采样，我们就设置为1个样本。

```c++
colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
```

`loadOp`和`storeOp`分别指定在渲染之前和之后如何处理附件中的数据。对于`loadOp`而言，有如下选项：

* `VK_ATTACHMENT_LOAD_OP_LOAD`：保留附件中已有的内容
* `VK_ATTACHMENT_LOAD_OP_CLEAR`：在开始渲染时用一个常量将附件中的值清空
* `VK_ATTACHMENT_LOAD_OP_DONT_CARE`：附件中现有的内容未定义；我们不需要关心它们

在我们的程序中我们会使用清除操作来在绘制新一帧之前把帧缓冲中的颜色清空为黑色。对于`storeOp`二眼只有两种选项：

* `VK_ATTACHMENT_STORE_OP_STORE`：绘制过的内容会被储存在内存中以便稍后读取
* `VK_ATTACHMENT_STORE_OP_DONT_CARE`：帧缓冲中的内容在绘制操作结束后将变为未定义的

我们想在屏幕上看到绘制出的三角形，因此我们在此选择存储操作。

```c++
colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
```

`loadOp`和`storeOp`应用于颜色和深度数据，而`stencilLoadOp`和`stencilStoreOp`则应用于模板数据。我们的程序中不会对模板缓冲执行任何操作，因此加载和存储的结果与我们的程序无关。

```c++
colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

Vulkan中的纹理和帧缓冲由使用特定像素格式的[`VkImage`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImage.html)对象来表示，但是内存中的像素布局可以根据你想对图像执行的操作而进行改变。

一些最常用的布局是：

* `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`：将图像用作颜色附件
* `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`：图像将会在交换链中用作显示
* `VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL`：图像将作为一个内存复制操作的目标

我们会在纹理一节深入探讨这个话题，不过现在需要知道的是，图像需要被转换，以适应它们要参与的下一步操作的特定布局。

`initialLayout`指定了在渲染过程开始之前图像的布局。`finalLayout`指定了渲染过程结束后图像将会自动转换成什么布局。给`initialLayout`指定`VK_IMAGE_LAYOUT_UNDEFINED`意味着我们不关心图像之前是什么布局。使用这个特殊值会导致图像中的内容不一定会被保留，不过对我们来说无所谓，反正我们都会清空图像内容。我们希望图像在渲染之后可以用在交换链里来显示，因此我们给`finalLayout`指定了`VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`。

## 子过程与附件的引用

一个渲染过程可以包含若干子过程。子过程是一些在渲染后期进行的操作，这些操作依赖于之前的过程在帧缓冲中已经绘制的内容，比如一个接一个地应用一系列后期处理效果。如果你把这些渲染操作全都集中到一个渲染过程中去的话，Vulkan就可以对这些操作重新排序，并且节省内存带宽以得到最佳性能。对于我们最初的这个三角形来说，我们只需要一个子过程。

每个子过程都引用了一个或多个附件，就是我们在上一节用结构体描述的那些。这些引用就是[`VkAttachmentReference`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkAttachmentReference.html)结构体，它看起来像是这样：

```c++
VkAttachmentReference colorAttachmentRef = {};
colorAttachmentRef.attachment = 0;
colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

`attachment`参数使用索引指定了在附件描述数组中要引用哪个附件。我们的数组中只有一个[`VkAttachmentDescription`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkAttachmentDescription.html)，因此它的索引是`0`。`layout`指定了在子过程使用这个附件的时候，我们希望附件拥有什么布局。当子过程开始的时候，Vulkan会自动把附件转换成这个布局。我们想把附件当作一个颜色缓冲来使用，而`VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`如其名字所示，可以发挥出最佳的性能。

子过程使用[`VkSubpassDescription`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkSubpassDescription.html)结构体来描述：

```c++
VkSubpassDescription subpass = {};
subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS;
```

在未来，Vulkan有可能支持计算子过程，索移我们必须显式声明这是一个图形子过程。接下来，我们要指定颜色附件的引用：

```c++
subpass.colorAttachmentCount = 1;
subpass.pColorAttachments = &colorAttachmentRef;
```

在这个数组中，附件的索引直接引用片段着色器的`layout(location = 0) out vec4 outColor`指令。

下面几种类型的附件也可以被子过程引用：

* `pInputAttachments`：从着色器中读取出来的附件
* `pResolveAttachments`：用作“多重采样的颜色附件”的附件
* `pDepthStencilAttachment`：用于深度数据或模板数据的附件
* `pPreserveAttachments`：这个子过程不会用到，但是其数据必须被保留的附件

## 渲染过程

现在 ，附件和一个引用了它的基本的子过程就描述好了，我们可以来创建渲染过程本身了。新建一个成员变量来保存[`VkRenderPass`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkRenderPass.html)对象并放在`pipelineLayout`变量之前：

```c++
VkRenderPass renderPass;
VkPipelineLayout pipelineLayout;
```

渲染过程对象可以通过使用附件和子过程的数组来填充[`VkRenderPassCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkRenderPassCreateInfo.html)结构体的方式创建。[`VkAttachmentReference`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkAttachmentReference.html)对象通过这个数组中的索引来引用附件。

```c++
VkRenderPassCreateInfo renderPassInfo = {};
renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
renderPassInfo.attachmentCount = 1;
renderPassInfo.pAttachments = &colorAttachment;
renderPassInfo.subpassCount = 1;
renderPassInfo.pSubpasses = &subpass;

if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS) {
    throw std::runtime_error("failed to create render pass!");
}
```

与管线布局类似，渲染过程也会在整个程序中被引用，因此只能在程序末尾销毁它：

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    vkDestroyRenderPass(device, renderPass, nullptr);
    ...
}
```

到目前为止我们已经做了很多工作，不过下一章我们会把它们组合到一起来最终创建图形渲染管线对象！

[C++代码](https://vulkan-tutorial.com/code/11_render_passes.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)