# 小结

我们现在可以把前几章中的那些结构体与对象全部都组合起来以创建图形渲染管线了！简要概括一下，我们现在所有的对象有这么几种：

* 着色器阶段：定义了图形渲染管线中可编程阶段的功能的着色器模块
* 固定功能部分：所有定义了管线的固定功能阶段的结构体，比如顶点装配器、光栅化和颜色绑定等
* 管线布局：所有被着色器引用的uniform变量和推送（push）常量，它们可以在绘制时被更新
* 渲染过程：被管线中的过程引用的附件及其使用方法

把这些全都组合在一起，就定义了整个图形渲染管线的功能，因此我们现在可以开始在`createGraphicsPipeline`函数的末尾填充[`VkGraphicsPipelineCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkGraphicsPipelineCreateInfo.html)结构体了。但是要在调用[`vkDestroyShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDestroyShaderModule.html)函数之前，因为这些东西会在整个创建阶段被调用。

```c++
VkGraphicsPipelineCreateInfo pipelineInfo = {};
pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
pipelineInfo.stageCount = 2;
pipelineInfo.pStages = shaderStages;
```

我们从引用[`VkPipelineShaderStageCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineShaderStageCreateInfo.html)结构体的数组开始。

```c++
pipelineInfo.pVertexInputState = &vertexInputInfo;
pipelineInfo.pInputAssemblyState = &inputAssembly;
pipelineInfo.pViewportState = &viewportState;
pipelineInfo.pRasterizationState = &rasterizer;
pipelineInfo.pMultisampleState = &multisampling;
pipelineInfo.pDepthStencilState = nullptr; // Optional
pipelineInfo.pColorBlendState = &colorBlending;
pipelineInfo.pDynamicState = nullptr; // Optional
```

然后我们会引用描述了固定功能过程的所有结构体：

```c++
pipelineInfo.layout = pipelineLayout;
```

之后引入的是管线布局，这是一个Vulkan句柄，而不是一个结构体指针。

```c++
pipelineInfo.renderPass = renderPass;
pipelineInfo.subpass = 0;
```

最后，我们引入渲染过程，以及要使用的子过程的索引。你也可以在这个管线中使用其它的渲染过程，而不是给定的这个，不过它们需要与`renderPass`相兼容。对于兼容性的要求在[这里](https://www.khronos.org/registry/vulkan/specs/1.0/html/vkspec.html#renderpass-compatibility)，不过在此教程中我们不会用到这个特性。

```c++
pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
pipelineInfo.basePipelineIndex = -1; // Optional
```

其实还有两个参数：`basePipelineHandle`和`basePipelineIndex`。Vulkan允许你基于现有的图形渲染管线来创建一个新管线。派生管线的想法来源于，当新管线的大多数功能都与一个现有管线相同时，派生管线的开销更小，而且在由同一父管线派生出来的两个子管线之间切换时也会更快。你可以通过`basePipelineHandle`指定一个现有管线的句柄，或者是将要通过`basePipelineIndex`指定的索引创建的另一个管线的引用。现在我们只有一个引用，因此我们简单地指定一个空句柄和一个非法索引。它们的值只会在[`VkGraphicsPipelineCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkGraphicsPipelineCreateInfo.html)结构体中的`flags`成员被设为`VK_PIPELINE_CREATE_DERIVATIVE_BIT`标志时才会被使用。

现在来为最后一步做准备，新建一个成员变量来保存[`VkPipeline`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipeline.html)对象：

```c++
VkPipeline graphicsPipeline;
```

最后，创建图形渲染管线：

```c++
if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS) {
    throw std::runtime_error("failed to create graphics pipeline!");
}
```

[`vkCreateGraphicsPipelines`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateGraphicsPipelines.html)函数的参数比Vulkan中其它对象创建函数的参数要多。它被设计为一次调用可以接受多个[`VkGraphicsPipelineCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkGraphicsPipelineCreateInfo.html)对象来创建多个[`VkPipeline`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipeline.html)对象的。

第二个参数，我们传递了`VK_NULL_HANDLE`的那个，引用的是一个可选的[`VkPipelineCache`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineCache.html)对象，管线缓存（pipeline cache）可以用于存储以及重用那些与[`vkCreateGraphicsPipelines`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateGraphicsPipelines.html)的多次调用有关的数据，如果缓存被保存为一个文件，甚至可以跨程序调用。这可以显著提高后续的管道创建速度。我们会在管道缓存一章详细讲解它。

图形渲染管线被所有的渲染操作所需要，因此它必须在程序的末尾被销毁：

```c++
void cleanup() {
    vkDestroyPipeline(device, graphicsPipeline, nullptr);
    vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
    ...
}
```

现在运行你的程序来确认这些辛苦的工作终于得到了回报——一次成功的图形渲染管线创建！我们越来越接近那个屏幕上的三角形了。在接下来的几章里，我们会通过交换链图片建立真正的帧缓冲并且准备渲染命令。

[C++代码](https://vulkan-tutorial.com/code/12_graphics_pipeline_complete.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)