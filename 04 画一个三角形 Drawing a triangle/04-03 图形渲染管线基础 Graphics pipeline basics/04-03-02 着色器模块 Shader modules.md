# 着色器模块

与早期API不同，Vulkan中使用的着色器代码是字节码格式的，而不是[GLSL](https://zh.wikipedia.org/wiki/GLSL)和[HLSL](https://zh.wikipedia.org/wiki/%E9%AB%98%E7%BA%A7%E7%9D%80%E8%89%B2%E5%99%A8%E8%AF%AD%E8%A8%80)那种人类可读的语法。这种字节码格式叫做[SPIR-V](https://www.khronos.org/spir)，并且被设计为可同时用于Vulkan与OpenCL上（它们都是Khronos API）。这是一种可被用于编写图形和计算着色器的格式，但是在此教程中我们关注那些用在Vulkan的图形渲染管线中的着色器。

使用字节码格式的优点是，GPU厂商编写的编译器在把着色器代码编译成本地代码时会简单得多。过去的事实证明，在使用人类可读的语法，比如GLSL时，某些GPU厂商对标准的解释相当灵活。如果你恰好用这么一种GPU写了一个不合标准的着色器，那么，由于语法错误，你就面临着其它厂商的显卡驱动拒绝你的代码的风险，或者更糟糕，由于编译错误，你的着色器的行为完全不同了。使用简单明了的字节码格式，比如SPIR-V，有望避免这种情况。

然而，这并不意味着我们需要手写字节码。Khronos发布了一个厂商无关的编译器，用来把GLSL编译成SPIR-V。这个编译器可以验证你的代码是否完全符合规范，并且生成一个SPIR-V二进制文件，你就可以把这个文件用在你的程序中了。你也可以把这个编译器作为一个库引入到你的程序中，以便在运行时生成SPIR-V，不过在此教程中我们不需要这么做。这个编译器已经包含在了LunarG SDK中，叫做`glslangValidator.exe`，因此不必额外下载。

GLSL是一种C风格语法的着色器语言。写在`main`函数中的程序会被每个对象调用。GLSL使用全局变量来管理输入与输出，而不是使用参数和返回值。这种语言包含了许多有助于图形编程的功能，比如内建的向量与本原矩阵，以及求叉积、求矩阵向量积和求反射向量等操作的函数。向量坐标被称为`vec`再加上一个表示其中元素数量的数字，比如一个3D坐标可以保存在`vec3`中。可以通过其中的一个成员来访问向量中的某一个分量，例如`.x`，不过也可以通过多个现有分量来创建新向量，比如`vec3(1.0, 2.0, 3.0).xy`表达式会返回一个`vec2`。向量的构造函数也可以把向量和标量合并到一起。比如一个`vec3`可以用`vec3(vec2(1.0, 2.0), 3.0)`来构造。

就像上一章提到过的，我们需要编写一个顶点着色器和一个片段着色器来在屏幕上显示一个三角形。下面两节将会包含它们的GLSL代码并且我会演示如何生成两个SPIR-V二进制文件并且把它们加载到程序中。

## 顶点着色器

顶点着色器处理每个输入进来的顶点。它需要输入每个顶点的属性，比如世界位置（position）、颜色、法线以及纹理坐标。它的输出是在裁剪坐标中的最终位置，以及需要被传递给片段着色器的属性，比如颜色和纹理坐标。这些值会在光栅化的时候被插入到片段上以获得平滑的渐变。

裁剪坐标（clip coordinate）是一个顶点着色器生成的四维向量，随后它会用它的最后一个分量除以整个向量来转换成“标准化设备坐标”（normalized device coordinate）。这些标准化设备坐标是把一种帧缓冲映射到[-1, 1]到[-1, 1]坐标系统的齐次坐标，如下图所示：

![](https://vulkan-tutorial.com/images/normalized_device_coordinates.svg)

*Framebuffer coordinates：帧缓冲坐标*

*Normalized device coordinates：标准化设备坐标*

如果你接触过计算机图形学，你应该对此很熟悉。如果你使用过OpenGL，你会发现Y坐标的符号被翻转了。Z坐标现在与Direct3D中的范围一样，是从0到1。

对于我们的第一个三角形而言，我们不需要应用任何变换，我们仅仅会直接以标准化设备坐标指定三个顶点的位置来创建如下所示的形状：

![](https://vulkan-tutorial.com/images/triangle_coordinates.svg)

我们可以通过把顶点着色器输出的裁剪坐标的最后一个分量设为`1`的方式来直接输出标准化设备坐标。这样在执行把裁剪坐标转换为标准化设备坐标的除法时将不会发生任何变化。

通常来讲，这些坐标应该保存在顶点缓冲里，但是在Vulkan中创建一个顶点缓冲并在其中填充数据很难。因此我决定把顶点缓冲推迟到在屏幕上绘制出三角形之后再讲解。在这里我们做一点不那么标准的动作：直接把坐标硬编码到顶点着色器里。代码看起来像这样：

```glsl
#version 450

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
}
```

`main`函数会被每个顶点调用。内建的`gl_VertexIndex`变量包含了当前顶点的索引。这个索引通常是顶点缓冲的索引，但是在这里它是我们硬编码的顶点数据数组的索引。每个顶点的位置从着色器的常量数组中读出，并且与虚拟的`z`和`w`分量组成一个裁剪坐标中的位置。内建的`gl_Position`变量作为输出使用。

## 片段着色器

由顶点着色器中的坐标组成的三角形使用片段在屏幕上填充了一块区域。片段着色器会被这些片段调用来为帧缓冲生成颜色和深度数据。一个简单地给整个三角形输出红色的片段着色器大概是这样的：

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

`main`函数会被每个片段调用，就像顶点着色器的`main`函数会被每个顶点调用一样。GLSL中的颜色是有四个分量的向量，四个分量分别是R、G、B以及alpha通道，值在[0, 1]之间。与顶点着色器中的`gl_Position`不同，没有为当前片段输出颜色的内建变量。你必须为每个帧缓冲指定你自己的输出变量，其中`layout(location = 0)`修饰符指定帧缓冲的索引。红色会被写入到`outColor`变量中，这个变量会被连接到索引为0的第一个（也是唯一一个）帧缓冲上。

## 每个顶点的颜色

把整个三角形都设置为红色不好玩，下面这种是不是更好一点？

![](https://vulkan-tutorial.com/images/triangle_coordinates_colors.png)

我们需要对两个着色器做一系列改动才能实现这个。首先，我们需要为每个顶点指定一个不同的颜色。顶点着色器现在应该包含一个颜色数组，就像位置数组那样：

```glsl
vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);
```

现在我们只需要把每个顶点的颜色传递给片段着色器，这样它就能把这些颜色插值到帧缓冲中去了。在顶点着色器中添加一个输出，然后把它写到`main`函数里：

```glsl
layout(location = 0) out vec3 fragColor;

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

接下来，我们在片段着色器中添加一个匹配的输入：

```glsl
layout(location = 0) in vec3 fragColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

输入变量不一定必须要用与输出相同的名字，它们之间是通过`location`指令指定的索引相关联的。`main`函数被修改成了输出加上一个alpha值的颜色。如上图所示，`fragColor`的值会被自动插值到三个顶点之间的片段中，以得到一个平滑的渐变。

## 编译着色器

在项目根目录创建一个`shaders`文件夹，然后把顶点着色器保存为`shader.vert`，把片段着色器保存为`shader.frag`，放在这个文件夹里。GLSL着色器没有官方扩展名，不过这两个扩展名比较常用。

`shader.vert`的内容应该是：

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
    vec2(0.0, -0.5),
    vec2(0.5, 0.5),
    vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
    vec3(1.0, 0.0, 0.0),
    vec3(0.0, 1.0, 0.0),
    vec3(0.0, 0.0, 1.0)
);

void main() {
    gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
    fragColor = colors[gl_VertexIndex];
}
```

然后`shader.frag`的内容应该是：

```glsl
#version 450
#extension GL_ARB_separate_shader_objects : enable

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main() {
    outColor = vec4(fragColor, 1.0);
}
```

我们现在要使用`glslangValidator`程序把它们编译成SPIR-V字节码。

Windows：

用以下内容创建一个`compile.bat`文件：

```bash
C:/VulkanSDK/x.x.x.x/Bin32/glslangValidator.exe -V shader.vert
C:/VulkanSDK/x.x.x.x/Bin32/glslangValidator.exe -V shader.frag
pause
```

把其中`glslangValidator.exe`的路径替换成你自己安装Vulkan SDK的路径，双击文件来运行它。

Linux：

用以下内容创建一个`compile.sh`文件：

```bash
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslangValidator -V shader.vert
/home/user/VulkanSDK/x.x.x.x/x86_64/bin/glslangValidator -V shader.frag
```

把其中`glslangValidator`的路径替换成你自己安装Vulkan SDK的路径。使用`chmod +x compile.sh`把脚本变成可执行的，然后运行它。

不同平台之间的差异到此结束。

这两条调用编译器的命令都有`-V`选项，这告知编译器把GLSL源文件编译成SPIR-V字节码。当你执行了交易脚本，你会看到创建了两个SPIR-V二进制文件：`vert.spv`和`frag.spv`。文件名被自动地根据着色器的类型进行了命名，不过你也可以改成任何你喜欢的名字。在编译着色器的时候，你可能会看到一条丢失某些功能（some missing features）的错误，不过你可以安全地无视它们。

如果你的着色器中有语法错误，那么编译器会告诉你行号和错误，就像你希望的那样。比如，尝试去掉一个分号然后再执行一次编译脚本。或者不带任何选项地运行一次编译器来看看它都支持什么选项。例如，它还能把字节码还原成人类可读的形式，这样你就可以看到你的着色器到底是怎么工作的，以及在这一步进行了哪些优化。

使用命令行来编译着色器是一个很直观的选项，这也是我们在教程中使用的方法，不过也可以直接用你自己的代码来编译着色器。Vulkan SDK包含了[libshaderc](https://github.com/google/shaderc)，这是一个可以在程序中把GLSL编译成SPIR-V的库。

## 加载着色器

我们已经生成了SPIR-V着色器，现在是时候把它们加载到程序里，然后把它们插入到图形渲染管线的某一阶段了。首先我们会写一个简单的帮助函数来把二进制数据从文件中加载出来。

```c++
#include <fstream>

...

static std::vector<char> readFile(const std::string& filename) {
    std::ifstream file(filename, std::ios::ate | std::ios::binary);

    if (!file.is_open()) {
        throw std::runtime_error("failed to open file!");
    }
}
```

`readFile`函数会把指定文件中所有的字节读出来，然后返回到一个由`std::vector`管理的字节数组中。我们从带有两个选项的读取文件开始：

* `ate`：从文件末尾开始读取
* `binary`：以二进制形式读取文件（以避免文本转换）

从文件末尾开始读取的好处是我们可以通过读取文件位置来获取文件的大小，然后分配缓存：

```c++
size_t fileSize = (size_t) file.tellg();
std::vector<char> buffer(fileSize);
```

然后，我们就可以把文件指针（seek）移回文件开头来一次性读取所有字节了：

```c++
file.seekg(0);
file.read(buffer.data(), fileSize);
```

最后关闭文件然后返回字节：

```c++
file.close();

return buffer;
```

我们现在可以在`createGraphicsPipeline`函数中调用这个函数来从两个着色器中读取字节码了：

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");
}
```

通过输出缓存的大小并且检查其是否与文件的实际大小相匹配来确保着色器被正确地加载了。注意，代码不一定必须是C风格字符串（以`\0`结尾），因为它是二进制代码，并且稍后我们会确定它的大小。

## 创建着色器模块

在我们能够把代码传递给图形渲染管线之前，我们必须用[`VkShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModule.html)对象来包裹它。创建一个`createShaderModule`帮助函数来实现这一点。

```c++
VkShaderModule createShaderModule(const std::vector<char>& code) {

}
```

这个函数将会以参数形式接受一个字节码缓存，然后根据缓存创建一个[`VkShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModule.html)。

创建着色器模块非常简单，我们只需要指定一个指向字节码缓存的指针，以及缓存的长度就可以了。这些信息由一个[`VkShaderModuleCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModuleCreateInfo.html)结构体指定。有一个问题是，字节码代码的单位是字节，但是字节码指针是一个`uint32_t`指针，而不是`char`指针。因此我们需要用`reinterpret_cast`来转换指针，就像下面代码所示一样。执行这样的转换时，还需要保证数据满足`uint32_t`的对齐要求。幸运的是，数据保存在`std::vector`中，它的默认分配器已经保证了数据能够满足任何情况的对齐要求。

```c++
VkShaderModuleCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
createInfo.codeSize = code.size();
createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());
```

现在[`VkShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkShaderModule.html)可以通过调用[`vkCreateShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateShaderModule.html)来创建了：

```c++
VkShaderModule shaderModule;
if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS) {
    throw std::runtime_error("failed to create shader module!");
}
```

参数与之前那些对象创建函数相同：逻辑设备、指向创建信息结构体的指针、可选的分配器指针以及指向保存句柄的变量的指针。在创建了着色器模块后，代码缓存可以被立即释放了。别忘了返回着色器模块：

```c++
return shaderModule;
```

着色器模块只是对我们之前加载的着色器字节码以及其中定义的函数的一个简单包裹。在图形渲染管线创建之前，不会对SPIR-V字节码进行编译和链接以供GPU执行。这就是说，当图形渲染管线创建好之后，我们可以再次销毁着色器模块，这就是为什么我们把这些变量设置为`createGraphicsPipeline`函数中的局部变量，而不是类的成员变量的原因：

```c++
void createGraphicsPipeline() {
    auto vertShaderCode = readFile("shaders/vert.spv");
    auto fragShaderCode = readFile("shaders/frag.spv");

    VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
    VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);
```

着色器模块的销毁应该在函数结尾处调用两个[`vkDestroyShaderModule`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDestroyShaderModule.html)函数。本章中所有剩下的代码都会在这两行之前插入：

```c++
   ...
    vkDestroyShaderModule(device, fragShaderModule, nullptr);
    vkDestroyShaderModule(device, vertShaderModule, nullptr);
}
```

## 创建着色器阶段

要使用着色器，我们需要通过[`VkPipelineShaderStageCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPipelineShaderStageCreateInfo.html)结构体把它们分配到图形渲染管线上的某一阶段，作为管线创建过程的一部分。

我们首先来在`createGraphicsPipeline`函数中填充顶点着色器的结构。

```c++
VkPipelineShaderStageCreateInfo vertShaderStageInfo = {};
vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT;
```

除了必需的`sType`成员以外，第一步是要告诉Vulkan这个着色器将在哪一阶段使用。一个枚举值包括了上一章提到的所有可编程阶段。

```c++
vertShaderStageInfo.module = vertShaderModule;
vertShaderStageInfo.pName = "main";
```

接下来两个成员指定包含了代码的着色器模块，以及要调用的、叫做“入口点”的函数。这意味着你可以把多个片段着色器放到一个着色器模块中，然后通过使用不同的入口点来切换它们的行为。不过这里我们用标准的`main`。

还有一个可选的成员，`pSpecializationInfo`，我们在这里不用它，但是它有讲解一下的价值。它允许你为着色器常量指定值。通过它，你可以使用一个在管线创建阶段通过不同常量值改变配置的着色器模块。这比起在渲染时通过变量改变配置的效率更高，因为编译器可以-进行优化，比如删除依赖于这些常量值的`if`语句。如果你没有设置常量值，你可以把这个成员设为`nullptr`，我们的结构体会自动进行初始化。

把结构体修改成适合片段着色器的也很简单：

```c++
VkPipelineShaderStageCreateInfo fragShaderStageInfo = {};
fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT;
fragShaderStageInfo.module = fragShaderModule;
fragShaderStageInfo.pName = "main";
```

最后，定义一个数组来包含这两个皆否提，过一会儿真正来创建管线额时候会用到它们的引用。

```c++
VkPipelineShaderStageCreateInfo shaderStages[] = {vertShaderStageInfo, fragShaderStageInfo};
```

到此为止，管线中的可编程部分就描述完了。下一章我们会来看看固定功能部分。

[C++代码](https://vulkan-tutorial.com/code/09_shader_modules.cpp)/[顶点着色器](https://vulkan-tutorial.com/code/09_shader_base.vert)/[片段着色器](https://vulkan-tutorial.com/code/09_shader_base.frag)