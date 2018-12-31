# 基本代码

## 通用结构

在上一章你已经使用正确的设置创建了一个Vulkan项目，并且已经用一些简单的代码测试过了。在这一章我们会用下面的代码重新开始：

```cpp
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <functional>
#include <cstdlib>

class HelloTriangleApplication {
public:
    void run() { 
        initVulkan();
        mainLoop(); 
        cleanup();
    } 
private: 
    void initVulkan() {
    
    }
    
    void mainLoop() {
    
    }
    
    void cleanup() {
    
    }
}; 

int main() {
    HelloTriangleApplication app;
    
    try {
        app.run();
    } catch (const  std::exception& e) {
        std::cerr << e.what() << std::endl;
        return EXIT_FAILURE;
    } 
    
    return EXIT_SUCCESS;
}
```

首先我们从LunarG SDK中引入Vulkan的头文件，这个头文件提供了函数、结构体和枚举类型。`stdexcept`和`iostream`头文件用来报告和输出错误。`functional`头文件为资源管理部分提供lambda函数支持，`cstdlib`头文件提供`EXIT_SUCCESS`和`EXIT_FAILURE`宏定义。

这个程序本身被包装在了一个类里，我们把Vulkan对象存储成这个类的私有成员，并且添加成员函数来初始化它们，这些成员函数会被`initVulkan`函数调用。当准备工作都做好了之后，我们进入主循环开始渲染每一帧。我们会用一个一直循环到窗口被关闭为止的的循环来填充`mainLoop`函数。一旦窗口被关闭，`mainLoop`返回，我们要确定我们用过的每一个资源都会被`cleanup`函数释放。

如果在运行过程中有发生了任何致命错误，我们会抛出一个`std::runtime_error`异常并给出一个异常描述信息，这个异常描述信息会被传递到`main`函数，然后被输出到命令提示符上。虽然我们要处理各种各样的标准异常，但是我们捕获更为一般的`std::exception`。很快就会有一个关于错误处理的例子，我们会检查我们需要的扩展是否受支持。

大概此后的每一章我们都会添加一个会被`initVulkan`函数调用的新函数，而且每个成为私有成员的新Vulkan对象都必须在程序末尾通过`cleanup`函数释放。

## 资源管理

就像通过`malloc`申请到的每一块内存都必须通过`free`函数释放一样，每个Vulkan对象在当我们不需要它的时候都需要被显式销毁。在现代C++中，`<memory>`头文件提供了自动管理资源的功能，但是在此教程中，我选择显式地分配和回收Vulkan对象。毕竟Vulkan的卖点就在于显式地进行每一个操作从而避免出错，所以最好明确对象的生命周期来学习API如何工作。

在你跟着此教程做了一遍之后，你可以通过例如重载`std::shared_ptr`的方式进行自动资源管理。在大型Vulkan程序中，使用RAII是推荐方法，但是为了学习目的，知道这些东西背后的具体操作总是好的。

Vulkan对象要么是直接用形如[`vkCreateXXX`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateXXX.html)的函数直接创建的，要么是通过形如[`vkAllocateXXX`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkAllocateXXX.html)的函数从另一个对象分配的。当你确定一个对象不再被任何地方所使用的时候。你需要使用相应的[`vkDestroyXXX`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDestroyXXX.html)和[`vkFreeXXX`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkFreeXXX.html)来销毁它。这些函数的参数通常因对象的类型不同而不同，不过有一个参数是它们公有的：`pAllocator`。这是一个可选的参数，允许你为自定义的内存分配器指定回调函数。在此教程中我们将忽略这个参数并一直传一个`nullptr`作为参数。

## 整合GLFW

如果你只想离屏渲染的话，Vulkan在不创建窗口的情况下也能工作良好，但是事实上显示出点什么东西会更让人兴奋！首先删掉`#include <vulkan/vulkan.h>`这一行，换成：

```cpp
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

_译者注：如果你在使用SDL2，则不能移除Vulkan本身的头文件。_

这样，GLFW会使用它自己的定义并且自动加载Vulkan头文件。添加一个`initWindow`函数并且在`run`函数中第一个调用它。我们会用这个函数初始化GLFW并创建一个窗口。

```cpp
void run() {
    initWindow();
    initVulkan(); 
    mainLoop(); 
    cleanup(); 
} 

private: 
    void initWindow() { 
    
    }
```

在`initWindow`函数中第一个调用的应该是`glfwInit()`，这个函数初始化GLFW库。因为GLFW原本是为创建OpenGL上下文（context）设计的，所以我们接下来需要调用函数告诉GLFW不要创建OpenGL上下文：

```cpp
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

允许窗口调整大小会产生许多额外的问题，这一点日后再谈，现在先通过调用另一个window hint函数禁用掉：

```cpp
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

现在可以创建真正的窗口了。添加一个`GLFWwindow* window;`私有成员变量来保存一个GLFW窗口的引用，然后用以下函数初始化它：

```cpp
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

前三个参数知名了窗口的长度、宽度和标题。第四个参数是可选的，允许你指定一个显示器来显示这个窗口。最后一个参数只与OpenGL有关。

比起硬编码，使用常量来表示长度和宽度显然更好，因为一会儿我们还要用到这些值好几次。我在`HelloTriangleApplication`类的定义里加入了如下几行：

```cpp
const  int WIDTH = 800;
const  int HEIGHT = 600;
```

然后把创建窗口的函数改成这样

```cpp
window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
```

现在`initWindow`函数看起来应该长这样：

```cpp
void initWindow() {
    glfwInit();
    
    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
    
    window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
}
```

为了能让这个程序在不发生错误或者关闭窗口的情况下一直运行下去，我们需要在`mainLoop`函数中添加一个如下所示的事件循环：

```cpp
void mainLoop() {
    while (!glfwWindowShouldClose(window)) {
      glfwPollEvents();
    }
}
```

这段代码简直不用做任何解释。它是一个循环，每次循环都会检查事件，比如X按钮有没有被按下，一直循环到窗口被用户关闭为止。我们过一会儿还要在这个循环里调用绘制单个帧的函数。

一旦窗口被关闭，我们需要通过销毁资源并退出GLFW的方式把资源清理干净。这就是我们的第一个`cleanup`代码：

```cpp
void cleanup() {
    glfwDestroyWindow(window);
    glfwTerminate();
}
```

现在运行这个程序，你应该会看到一个标题为`Vulkan`的窗口，除非你把它关掉，否则它会一直显示着。现在我们有了一个Vulkan程序的骨架，让我们创建第一个Vulkan对象吧！

[C++代码](https://vulkan-tutorial.com/code/00_base_code.cpp)