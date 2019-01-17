# 逻辑设备与队列

## 简介

在选用了一个物理设备之后，我们需要创建一个“逻辑设备”（logical device）作为它的接口。创建逻辑设备的过程与创建实例的过程很相似，并且需要描述我们想要使用的功能。同样，在查询了有哪些队列家族可用之后，我们需要在创建逻辑设备时指定创建哪些队列。如果你需要的话，甚至可以根据同一个物理设备创建多个逻辑设备。

首先，在类中添加一个新的成员变量来保存逻辑设备的句柄。

```c++
VkDevice device;
```

接下来，添加一个`createLogicalDevice`成员函数，然后在`initVulkan`函数中调用它。

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    pickPhysicalDevice();
    createLogicalDevice();
}

void createLogicalDevice() {

}
```

## 指定要创建的队列

创建逻辑设备又会涉及到许多结构体中的许多详细信息，其中的第一个是[`VkDeviceQueueCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDeviceQueueCreateInfo.html)。则个结构体描述了对于单个的队列家族，我们想要创建的队列的数量。现在我们只对支持绘图功能的队列感兴趣。

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

VkDeviceQueueCreateInfo queueCreateInfo = {};
queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
queueCreateInfo.queueCount = 1;
```

当前可用的驱动只允许你为每个队列家族创建很少的几个队列，不过事实上你也的确只需要一个队列。这是因为，你可以使用多线程创建所有命令缓冲（command buffer）然后在主线程上低开销地一次性提交它们。

Vulkan允许你使用一个介于`0.0`和`1.0`之间的浮点数来为队列分配优先级以影响命令缓冲执行的调度。就算只有一个队列，这个优先级也是必需的：

```c++
float queuePriority = 1.0f;
queueCreateInfo.pQueuePriorities = &queuePriority;
```

## 指定要使用的设备功能

下一个要指定的信息是我们要使用的设备功能。这些功能是我们在上一章用[`vkGetPhysicalDeviceFeatures`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetPhysicalDeviceFeatures.html)查询过支持性的功能，比如几何着色器。现在我们不需要做什么特别的事情，所以我们可以简单地定义它然后让所有内容都是默认的`VK_FALSE`。当我们准备用Vulkan做一些有趣的事情的时候，我们会回到这个结构体上的。

```c++
VkPhysicalDeviceFeatures deviceFeatures = {};
```

## 创建逻辑设备

上面两个结构体设置好了之后，我们就可以开始填充主要的[`VkDeviceCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkDeviceCreateInfo.html)结构体了。

```c++
VkDeviceCreateInfo createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
```

首先添加指向队列创建信息结构体和设备功能结构体的指针：

```c++
createInfo.pQueueCreateInfos = &queueCreateInfo;
createInfo.queueCreateInfoCount = 1;

createInfo.pEnabledFeatures = &deviceFeatures;
```

剩下的信息与[`VkInstanceCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstanceCreateInfo.html)结构体有相似之处，并且需要你指定扩展和验证层。不同之处在于，这次是设备特定的。

设备特定扩展的一个例子是`VK_KHR_swapchain`，它允许你通过设备把渲染好的图像显示到窗口上。有可能系统中有一些Vulkan设备不支持这个功能，例如它们只支持计算功能操作。我们将会在交换链一章讲解这个扩展。

就像验证层一章讲过的，我们会启用与实例相同的验证层。现在我们不需要任何设备特定扩展。

```c++
createInfo.enabledExtensionCount = 0;

if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

就这样，我们现在已经准备好调用[`vkCreateDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateDevice.html)函数来创建逻辑设备了。

```c++
if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS) {
    throw std::runtime_error("failed to create logical device!");
}
```

函数参数分别是，要连接的物理设备、我们刚指定过的队列和使用信息、可选的分配器回调函数指针以及指向存储着逻辑设备句柄的变量的指针。与创建实例的函数相似，这个函数在启用了不存在的扩展或指定了不支持的功能时会返回错误。

逻辑设备应该在`cleanup`函数中用[`vkDestroyDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkDestroyDevice.html)函数销毁：

```c++
void cleanup() {
    vkDestroyDevice(device, nullptr);
    ...
}
```

逻辑设备与实例没有直接关系，因此参数中没有包含实例。

## 检索队列句柄

队列会随着逻辑设备的创建而自动创建，但是我们还没有与它们相连的句柄。首先在类中添加一个新成员变量来保存图形队列的句柄：

```c++
VkQueue graphicsQueue;
```

设备队列会随着逻辑设备的销毁而被隐式销毁，所以我们不必在`cleanup`中做额外工作。

我们可以使用[`vkGetDeviceQueue`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetDeviceQueue.html)函数来在每个队列家族中检索队列句柄。这个函数的参数是逻辑设备、队列家族、队列索引以及一个指向要存储队列句柄的变量的指针。因为我们在这个队列家族中只创建了一个队列，索引就简单地设为`0`。

```c++
vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

逻辑设备和队列句柄可以让我们真正地开始用显卡做些什么东西了！在接下来的几章里，我们会创建一些资源，使得结果可以被显示在窗口上。

[C++代码](https://vulkan-tutorial.com/code/04_logical_device.cpp)