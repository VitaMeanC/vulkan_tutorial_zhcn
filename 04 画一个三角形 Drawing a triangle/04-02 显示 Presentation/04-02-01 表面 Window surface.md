# 表面

尽管Vulkan是平台无关的API，它也不能直接与窗口系统进行交互。为了在Vulkan与窗口系统之间建立连接，以让我们把渲染结果显示到屏幕上，我们需要WSI（Window System Interface，窗口系统接口）扩展。在这一章我们会先讨论第一个扩展，`VK_KHR_surface`。它暴露一个`VkSurfaceKHR`对象，这个对象作为表面的一个抽象类型，来显示渲染好的图像（image）。在我们程序中的表面会受到我们已经用GLFW创建的窗口的支持。

`VK_KHR_surface`扩展是一个实例层面的扩展，并且我们已经启用它了，因为它被包含在了`glfwGetRequiredInstanceExtensions`返回的列表里。这个列表也包含了一些其它的WSI扩展，这些扩展会在后续章节用到。

表面需要在实例创建之后再创建，因为它可以影响物理设备的选择。之所以我们把表面的创建拖到现在才讲，是因为它属于另外一个更大的话题——渲染目标和显示，如果在基础的配置部分就讨论这个话题会打乱节奏。另一个需要注意的问题是，当你只需要离屏渲染的时候，表面就是Vulkan中一个完全可选的组件。Vulkan允许你离屏渲染而不需要hack，比如创建一个不可见的窗口（OpenGL则不行）。

## 创建表面

首先在类中的调试回调函数下面添加一个`surface`成员变量。

```c++
VkSurfaceKHR surface;
```

虽然说`VkSurfaceKHR`对象是平台无关的，但是这并不意味着创建它就不需要窗口系统的详细信息了。举个例子，在Windows平台上，它需要`HWND`和`HMODULE`句柄。因此这个扩展有一个附加的平台特定插件，Windows下叫做`VK_KHR_win32_surface`，这个插件也包含在了`glfwGetRequiredInstanceExtensions`返回的列表里。

我会演示一下如何用这个平台特定的插件在Windows上创建表面，但是教程里我们不会用这个。使用GLFW这样的库却还要用平台特定的代码实在是没什么意义。GLFW库中有一个`glfwCreateWindowSurface`函数，这个函数为我们处理了不同平台之间的差异。不过，在我们开始依赖这个函数之前，看一看这个函数里面到底是怎么工作的也是有好处的。

因为表面是一个Vulkan对象，它也自带了一个需要填充的`VkWin32SurfaceCreateInfoKHR`结构体。这个结构体中有两个重要的参数：`hwnd`和`hinstance`。它们是窗口和进程的句柄。

```c++
VkWin32SurfaceCreateInfoKHR createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_WIN32_SURFACE_CREATE_INFO_KHR;
createInfo.hwnd = glfwGetWin32Window(window);
createInfo.hinstance = GetModuleHandle(nullptr);
```

`glfwGetWin32Window`函数返回GLFW窗口的原生`HWND`句柄。`GetModuleHandle`函数返回当前进程的`HINSTANCE`句柄。

现在可以用`vkCreateWin32SurfaceKHR`来创建表面了。这个函数的参数分别是实例、表面创建信息、自定义分配器和指向要存储表面句柄的变量的指针。按理说，这是一个WSI扩展函数，不过它运用得非常广泛以至于标准Vulkan加载器都包含了它，所以不需要像其它扩展函数那样显式加载它。

```c++
if (vkCreateWin32SurfaceKHR(instance, &createInfo, nullptr, &surface) != VK_SUCCESS) {
    throw std::runtime_error("failed to create window surface!");
}
```

在Linxu等其他平台上创建表面的过程大同小异，`vkCreateXcbSurfaceKHR`接受一个XCB连接和窗口作为X11上的创建信息。

`glfwCreateWindowSurface`函数通过在不同的平台上使用不同的实现解决了这个问题。我们现在把它整合进我们的程序中。在`initVulkan`函数中，在创建实例和`setupDebugCallback`之后调用`createSurface`函数。

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createSurface() {

}
```

GLFW库中的函数只需要几个简单的参数，而不需要结构体，这使得函数看起来非常直观：

```c++
void createSurface() {
    if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS) {
        throw std::runtime_error("failed to create window surface!");
    }
}
```

参数有[`VkInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstance.html)、GLFW窗口指针、自定义分配器以及指向`VkSurfaceKHR`变量的指针。它直接将相关平台的原生函数的[`VkResult`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkResult.html)返回值传递出来。GLFW不提供销毁表面的函数，不过这个可以用原生API轻松搞定：

```c++
void cleanup() {
        ...
        vkDestroySurfaceKHR(instance, surface, nullptr);
        vkDestroyInstance(instance, nullptr);
        ...
    }
```

确保表面在实例之前被销毁。

## 查询显示支持

尽管Vulkan实现支持WSI，也不意味着系统中的每个设备都支持它。因此我们需要扩展`isDeviceSuitable`来确保设备能把图像显示到我们创建的表面上。由于显示是一个队列特定的功能，这个问题实际上是寻找一个支持把图像显示到我们创建的表面上的队列家族。

有可能支持绘制命令的队列家族与支持显示命令的队列家族不是同一个。因此，考虑到可能存在一个不同的显示队列，我们需要修改`QueueFamilyIndices`结构体：

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;
    std::optional<uint32_t> presentFamily;

    bool isComplete() {
        return graphicsFamily.has_value() && presentFamily.has_value();
    }
};
```

接下来。我们来修改`findQueueFamilies`函数来查找支持把图像显示到我们创建的表面上的队列家族。用来查找的函数是`vkGetPhysicalDeviceSurfaceSupportKHR`，它的参数是物理设备、队列家族索引和表面。在与`VK_QUEUE_GRAPHICS_BIT`相同的循环中调用它：

```c++
VkBool32 presentSupport = false;
vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
```

然后简单检查一下布尔值的值，接着保存显示队列家族的索引：

```c++
if (queueFamily.queueCount > 0 && presentSupport) {
    indices.presentFamily = i;
}
```

注意，很有可能最后得到了同一个队列家族，但是在整个程序中，我们将它们视为两个不同的独立队列。虽然你可以添加一些逻辑来显式提供一个支持在同一个队列中进行绘制和显示的物理设备来提高性能。

## 创建显示队列

剩余的工作就是修改创建逻辑设备的部分来创建显示队列并且取回[`VkQueue`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkQueue.html)句柄。添加一个成员变量来保存句柄：

```c++
VkQueue presentQueue;
```

接下来，我们需要多个[`VkDeviceQueueCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDeviceQueueCreateInfo.html)结构体来从两个队列家族创建队列。一个优雅的解决方式是创建一个集合，将所有需要要创建队列的、独立的队列家族都包含进去：

```c++
#include <set>

...

QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
std::set<uint32_t> uniqueQueueFamilies = {indices.graphicsFamily.value(), indices.presentFamily.value()};

float queuePriority = 1.0f;
for (uint32_t queueFamily : uniqueQueueFamilies) {
    VkDeviceQueueCreateInfo queueCreateInfo = {};
    queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
    queueCreateInfo.queueFamilyIndex = queueFamily;
    queueCreateInfo.queueCount = 1;
    queueCreateInfo.pQueuePriorities = &queuePriority;
    queueCreateInfos.push_back(queueCreateInfo);
}
```

然后修改[`VkDeviceCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDeviceCreateInfo.html)来指向这个vector：

```c++
createInfo.queueCreateInfoCount = static_cast<uint32_t>(queueCreateInfos.size());
createInfo.pQueueCreateInfos = queueCreateInfos.data();
```

如果两个队列家族是相同的，那我们就只需要传递一次队列家族索引。最后，调用函数来取回队列句柄：

```c++
vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
```

如果两个队列是相同的，那么两个句柄的值很有可能也是相同的。在下一章我们会看看交换链，以及它如何让我们能够在表面上显示图像。

[C++代码](https://vulkan-tutorial.com/code/05_window_surface.cpp)