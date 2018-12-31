# 开发环境

这一章我们会配置一个Vulkan应用程序并且安装一些实用的库。我们用到的所有工具，除了编译器之外都是同时兼容Windows、Linux和MacOS的，不过安装的时候可能会有一点困难，也正是因此在这一章里我们详细地讲解这些工具。

## Windows

如果你在Windows下开发，那么我假设你用的是Visual Studio 2017来编译代码。你也可以用Visual Studio 2013或者2015，不过步骤有点不一样。

### Vulkan SDK

开发Vulkan应用时最重要的组件就是Vulkan SDK。它包括了头文件、标准验证层、调试工具以及加载Vulkan函数用的一个加载器。这个加载器在运行时从显卡驱动中加载一个函数，就像是GLEW之于OpenGL那样——如果你熟悉这两者。

 你可以在[LunarG网站](https://vulkan.lunarg.com/)底部的按钮处下载这个SDK。不需要你创建账号，但是有一个账号可以让访问一些对你来说可能会有用的文档。

![](https://vulkan-tutorial.com/images/vulkan_sdk_download_buttons.png)

安装时记下SDK的安装位置。首先我们要确认一下你的显卡和驱动是否正确地支持Vulkan。进入SDK安装目录，打开`Bin`文件夹，运行`cude.exe`，你看到的应该是如下画面：

![](https://vulkan-tutorial.com/images/cube_demo.png)

如果你收到了一个错误提示，确保你的驱动是最新的，以及你的显卡的确支持Vulkan运行时。阅读简介那一章来获取主要显卡供应商的驱动下载地址。

这个文件夹里还有一个在开发时会很有用的程序。`glslangValidator.exe`程序可以把人类可读的[GLSL](https://zh.wikipedia.org/wiki/GLSL)编译成字节码。我们会在着色器模块那一章详细讲解这个程序。`Bin`文件夹中也有关于Vulkan加载器和验证层的二进制程序，然后`Lib`文件夹中是库。

`Doc`文件夹里面有很多关于VulkanSDK的实用信息，以及一个Vulkan规范的离线版本。最后，`Include`文件夹中包含了Vulkan头文件。你可以随意看看其它的文件夹里都有什么，不过在此教程中我们用不到它们。

### GLFW

就像之前提过的，Vulkan本身是一套平台无关的API，并且没有任何关于创建窗口并在上面显示渲染结果这类的工具。为了能体现出Vulkan跨平台的优点，并且避免直接操作WinAPI的恐怖，我们会用兼容Windows、Linux和MacOS的[GLFW库](http://www.glfw.org/)来创建窗口。虽然也有别的库可以用，比如[SDL](https://www.libsdl.org/)，不过GLFW的优点在于它把一些与具体平台相关的东西都抽象掉了，专注于创建窗口。

你可以在GLFW的[官网](http://www.glfw.org/download.html)找到GLFW的最新发行版。在此教程中我们使用64位版本，不过你也可以用32位版本。如果你要用32位版本，那么在链接Vulkan SDK的时候就要选择`Lib32`文件夹里的库而不是`Lib`文件夹里的。下载好之后，把它解压到一个你喜欢的地方。我放在了“文档”下的Visual Studio文件夹里面。如果没有`libvc-2017`文件夹的话，别慌，`libvc-2015`文件夹里面的也可以。

![](https://vulkan-tutorial.com/images/glfw_directory.png)

*译者注：SDL 2.0.6及以上版本加入了对Vulkan的支持，截止本文发稿时，最新的SDL版本是2.0.8，已经可以良好地支持Vulkan了。关于如何在SDL中使用Vulkan，请参阅SDL Wiki中有关于[Vulkan支持](https://wiki.libsdl.org/CategoryVulkan)的部分。*

### GLM

不同于DirectX 12，Vulkan没有自带的线性代数库，所以我们需要自己下载一个。[GLM](http://glm.g-truc.net/)是一个不错的库，对图形API友好，并且已经在OpenGL开发中广泛应用。

GLM是一个只有头文件的库，所以只需要下载[最新版本](https://github.com/g-truc/glm/releases)然后把它放在任意一个位置。你的目录结构现在应该差不多是这样的：

![](https://vulkan-tutorial.com/images/library_directory.png)

### 配置Visual Studio

现在你已经安装好了所有的依赖，我们可以为Vulkan配置一个基础的Visual Studio项目然后写一点代码来确定每个部分是否都在正常工作。

启动Visual Studio，新建一个`Windows 桌面向导`项目，起一个项目名字，然后点击“确定”。

![](https://vulkan-tutorial.com/images/vs_new_cpp_project.png)

确保你的应用类型是`控制台应用(.exe)`，我们需要一个地方输出调试信息，然后确保`空项目`是选中状态。这样可以避免Visual Studio添加样板代码。

![](https://vulkan-tutorial.com/images/vs_application_settings.png)

点击“确定”按钮来创建项目，然后添加一个C++源文件。你应该已经知道应该怎么做了，不过为了完整，此处列出相应步骤。

![](https://vulkan-tutorial.com/images/vs_new_item.png)

![](https://vulkan-tutorial.com/images/vs_new_source_file.png)

现在我们在文件中写入下列代码。现在看不懂这些代码也没关系。我们只是确定一下你能否编译并且运行Vukan应用程序。在下一章我们会重新开始的。

```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>
int main() {
    glfwInit();
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);
    
    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
    std::cout << extensionCount << " extensions supported" << std::endl;
    
    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;
    
    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
    
    glfwDestroyWindow(window);
    
    glfwTerminate();
    
    return  0;
}
```

现在配置一下项目来避免出错。打开项目属性对话框并且确保选择的是`所有配置`，因为很多设置都需要同时在`Debug`和`Release`模式下生效。

![](https://vulkan-tutorial.com/images/vs_open_project_properties.png)

![](https://vulkan-tutorial.com/images/vs_all_configs.png)

进入到`C/C++ → 常规 → 附加包含目录`然后点击下拉列表里的`<编辑...>`。

![](https://vulkan-tutorial.com/images/vs_cpp_general.png)

然后添加Vulkan、GLFW和GLM的头文件路径：

![](https://vulkan-tutorial.com/images/vs_include_dirs.png)

接下来，在`链接器 → 常规 → 附加库目录`里面设置库目录。

![](https://vulkan-tutorial.com/images/vs_link_settings.png)

然后把Vulkan和GLFW的库路径添加进去：

![](https://vulkan-tutorial.com/images/vs_link_dirs.png)

进入到`链接器 → 输入 → 附加依赖项`然后点击下拉列表里的`<编辑...>`。

![](https://vulkan-tutorial.com/images/vs_link_input.png)

输入Vulkan和GLFW的对象文件：

![](https://vulkan-tutorial.com/images/vs_dependencies.png)

最后启用C++17支持：（`C/C++ → 语言 → C++语言标准`）

![](https://vulkan-tutorial.com/images/vs_cpp17.png)

现在你可以关掉项目属性对话框了。如果你的每一步都做对了，代码中所有的高亮错误提示都应该消失了。

最后，确保你在使用64位编译器：

![](https://vulkan-tutorial.com/images/vs_build_mode.png)

按下`F5`来编译并运行程序，你应该看到一个命令提示符窗口，和一个在它上方的窗口，就像这样：

![](https://vulkan-tutorial.com/images/vs_test_window.png)

插件（extension）的数量应用是一个非0值。恭喜你，你已经准备好玩Vulkan了！

## Linux

这个部分是针对Ubuntu用户的，不过你也可以跟着这个部分编译LunarG SDK，然后把那些`apt`命令换成你自己的包管理器的。你应该已经安装好了支持现代C++的GCC（4.8及以上版本）。你还需要CMake和make。

### Vulkan SDK

开发Vulkan应用时最重要的组件就是Vulkan SDK。它包括了头文件、标准验证层、调试工具以及加载Vulkan函数用的一个加载器。这个加载器在运行时从显卡驱动中加载一个函数，就像是GLEW之于OpenGL那样——如果你熟悉这两者。

 你可以在[LunarG网站](https://vulkan.lunarg.com/)底部的按钮处下载这个SDK。不需要你创建账号，但是有一个账号可以让访问一些对你来说可能会有用的文档。

![](https://vulkan-tutorial.com/images/vulkan_sdk_download_buttons.png)

在下载目录里打开一个终端，然后解压SDK：

```bash
tar -xzf vulkansdk-linux-x86_64-xxx.tar.gz
```

这会把SDK中的所有文件夹解压到以SDK版本为名字的一个子文件夹里。把这个文件夹放在任何你喜欢的地方然后记住这个路径。在SDK的根目录，也就是有`build_examples.sh`的那个目录，打开一个终端。

这个SDK示例程序，以及一会儿你自己的程序，都需要依赖XCB库。这是一个X Window System的C接口。在Ubuntu下可以通过安装`libxcb1-dev`包来安装它。你还需要`xorg-dev`包中的通用X开发文件（generic X development files）。

```bash
sudo apt install libxcb1-dev xorg-dev
```

现在你可以通过运行如下命令，构建Vulkan SDK中的示例程序：

```bash
./build_examples.sh
```

如果编译成功了，现在应该有一个`./examples/build/vkcube`可执行文件。运行它，你会看到这样一个窗口：

![](https://vulkan-tutorial.com/images/cube_demo_nowindow.png)

如果你收到了一个错误提示，确保你的驱动是最新的，以及你的显卡的确支持Vulkan运行时。阅读简介那一章来获取主要显卡供应商的驱动下载地址。

### GLFW

就像之前提过的，Vulkan本身是一套平台无关的API，并且没有任何关于创建窗口并在上面显示渲染结果这类的工具。为了能体现出Vulkan跨平台的优点，并且避免直接操作WinAPI的恐怖，我们会用兼容Windows、Linux和MacOS的[GLFW库](http://www.glfw.org/)来创建窗口。虽然也有别的库可以用，比如[SDL](https://www.libsdl.org/)，不过GLFW的优点在于它把一些与具体平台相关的东西都抽象掉了，专注于创建窗口。

我们从源代码自己编译GLFW库，而不是安装源里的版本，因为Vulkan支持需要一个比较新的版本。你可以在GLFW的[官网](http://www.glfw.org/download.html)找到它的源代码。把它解压到任意一个目录里，然后在有`CMakeLists.txt`的文件夹里打开终端。

运行下列指令来生成makefile并编译GLFW：

```bash
cmake .
make
```


你有可能看见`Could NOT find Vulkan`这样的错误提示，不过你可以安全地忽略它。如果编译成功了，你可以运行如下指令来把GLFW安装到系统库中：

```bash
sudo make install
```

### GLM

不同于DirectX 12，Vulkan没有自带的线性代数库，所以我们需要自己下载一个。[GLM](http://glm.g-truc.net/)是一个不错的库，对图形API友好，并且已经在OpenGL开发中广泛应用。

这是一个只有头文件的库，可以通过安装`libglm-dev`包来安装。

```bash
sudo apt install libglm-dev
```

### 配置makefile项目

现在你已经安装好了所有依赖项目，我们可以为Vulkan配置一个基础的makefile项目然后写一点代码来确定每个部分是否都在正常工作。

在你喜欢的地方创建一个新文件夹，起个名字，比如`VulkanTest`。创建一个名为`main.cpp`的C++源文件并且写入如下代码。现在看不懂这些代码也没关系。我们只是确定一下你能否编译并且运行Vukan应用程序。在下一章我们会重新开始的。

```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>
int main() {
    glfwInit();
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);
    
    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
    std::cout << extensionCount << " extensions supported" << std::endl;
    
    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;
    
    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }
    
    glfwDestroyWindow(window);
    
    glfwTerminate();
    
    return  0;
}
```

接下来，我们会编写一个用来编译和运行基础Vulkan代码的makefile。创建一个新的空文件并命名为`Makefile`。我默认你已经有了一些编写makefile的基本经验，比如变量和规则是怎么工作的。如果你不懂makefile，你可以看看[这份教程](http://mrbook.org/blog/tutorials/make/)来快速入门。

首先我们定义几个变量来简化文件的其余部分。定义一个`VULKAN_SDK_PATH`变量，指向LunarG SDK的`x86_64`文件夹，比如：

```make
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64
```

把`user`换成你自己的用户名，把`x.x.x.x`换成相应的版本号。接下来，定义一个`CFLAGS`变量来指定基础的编译器选项：

```make
CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include
```

我们会使用现代C++（`-std=c++17`），然后我们需要能够定位`vulkan.h`在LunarG SDK中的位置。

类似地，定义`LDFLAGS`变量作为链接选项：

```make
LDFLAGS = -L$(VULKAN_SDK_PATH)/lib `pkg-config --static --libs glfw3` -lvulkan
```

第一个选项指明了我们要查找类似`libvulkan.so`这种库文件时，在LunarG SDK的`x86_64/lib`目录中查找。第二个组件调用`pkg-config`来自动获取编译一个GLFW程序时需要的所有链接选项。最后，`-lvulkan`会把LunarG SDK中的Vulkan函数加载器链接进去。

现在来直接指定编译`VulkanTest`的规则。确保使用tab进行缩进而不是空格：

```make
VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)
```

保存makefile并在`main.cpp`和`Makefile`的目录下运行`make`命令来验证一下这个规则是否正常工作。它应该最终生成一个`VulkanTest`可执行文件。

现在我们再来添加两条规则：`test`和`clean`，前者运行编译出的可执行文件，后者则删除这个可执行文件。

```make
.PHONY: test clean

test: VulkanTest
    ./VulkanTest

clean:
    rm -f VulkanTest
```

你会发现`make clean`工作正常，但是`make test`大概率会运行失败并报如下错误：

```text
./VulkanTest: error while loading shared libraries: libvulkan.so.1: cannot open shared object file: No such file or directory
```

这是因为`libvulkan.so`没有被安装成为一个系统库。为了解决这个问题，我们需要使用`LD_LIBRARY_PATH`环境变量来显式指定库的加载路径。

```make
test: VulkanTest
    LD_LIBRARY_PATH=$(VULKAN_SDK_PATH)/lib ./VulkanTest
```

这个程序现有在应该成功运行，并且显示出Vulkan插件（extension）的数量。这个程序在你关闭空窗口后应该退出并返回成功的返回代码（`0`）。然而，还有一个你应该设置的变量。我们会在Vulkan中使用验证层，所以你应该使用`VK_LAYER_PATH`变量来告诉Vulkan库在哪加载验证层：

```make
test: VulkanTest
    LD_LIBRARY_PATH=$(VULKAN_SDK_PATH)/lib VK_LAYER_PATH=$(VULKAN_SDK_PATH)/etc/explicit_layer.d ./VulkanTest
```

完成的makefile大概应该是这样的：

```make
VULKAN_SDK_PATH = /home/user/VulkanSDK/x.x.x.x/x86_64

CFLAGS = -std=c++17 -I$(VULKAN_SDK_PATH)/include
LDFLAGS = -L$(VULKAN_SDK_PATH)/lib `pkg-config --static --libs glfw3` -lvulkan

VulkanTest: main.cpp
    g++ $(CFLAGS) -o VulkanTest main.cpp $(LDFLAGS)

.PHONY: test clean

test: VulkanTest
    LD_LIBRARY_PATH=$(VULKAN_SDK_PATH)/lib VK_LAYER_PATH=$(VULKAN_SDK_PATH)/etc/explicit_layer.d ./VulkanTest

clean:
    rm -f VulkanTest
```

现在你可以把这个文件夹当成一个Vulkan项目的模板。复制或重命名成`HelloTriangle`之类的，然后删掉`main.cpp`里的所有代码。

在我们进行下一步之前，让我们再稍微探索一下Vulkan SDK。这里面还有一个在开发时会很有用的程序。`x86_64/bin/glslangValidator`程序可以把人类可读的[GLSL](https://zh.wikipedia.org/wiki/GLSL)编译成字节码。我们会在着色器模块那一章详细讲解这个程序。

`Doc`文件夹里面有很多关于VulkanSDK的实用信息，以及一个Vulkan规范的离线版本。最后，`Include`文件夹中包含了Vulkan头文件。你可以随意看看其它的文件夹里都有什么，不过在此教程中我们用不到它们。

现在，你已经完全准备好进行真正的探险了。

## MacOS

*译者：我没用过MacOS，万一翻译出现了偏差我是要负责的，大体上和上面两个差不多，反正MacOS也是类UNIX系统，我寻思跟Linux应该差不多（逃）*