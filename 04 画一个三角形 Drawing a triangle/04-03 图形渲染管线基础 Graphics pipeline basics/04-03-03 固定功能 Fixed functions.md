# 固定功能

在那些老式图形API中，几乎为图形渲染管线中的每一步都提供了默认值。在Vulkan中则需要你显式指定所有东西，从视口大小到颜色混合函数。在这一章我们会填充所有必需的结构体来配置这些固定功能操作。

## 顶点输入

[`VkPipelineVertexInputStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineVertexInputStateCreateInfo.html)结构体描述了将要传递给顶点着色器的顶点数据的格式。它大致以两种方式描述这个格式：

* 绑定（Bindings）：数据之间的间距（spacing），以及数据是按顶点的还是按实例的（请参阅 [instancing](https://en.wikipedia.org/wiki/Geometry_instancing)（几何体实例化））。
* 描述属性（Attribute descriptions）：传递给顶点着色器的属性的类型，用来加载这些属性，并指明在何处（offset）加载。

因为我们把顶点数据硬编码到了顶点着色器中，因此我们在这里暂时按照没有顶点数据的格式来填充这个结构体，在顶点缓冲一章我们会回来研究它。

```c++
VkPipelineVertexInputStateCreateInfo vertexInputInfo = {};
vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
vertexInputInfo.vertexBindingDescriptionCount = 0;
vertexInputInfo.pVertexBindingDescriptions = nullptr; // Optional
vertexInputInfo.vertexAttributeDescriptionCount = 0;
vertexInputInfo.pVertexAttributeDescriptions = nullptr; // Optional
```

`pVertexBindingDescriptions`和`pVertexAttributeDescriptions`成员指向一个描述之前说过的用来加载顶点数据的详细信息的结构体的数组。把这个结构体加到`createGraphicsPipeline`函数中的`shaderStages`数组后面。

## 顶点装配

[`VkPipelineInputAssemblyStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineInputAssemblyStateCreateInfo.html)结构体描述了两个东西：从顶点绘制什么样的几何体，以及是否启用图元重启（primitive restart）。前一项由`topology`（拓扑）成员指定，并且可以拥有下列值：

* `VK_PRIMITIVE_TOPOLOGY_POINT_LIST`：把每个顶点绘制成点
* `VK_PRIMITIVE_TOPOLOGY_LINE_LIST`：每两个顶点绘制成一条线，不重用顶点
* `VK_PRIMITIVE_TOPOLOGY_LINE_STRIP`每条线的末端顶点用作下一条线的起始顶点
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST`：每三个顶点绘制成一个三角形，不重用顶点
* `VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP`：每个三角形的第二个和第三个顶点作为下一个三角形的前两个顶点

通常来说，从顶点缓冲中按索引加载的顶点都是按顺序排列的，不过使用“索引缓冲”（element buffer）可以让你自己指定想要使用的索引。这允许你进行优化，比如重用顶点。如果你把`primitiveRestartEnable`成员设为了`VK_TRUE`，那么就可以通过使用一个特殊顶点`0xFFFF`或`0xFFFFFFFF`分解`_STRIP`拓扑模式中的线和三角形。

在此教程中我们打算画一个三角形，所以我们用以下数据填充结构体：

```c++
VkPipelineInputAssemblyStateCreateInfo inputAssembly = {};
inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
inputAssembly.primitiveRestartEnable = VK_FALSE;
```

## 视口与裁剪

视口基本上描述了将会被输出所渲染的帧缓冲的范围。这个范围绝大多数情况下都是`(0, 0)`到`(width, height)`，并且在我们的教程中也是如此。

```c++
VkViewport viewport = {};
viewport.x = 0.0f;
viewport.y = 0.0f;
viewport.width = (float) swapChainExtent.width;
viewport.height = (float) swapChainExtent.height;
viewport.minDepth = 0.0f;
viewport.maxDepth = 1.0f;
```

记住，交换链及其中的图像的大小可能与窗口的大小不同。交换链中的图像稍后会被用作帧缓冲，因此我们使用它们的大小。

`minDepth`和`maxDepth`指定了帧缓冲中深度的范围。这些值必须在`[0.0f, 1.0f]`范围中，但是`minDepth`有可能大于`maxDepth`。如果你不打算做什么特别的操作，你应该把它们设为标准值`0.0f`和`1.0f`。

视口定义了从图像到帧缓冲的转换方式，而裁剪矩形定义了什么范围里的像素会被储存。所有在裁剪矩形外面的像素都会在光栅化时被丢弃。裁剪矩形的功能更像是过滤器，而不是转换方式。裁剪矩形与视口之间的差异如下图所示。注意，左边的裁剪矩形只是导致这个输出结果的一种可能，只要裁剪矩形大于视口就会产生类似效果。

![](https://vulkan-tutorial.com/images/viewports_scissors.png)

在此教程中我们只想简单地绘制整个帧缓冲，因此我们定义一个覆盖了整个帧缓冲的裁剪矩形：

```c++
VkRect2D scissor = {};
scissor.offset = {0, 0};
scissor.extent = swapChainExtent;
```

现在视口和裁剪矩形需要通过[`VkPipelineViewportStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineViewportStateCreateInfo.html)结构体链接到一个视口状态（viewport state ）中。在某些显卡上，你可以使用多个视口与裁剪矩形，所以它的成员是视口与裁剪矩形的数组。使用多个视口与裁剪矩形需要打开一个GPU功能（参阅创建逻辑设备一节）。

```c++
VkPipelineViewportStateCreateInfo viewportState = {};
viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
viewportState.viewportCount = 1;
viewportState.pViewports = &viewport;
viewportState.scissorCount = 1;
viewportState.pScissors = &scissor;
```

## 光栅化

光栅化阶段接受由顶点着色器产生的顶点组成的图元，并把它们转换成片段以供片段着色器上色。这一步还会进行[深度测试](https://zh.wikipedia.org/wiki/%E6%B7%B1%E5%BA%A6%E7%BC%93%E5%86%B2)、[面剔除](https://en.wikipedia.org/wiki/Back-face_culling)（英文链接）和裁剪测试，它还可以进行配置，以渲染整个多边形，或者只渲染多边形的边缘（线框渲染）。所有的这些配置都可以通过[`VkPipelineRasterizationStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineRasterizationStateCreateInfo.html)结构体进行。

```c++
VkPipelineRasterizationStateCreateInfo rasterizer = {};
rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
rasterizer.depthClampEnable = VK_FALSE;
```

如果`depthClampEnable`被设为了`VK_TRUE`，那么过近或过远的片段会被放到远近平面中间，而不是丢弃这些片段。（此句存疑，原文：If `depthClampEnable` is set to `VK_TRUE`, then fragments that are beyond the near and far planes are clamped to them as opposed to discarding them.）这在某些情况下很有用，比如阴影映射。使用它需要打开一项GPU功能。

```c++
rasterizer.rasterizerDiscardEnable = VK_FALSE;
```

如果`rasterizerDiscardEnable`被设为了`VK_TRUE`，那么图元将永远不会被传送到光栅化这一步。这基本上禁用了向帧缓冲的输出。

```c++
rasterizer.polygonMode = VK_POLYGON_MODE_FILL;
```

`polygonMode`指定了如何从图元生成片段。有下列模式可用：

* `VK_POLYGON_MODE_FILL`：用片段填充整个图元的范围
* `VK_POLYGON_MODE_LINE`：将图元的边绘制为线段
* `VK_POLYGON_MODE_POINT`：将图元的顶点绘制为点

使用除填充模式以外的模式需要启用一项GPU功能。

```c++
rasterizer.lineWidth = 1.0f;
```

`lineWidth`成员的含义很直观，它描述了线的宽度。最大线宽取决于硬件，并且要使用大于`1.0f`的线宽需要你启用`wideLines`GPU功能。

```c++
rasterizer.cullMode = VK_CULL_MODE_BACK_BIT;
rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE;
```

`cullMode`成员指定你要使用的面剔除类型。你可以关闭面剔除、剔除正面朝向（front face）的面、剔除背面朝向（back face）的面，或者都剔除。`frontFace`成员指定了正面的顶点连接顺序，可以是顺时针或者逆时针。

```c++
rasterizer.depthBiasEnable = VK_FALSE;
rasterizer.depthBiasConstantFactor = 0.0f; // Optional
rasterizer.depthBiasClamp = 0.0f; // Optional
rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

在进行光栅化时，可以通过添加常量值或者基于片段的斜率（slope）进行偏移（bias）来更改深度值。阴影映射中有时会用到这一点，不过我们不会用到。把`depthBiasEnable`设为`VK_FALSE`就好。

## 多重采样

[`VkPipelineMultisampleStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineMultisampleStateCreateInfo.html)结构体设置多重采样，这是实现[反锯齿](https://zh.wikipedia.org/wiki/%E5%8F%8D%E9%8B%B8%E9%BD%92#%E5%A4%9A%E9%87%8D%E9%87%87%E6%A0%B7%E6%8A%97%E9%94%AF%E9%BD%BF)的一种方式。它的工作方式是，把多个多边形执行片段着色器后的结果叠加到相同的像素上去。这主要影响边缘，而边缘也是产生锯齿最明显的地方。多重采样比单纯渲染一个高分辨率图像再缩小的方式开销要小得多，因为如果一个多边形只映射到一个像素时，只需运行一次片段着色器。启用多重采样需要开启一项GPU功能。

```c++
VkPipelineMultisampleStateCreateInfo multisampling = {};
multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
multisampling.sampleShadingEnable = VK_FALSE;
multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
multisampling.minSampleShading = 1.0f; // Optional
multisampling.pSampleMask = nullptr; // Optional
multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

在稍后的章节中我们会再次探讨多重采样，现在暂时把它关闭。

## 深度测试和模板测试

如果你使用了深度缓冲和/或模板缓冲，那么你需要通过[`VkPipelineDepthStencilStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineDepthStencilStateCreateInfo.html)来配置深度测试和模板测试。不过我们现在还没有用到深度缓冲和模板缓冲，因此我们就只传递一个`nullptr`来取代这个结构体的指针。我们会在深度缓冲一章再回来讨论它。

## 颜色混合

在片段着色器返回一个颜色之后，它需要与已经存在于帧缓冲中的颜色进行混合。这种变换叫做“颜色混合”，并且颜色混合有两种方式：

* 把旧的颜色和新的颜色混合之后产生最终的颜色
* 通过位运算来混合旧颜色和新颜色

有两种结构体可以设置颜色混合。第一种结构体，[`VkPipelineColorBlendAttachmentState`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineColorBlendAttachmentState.html)包含了针对每个帧缓冲进行的设置；第二种结构体，[`VkPipelineColorBlendStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineColorBlendStateCreateInfo.html)包含了全局的颜色混合选项。对于我们来说，我们只有一个帧缓冲：

```c++
VkPipelineColorBlendAttachmentState colorBlendAttachment = {};
colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
colorBlendAttachment.blendEnable = VK_FALSE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

这个针对每个帧缓冲的结构体允许你配置颜色混合的第一种方式。下面的伪代码很好地演示了将要执行的操作：

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

如果`blendEnable`被设为了`VK_FALSE`，那么片段着色器产生的新颜色不会经过任何修改就会被传递出去。否则，这两步混合操作就会计算出新的颜色。产生的颜色与`colorWriteMask`进行按位与运算，来决定最终会被传递出去的通道。

最常用的颜色混合是实现alpha混合，新颜色会基于它的不透明度与旧颜色进行混合。这样，`finalColor`就应该使用如下方式计算：

```c++
finalColor.rgb = newAlpha * newColor + (1 - newAlpha) * oldColor;
finalColor.a = newAlpha.a;
```

可以通过以下参数来实现：

```c++
colorBlendAttachment.blendEnable = VK_TRUE;
colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_SRC_ALPHA;
colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ONE_MINUS_SRC_ALPHA;
colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD;
colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE;
colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO;
colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD;
```

你可以在规范中的[`VkBlendFactor`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkBlendFactor.html)和[`VkBlendOp`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkBlendOp.html)枚举值中找到所有可以进行的操作。

第二个结构体引用了一个包含了所有针对单个帧缓冲进行设置的结构体的数组，并且允许你设置混合常量，你可以在上述计算中把这些常量作为混合因子。

```c++
VkPipelineColorBlendStateCreateInfo colorBlending = {};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

如果你想使用颜色混合的第二种方式（位运算混合），那么你应该把`logicOpEnable`设为`VK_TRUE`。位运算可以通过`logicOp`成员来指定。注意，这将会自动禁用第一种方式，就像是你把每个帧缓冲的设置中的`blendEnable`设为了`VK_FALSE`一样！`colorWriteMask`在第二种模式中同样可用，用来指定帧缓冲中的哪些通道最终会起作用。也可以把这两种模式都关闭，就像我们现在做的这样，这种情况下片段的颜色会被直接写入帧缓冲而不加修改。

## 动态设置

我们已经设置好的那些设置项里，有很少一部分可以不重建管线而直接修改，比如视口的大小、线宽和混合常量等。如果你想进行动态设置，那么你需要像这样来填充一个[`VkPipelineDynamicStateCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineDynamicStateCreateInfo.html)结构体：

```c++
VkDynamicState dynamicStates[] = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_LINE_WIDTH
};

VkPipelineDynamicStateCreateInfo dynamicState = {};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = 2;
dynamicState.pDynamicStates = dynamicStates;
```

这会使我们之前的设置全部被忽略，然后你必须在绘制时指定这些设置项的值。在之后的章节里我们会来讨论这个问题。如果你不想执行任何动态设置，你可以一会直接传递一个`nullptr`来替代这个结构体。

## 管线布局

你可以在着色器中使用`uniform`值，这是一种类似于动态状态变量（dynamic state variable）的全局变量，你可以在绘制时更改这些变量的值来更改着色器的行为，而不需要重新创建着色器。它们通常用来向顶点着色器传递变换矩阵，或者在片段着色器中创建纹理采样器（texture sampler）。

这些uniform值需要在创建管线时通过[`VkPipelineLayout`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineLayout.html)对象指定。尽管目前几章不会用到uniform值，我们还是必须创建一个空的管线布局（pipeline layout）。

新建一个成员变量来保存这个对象，因为之后有其它函数会用到它：

```c++
VkPipelineLayout pipelineLayout;
```

然后在`createGraphicsPipeline`函数中创建对象：

```c++
VkPipelineLayoutCreateInfo pipelineLayoutInfo = {};
pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
pipelineLayoutInfo.setLayoutCount = 0; // Optional
pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optional

if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS) {
    throw std::runtime_error("failed to create pipeline layout!");
}
```

这个结构体同时指定了“推送常量”（push constant），推送常量是向着色器传递动态值的另一种方式，我们会在以后的章节中深入探讨。管线布局在程序的整个生命周期中都会被引用，因此它应该在程序末尾被销毁：

```c++
void cleanup() {
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

## 小结

到此，就完成了所有固定功能的设置！从头设置好这一切是个大工程，不过好处在于我们现在几乎完全了解了图形渲染管线中将会发生的一切！这减少了由于某个组件的默认值与期望值不符而产生意外行为的可能。

不过在最终创建图形渲染管线之前，还有一个对象需要创建，它就是渲染过程。

[C++代码](https://vulkan-tutorial.com/code/10_fixed_functions.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)