# 物理设备与队列家族

## 选择一个物理设备

在通过一个[`VkInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstance.html)来实例化Vulkan库之后，我们在系统中需要寻找并选择一个支持我们所需功能的显卡。实际上，我们可以选择多个显卡并且同时使用它们，不过在此教程中我们将选择支持我们所需功能的所有显卡中的第一个。

我们添加一个`pickPhysicalDevice`函数，并且在`initVulkan`函数中调用它。

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    pickPhysicalDevice();
}

void pickPhysicalDevice() {

}
```

我们最终选择的显卡会被存储到一个[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)句柄中，这个句柄将成为类中的一个新成员。这个对象会随着[`VkInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstance.html)的销毁而被隐式销毁，因此我们不需要在`cleanup`函数中写新代码。

```c++
VkPhysicalDevice physicalDevice = VK_NULL_HANDLE;
```

列出显卡与列出扩展很相似，从查询显卡的数量开始：

```c++
uint32_t deviceCount = 0;
vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
```

如果有0个设备支持Vulkan，那就没法进行下一步了。

```c++
if (deviceCount == 0) {
    throw std::runtime_error("failed to find GPUs with Vulkan support!");
}
```

我们现在可以分配一个数组来保存所有的[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)句柄。

```c++
std::vector<VkPhysicalDevice> devices(deviceCount);
vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
```

现在我们需要检测每个显卡的功能来检查它们是否支持我们想要的操作，因为不是所有显卡都是相同的。为此，我们来介绍一个新函数：

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

我们会检查每个设备是否都支持加入到此函数中的必需特性。

```c++
for (const auto& device : devices) {
    if (isDeviceSuitable(device)) {
        physicalDevice = device;
        break;
    }
}

if (physicalDevice == VK_NULL_HANDLE) {
    throw std::runtime_error("failed to find a suitable GPU!");
}
```

下一节将会介绍我们要在`isDeviceSuitable`函数中检查的第一个必需特性。当我们在后面的章节里用到越来越多的Vulkan功能时，我们会扩展这个函数以包括更多的检查。

## 基本设备可用性检查

为了检测一个设备的可用性，我们可以从查询一些细节开始。我们可以用[`vkGetPhysicalDeviceProperties`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetPhysicalDeviceProperties.html)函数来查询基本的设备属性，比如名称，类型以及支持的Vulkan版本等。

```c++
VkPhysicalDeviceProperties deviceProperties;
vkGetPhysicalDeviceProperties(device, &deviceProperties);
```

[`vkGetPhysicalDeviceFeatures`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetPhysicalDeviceFeatures.html)函数可以用来检查可选的特性，例如纹理压缩。64位浮点数支持以及多视口渲染（对于VR很有用）等：

```c++
VkPhysicalDeviceFeatures deviceFeatures;
vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
```

稍后我们讨论有关设备内存以及队列家族时会查询设备的更多详细信息（请参阅下一节）。

作为一个例子，我们假设我们的应用只在支持几何着色器的独立显卡上可用。这样的话，`isDeviceSuitable`函数应该类似于这样：

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    VkPhysicalDeviceProperties deviceProperties;
    VkPhysicalDeviceFeatures deviceFeatures;
    vkGetPhysicalDeviceProperties(device, &deviceProperties);
    vkGetPhysicalDeviceFeatures(device, &deviceFeatures);

    return deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU &&
           deviceFeatures.geometryShader;
}
```

你可以采用给每个设备评分，然后选择得分最高的那个的方式来取代简单地检查设备是否支持某个特性然后选择排名第一的那个。在这种方式下，你可能偏爱一个独立显卡然后给它打一个高分，但是最终回归到了集成显卡上，因为只有它最合适。你可以用如下所示的代码来实现它：

```c++
#include <map>

...

void pickPhysicalDevice() {
    ...

    // Use an ordered map to automatically sort candidates by increasing score
    std::multimap<int, VkPhysicalDevice> candidates;

    for (const auto& device : devices) {
        int score = rateDeviceSuitability(device);
        candidates.insert(std::make_pair(score, device));
    }

    // Check if the best candidate is suitable at all
    if (candidates.rbegin()->first > 0) {
        physicalDevice = candidates.rbegin()->second;
    } else {
        throw std::runtime_error("failed to find a suitable GPU!");
    }
}

int rateDeviceSuitability(VkPhysicalDevice device) {
    ...

    int score = 0;

    // Discrete GPUs have a significant performance advantage
    if (deviceProperties.deviceType == VK_PHYSICAL_DEVICE_TYPE_DISCRETE_GPU) {
        score += 1000;
    }

    // Maximum possible size of textures affects graphics quality
    score += deviceProperties.limits.maxImageDimension2D;

    // Application can't function without geometry shaders
    if (!deviceFeatures.geometryShader) {
        return 0;
    }

    return score;
}
```

你不需要在此教程中实现这个函数，不过这可以启发你如何设计自己你自己的设备选择流程。当然，你也可以把所有可选设备的名字显示出来，让用户去选择。

因为我们刚刚开始，因此只要是支持Vulkan的显卡都可以，所以我们可以选择任何一个GPU：

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    return true;
}
```

在下一节我们会讨论我们需要检查的第一个真正的必需特性。

## 队列家族

之前已经简要介绍过，Vulkan中几乎所有的操作，从绘图上传纹理，都需要将命令提交到队列。从不同的队列家族中派生出不同类型的队列，每种队列家族只接受某些命令。例如，可能有只接受处理计算命令的队列家族，也可能有只接受有关内存传递命令的命令家族。

我们需要检查设备都支持哪些队列家族，以及它们中的哪一个支持我们想要用的命令。为此，我们需要添加一个新的`findQueueFamilies`函数来寻找我们需要的所有队列家族。现在我们只寻找一个支持图形命令的队列，不过以后我们可能会扩展这个函数来寻找更多的队列。

这个函数会返回那些支持所需特性的队列家族的索引。实现它的最佳方式是使用一个用`std::optional`来追踪索引是否存在的结构体：

```c++
struct QueueFamilyIndices {
    std::optional<uint32_t> graphicsFamily;

    bool isComplete() {
        return graphicsFamily.has_value();
    }
};
```

注意，这需要引入`<optional>`头文件。现在我们可以着手实现`findQueueFamilies`函数了：

```c++
QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device) {
    QueueFamilyIndices indices;

    ...

    return indices;
}
```

检索队列家族列表的过程正如你所愿，然后使用[`vkGetPhysicalDeviceQueueFamilyProperties`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetPhysicalDeviceQueueFamilyProperties.html)函数：

```c++
uint32_t queueFamilyCount = 0;
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);

std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());
```

[`VkQueueFamilyProperties`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkQueueFamilyProperties.html)结构体包含了一个队列家族的详细信息，包括支持的操作类型，以及该队列家族可以创建的队列数量。我们至少需要找到一个支持`VK_QUEUE_GRAPHICS_BIT`的队列家族。

```c++
int i = 0;
for (const auto& queueFamily : queueFamilies) {
    if (queueFamily.queueCount > 0 && queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) {
        indices.graphicsFamily = i;
    }

    if (indices.isComplete()) {
        break;
    }

    i++;
}
```

现在我们完成了这个很棒的队列家族查找函数，我们可以在`isDeviceSuitable`函数中用它进行检查，以确保设备可以处理我们想要使用的命令：

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    return indices.isComplete();
}
```

棒，这样我们就可以找到正确的物理设备了！下一步是创建一个逻辑设备来做为物理设备的接口。

[C++代码](https://vulkan-tutorial.com/code/03_physical_device_selection.cpp)