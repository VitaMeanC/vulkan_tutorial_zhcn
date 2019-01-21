# 交换链

Vulkan中没有“默认帧缓冲”的概念，于是就需要有一个“基础设施”，它拥有这么一个缓冲区，让我们在显示到屏幕上之前把图像渲染在上面。这个“基础设施”就是“交换链”（swap chain），在Vulkan中它必须被显式创建。交换链本质上是一个等待着被显示到屏幕上的图像的队列。我们的程序会获得这么一个图像来进行绘制，然后再把这个图像放回到队列中。这个队列具体如何工作，以及从队列中显示一个图像的条件取决于交换链是如何设置的，但是一般来讲，交换链的作用是让图像的显示与屏幕刷新率同步。

## 检查交换链支持性

出于各种各样的原因，不是所有显卡都能直接把图像显示到屏幕上的。例如它们是为服务器设计的而且没有显示输出。第二，由于显示图像与窗口系统和关联到窗口的表面密切相关，它实际上不是Vulkan核心的一部分。在检查支持性之后，你需要启用`VK_KHR_swapchain`这个设备扩展。

为此我们首先来扩展`isDeviceSuitable`函数来检查这个扩展是否受支持。之前我们已经知道了如何列出一个[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)中所有受支持的扩展，所以这个应该非常直观。注意，Vulkan头文件中提供了一个不错的宏：`VK_KHR_SWAPCHAIN_EXTENSION_NAME`，这个宏被定义为`VK_KHR_swapchain`。使用宏的优点在于编译器会捕获拼写错误。

首先声明一个所需的设备扩展的列表，就像是要启用的验证层的列表那样。

```c++
const std::vector<const char*> deviceExtensions = {
    VK_KHR_SWAPCHAIN_EXTENSION_NAME
};
```

接下来，创建一个新函数：`checkDeviceExtensionSupport`，在`isDeviceSuitable`中调用这个函数作为一个额外检查：

```c++
bool isDeviceSuitable(VkPhysicalDevice device) {
    QueueFamilyIndices indices = findQueueFamilies(device);

    bool extensionsSupported = checkDeviceExtensionSupport(device);

    return indices.isComplete() && extensionsSupported;
}

bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    return true;
}
```

修改函数内容，遍历所有扩展，并且检查是不是每一个需要的扩展都在里面。

```c++
bool checkDeviceExtensionSupport(VkPhysicalDevice device) {
    uint32_t extensionCount;
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);

    std::vector<VkExtensionProperties> availableExtensions(extensionCount);
    vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

    std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());

    for (const auto& extension : availableExtensions) {
        requiredExtensions.erase(extension.extensionName);
    }

    return requiredExtensions.empty();
}
```

我在此选择了用一个字符串的集合（set）来代表那些未确认的所需扩展。这样我们就可以在遍历可用扩展序列的时候轻松地剔除它们。当然你也可以用一个嵌套循环，就像`checkValidationLayerSupport`里面那个。性能差异不是大问题。现在运行代码来验证你的显卡的确支持创建交换链。需要注意的是，我们在一章检查过的显示队列的可用性，如果显示队列可用，那就意味着交换链扩展也一定受支持。然而，显式检查一遍也没什么坏处，而且这个插件还需要显式启用。

## 启用设备插件

使用交换链首先需要启用`VK_KHR_swapchain`扩展。启用这个扩展只需要对创建逻辑设备的结构体做一点小修改：

```c++
createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

## 查询交换链支持性的详细信息

仅仅是查询交换链是否可用还不够，因为它有可能与我们的表面不兼容。创建交换链同样涉及到许多设置项目，而且比创建实例和设备的设置项目多得多，所以我们在创建之前需要再查询一些详细信息。

基本有三种属性是需要检查的：

* 基本的表面兼容性（交换链支持的最小/最大图像数量、最小/最大图像宽高）
* 表面格式（像素格式、色彩空间）
* 可用的显示模式

与`findQueueFamilies`类似，我们需要创建一个结构体来保存查询到的详细信息。上述三类属性会用下列结构体与结构体列表保存：

```c++
struct SwapChainSupportDetails {
    VkSurfaceCapabilitiesKHR capabilities;
    std::vector<VkSurfaceFormatKHR> formats;
    std::vector<VkPresentModeKHR> presentModes;
};
```

我们现在来创建一个新函数：`querySwapChainSupport`来填充这个结构体。

```c++
SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device) {
    SwapChainSupportDetails details;

    return details;
}
```

这一节只包括如何查询结构体中包含的信息。这些结构体的含义以及它们所包含的每一个数据留到下一节讨论。

让我们从基本的表面兼容性开始。这些属性非常容易查询并且只返回一个`VkSurfaceCapabilitiesKHR`结构体。

```c++
vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
```

这个函数会根据给定的[`VkPhysicalDevice`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkPhysicalDevice.html)和`VkSurfaceKHR`来判断兼容性。所有的查询兼容性的函数都会以这两个作为头两个参数，因为它们是交换链的核心部分。

下一步是查询支持的表面格式。因为这是一个结构体的列表，它与下面两个函数调用方式类似：

```c++
uint32_t formatCount;
vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);

if (formatCount != 0) {
    details.formats.resize(formatCount);
    vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
}
```

确保vector被resize过，以保存所有可用的格式。然后，最后再用`vkGetPhysicalDeviceSurfacePresentModesKHR`查询可用的显示模式，这与之前的步骤相同：

```c++
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

现在，所有详细信息都保存在了结构体里。然后再来扩展一次`isDeviceSuitable`函数，用这个函数来检查交换链是否被完整地支持。对于本教程来说，有至少一种图像格式以及一种显示模式支持给定的表面就足够了。

```c++
bool swapChainAdequate = false;
if (extensionsSupported) {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
    swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
}
```

在验证插件是否可用之后再尝试查询交换链支持性的详细信息是非常重要的。把函数的最后一行改成：

```c++
return indices.isComplete() && extensionsSupported && swapChainAdequate;
```

## 为交换链选择正确的设置

如果`swapChainAdequate`为真，那么交换链无疑是被完整支持的，然而仍然可能有许多不同的最优设置模式。我们现在来写几个函数，找出最佳的可用交换链的正确设置。有三种设置需要确定：

* 表面格式（色彩深度）
* 显示模式（在屏幕上“交换”图像的条件）
* 交换范围（交换链中图像的分辨率）

对于每个设置我们都有一个理想值，如果这个理想值可用则设为理想值，否则我们会写一些逻辑来寻找次佳值。

### 表面格式

设置这一项的函数刚开始类似于这样。我们会在稍后传递`SwapChainSupportDetails`结构体中的`formats`成员作为参数。

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {

}
```

每个`VkSurfaceFormatKHR`结构体都包含一个`format`和一个`colorSpace`成员。`format`成员指定颜色的通道数和类型。例如，`VK_FORMAT_B8G8R8A8_UNORM`代表我们按照B、G、R以及alpha通道的顺序存储，美个通道是8位的无符号整型，每个像素一共32位。`colorSpace`成员使用`VK_COLOR_SPACE_SRGB_NONLINEAR_KHR`标志来指示SRGB色彩空间是否可用。注意，根据标准，旧版API中这个标志曾经叫做`VK_COLORSPACE_SRGB_NONLINEAR_KHR`。

如果SRGB色彩空间可用，我们就会使用它，因为它[产生的颜色更加真实](http://stackoverflow.com/questions/12524623/)。直接使用SRGB颜色会有一点挑战，因此我们使用标准RGB作为颜色格式，就用常见颜色格式中的`VK_FORMAT_B8G8R8A8_UNORM`。

最好的情况是，表面没有推荐格式，这时Vulkan会只返回一个`format`成员为`VK_FORMAT_UNDEFINED`的`VkSurfaceFormatKHR`结构体。

```c++
if (availableFormats.size() == 1 && availableFormats[0].format == VK_FORMAT_UNDEFINED) {
    return {VK_FORMAT_B8G8R8A8_UNORM, VK_COLOR_SPACE_SRGB_NONLINEAR_KHR};
}
```

如果我们不能自由选择格式，那么我们就遍历可用格式列表来检查最佳组合是否在可用：

```c++
for (const auto& availableFormat : availableFormats) {
    if (availableFormat.format == VK_FORMAT_B8G8R8A8_UNORM && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
        return availableFormat;
    }
}
```

如果不可用，那我们会开始给所有可用格式排序来看看它们有多“好”，不过一般来说设置为列表中的第一个格式就行了。

```c++
VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats) {
    if (availableFormats.size() == 1 && availableFormats[0].format == VK_FORMAT_UNDEFINED) {
        return {VK_FORMAT_B8G8R8A8_UNORM, VK_COLOR_SPACE_SRGB_NONLINEAR_KHR};
    }

    for (const auto& availableFormat : availableFormats) {
        if (availableFormat.format == VK_FORMAT_B8G8R8A8_UNORM && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR) {
            return availableFormat;
        }
    }

    return availableFormats[0];
}
```

### 显示模式

理论上来说，显示模式是交换链最重要的设置项，因为它决定了如何把图像显示到屏幕上。Vulkan中有四种可用的显示模式：

* `VK_PRESENT_MODE_IMMEDIATE_KHR`：图像绘制完成后会被立即显示到屏幕上，这有可能导致画面撕裂。
* `VK_PRESENT_MODE_FIFO_KHR`：此时交换链是一个队列。画面刷新时，队列中的前一张图像会被显示到屏幕上，而绘制好的图像会被放到队列的末尾。如果队列已满，程序就只能等待。这种模式与现代游戏中常用的“垂直同步”很相似。画面刷新的那一瞬间叫做“垂直空白间隙”。（双缓冲）
* `VK_PRESENT_MODE_FIFO_RELAXED_KHR`：这种模式只在上一个垂直空白间隙中绘制超时且队列为空的时候才与上一种模式有区别。当图像终于渲染好的时候，它会被直接显示到屏幕上，而不是等待下一个垂直空白间隙。这有可能导致画面撕裂。
* `VK_PRESENT_MODE_MAILBOX_KHR`：这是第二种模式的另一个变种。队列已满时，它不会阻塞程序，而是直接用新绘制的图像替代队列中的图像。这种模式可以用三缓冲来实现，比起使用双缓冲的标准垂直同步，三缓冲可以避免可能导致严重延迟问题的画面撕裂。

只有`VK_PRESENT_MODE_FIFO_KHR`模式是始终可用的，所以我们需要再写一个函数来检查最佳模式是否可用：

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR> availablePresentModes) {
    return VK_PRESENT_MODE_FIFO_KHR;
}
```

我个人认为认为三缓冲是最佳选择。它通过在垂直空白间隙到来之前渲染尽可能新的图像，从而避免画面撕裂，同时还能保持相当低的延迟。所以，让我们遍历一下列表来检查三缓冲是否可用：

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR> availablePresentModes) {
    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        }
    }

    return VK_PRESENT_MODE_FIFO_KHR;
}
```

不幸的是，一些驱动至今仍不支持`VK_PRESENT_MODE_FIFO_KHR`，所以在`VK_PRESENT_MODE_FIFO_KHR`不可用时，我们应该尝试`VK_PRESENT_MODE_IMMEDIATE_KHR`：

```c++
VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR> availablePresentModes) {
    VkPresentModeKHR bestMode = VK_PRESENT_MODE_FIFO_KHR;

    for (const auto& availablePresentMode : availablePresentModes) {
        if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR) {
            return availablePresentMode;
        } else if (availablePresentMode == VK_PRESENT_MODE_IMMEDIATE_KHR) {
            bestMode = availablePresentMode;
        }
    }

    return bestMode;
}
```

### 交换范围

这是最后一个主要设置项目了，我们来添加最后一个函数：

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {

}
```

交换范围是指交换链中图像的分辨率，而且它几乎永远等同于要绘制的窗口大小。可用分辨率的范围由`VkSurfaceCapabilitiesKHR`结构体定义。Vulkan要求我们在`currentExtent`成员中设置宽度和高度来匹配窗口分辨率。不过有一些窗口管理器允许我们把`currentExtent`成员的宽度和高度设置为一个特殊值：`uint32_t`的最大值。在这种情况下，我们会选择在`minImageExtent`与`maxImageExtent`之间最符合窗口分辨率的分辨率。

```c++
VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities) {
    if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max()) {
        return capabilities.currentExtent;
    } else {
        VkExtent2D actualExtent = {WIDTH, HEIGHT};

        actualExtent.width = std::max(capabilities.minImageExtent.width, std::min(capabilities.maxImageExtent.width, actualExtent.width));
        actualExtent.height = std::max(capabilities.minImageExtent.height, std::min(capabilities.maxImageExtent.height, actualExtent.height));

        return actualExtent;
    }
}
```

`max`和`min`函数用来把`WIDTH`和`HEIGHT`的值限制在Vulkan实现所支持的最大与最小分辨率之间。确保你已经引入了`<algorithm>`头文件来使用这两个函数。

## 创建交换链

现在我们已经创建好了所有帮助函数，来帮助我们在运行时进行那些必须要做的选择，我们已经拥有了创建一个可工作的交换链的所有信息。

创建一个`createSwapChain`函数，在这个函数中调用之前那些帮助函数来获取结果。确保这个函数在`initVulkan`中是在创建了逻辑设备之后调用的。

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
    createSurface();
    pickPhysicalDevice();
    createLogicalDevice();
    createSwapChain();
}

void createSwapChain() {
    SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

    VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
    VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
    VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);
}
```

现在还有一个小问题需要确定，不过这个问题非常简单，不必为它专门创建一个函数。这个问题就是决定交换链中的图像数量，实际上也就是队列的长度。Vulkan实现指定了正常工作所需的最小图像数量，我们会尝试给这个数量+1来正确实现三缓冲。

```c++
uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount) {
    imageCount = swapChainSupport.capabilities.maxImageCount;
}
```

把`maxImageCount`的值设为0意味着对内存占用没有限制，因此我们需要检查它的值。

就像其它Vulkan对象那样，创建交换链对象需要填充一个庞大的结构体，它们的开头都很相似：

```c++
VkSwapchainCreateInfoKHR createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
createInfo.surface = surface;
```

在指定了交换链要关联的表面之后，交换链图像的信息也需要指定：

```c++
createInfo.minImageCount = imageCount;
createInfo.imageFormat = surfaceFormat.format;
createInfo.imageColorSpace = surfaceFormat.colorSpace;
createInfo.imageExtent = extent;
createInfo.imageArrayLayers = 1;
createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT;
```

`imageArrayLayers`指定了每个图像所包含的图层的数量。除非你在开发一个立体的3D程序，它的值始终应该被设为`1`。`imageUsage`指定了我们如何使用交换链中的图像。在这个教程中我们会直接显示它们，这意味着它们应该被用作“颜色附件”（color attachment）。你也可以把交换链中的图像全都渲染到一张单独的图像上来进行其他操作，比如后续处理。在这种情况下，`imageUsage`的值应该是`VK_IMAGE_USAGE_TRANSFER_DST_BIT`，然后使用内存操作来把渲染好的图像传送给交换链。

```c++
QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
uint32_t queueFamilyIndices[] = {indices.graphicsFamily.value(), indices.presentFamily.value()};

if (indices.graphicsFamily != indices.presentFamily) {
    createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
    createInfo.queueFamilyIndexCount = 2;
    createInfo.pQueueFamilyIndices = queueFamilyIndices;
} else {
    createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
    createInfo.queueFamilyIndexCount = 0; // Optional
    createInfo.pQueueFamilyIndices = nullptr; // Optional
}
```

接下来，如果图形队列家族与显示队列不是同一个队列家族的话，我们需要指定如何处理将要在多个队列家族中使用的交换链中的图像，这也正是我们程序中的情况。我们会在图形队列中绘制交换链中的图片，然后把它们提交到显示队列中去。有两种方法来处理会在多个队列中使用的图像：

* `VK_SHARING_MODE_EXCLUSIVE`：一个图像在在同一时间只能被一个队列家族占有，并且在另一个队列家族使用图像之前必须显式转移所有权。这种方法的性能最高。
* `VK_SHARING_MODE_CONCURRENT`：图像可以被多个队列使用而不需要显式转移所有权。

如果队列家族不同，我们就会在使用concurrent（同时）模式。在此教程中使用同时模式是因为这样可以避免写有关于处理所有权的章节，因为处理所有权包含了一些需要以后再解释的概念。同时模式需要你通过`queueFamilyIndexCount`和`pQueueFamilyIndices`参数提前指定所有权将会在哪些队列家族之间共享。如果图形队列家族与显示队列家族是同一个，也就是绝大多数硬件上的情况，那么我们就使用exclusive（独占）模式，因为同时模式需要你至少指定两个不同的队列家族。

```c++
createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
```

我们可以为交换链中的图像指定一个变换（`supportedTransforms`中的`capabilities`），比如顺时针旋转90度，或者水平翻转。如果你不想做任何变换，那就简单地把它指定为当前使用的变换。

```c++
createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
```

`compositeAlpha`参数指定了alpha通道是否应该与窗口系统中的其它窗口进行混合。你基本上都会忽略掉alpha通道，那就设成`VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR`。

```c++
createInfo.presentMode = presentMode;
createInfo.clipped = VK_TRUE;
```

`presentMode`成员的含义很明确（显示模式）。如果`clipped`成员被设为了`VK_TRUE`，那就说明我们不关心被遮挡的像素的颜色，例如有另一个窗口挡住了这个窗口。除非你的确需要读回这些像素以获得一个可预测的结果，你最好开启裁剪来获得最佳性能。

```c++
createInfo.oldSwapchain = VK_NULL_HANDLE;
```

`oldSwapChain`是最后一个设置项。在Vulkan中，程序运行时，你的交换链有可能失效或者未优化，比如窗口大小被改变了。在这种情况下，交换链必须被重新创建，并且需要在这里指定旧交换链的引用。这是一个很复杂的话题，因此我们会在以后的章节（《重新创建交换链》）里再了解它。现在我们假设我们只需要创建一个交换链。

现在添加一个新的成员变量来保存`VkSwapchainKHR`对象：

```c++
VkSwapchainKHR swapChain;
```

现在，创建交换链只需要简单地调用`vkCreateSwapchainKHR`：

```c++
if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS) {
    throw std::runtime_error("failed to create swap chain!");
}
```

这个函数的参数分别是逻辑设备、交换链的创建信息，可选的自定义分配器以及一个指向要存储交换链句柄的变量的指针。一如既往地，我们需要使用`vkDestroySwapchainKHR`来在销毁逻辑设备之前销毁交换链：

```c++
void cleanup() {
    vkDestroySwapchainKHR(device, swapChain, nullptr);
    ...
}
```

现在运行程序来检查交换链是否创建成功。如果你发现`vkCreateSwapchainKHR`函数中有一个访问冲突错误，或者是看见了一个类似`Failed to find 'vkGetInstanceProcAddress' in layer SteamOverlayVulkanLayer.dll`的错误信息，可以看看《常见问题》中有关Steam overlay layer的问题。

尝试在开启验证层的同时删除`createInfo.imageExtent = extent;`这行。你会看到有一个验证层立刻捕获了错误并且给出了有用的提示信息：

![](https://vulkan-tutorial.com/images/swap_chain_validation_layer.png)

## 取出交换链中的图像

交换链现在已经创建好了，最后一个问题就是如何取出交换链中的[`VkImage`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkImage.html)句柄。在后续章节中涉及到渲染操作时我们需要图像的引用。添加一个成员变量来保存句柄：

```c++
std::vector<VkImage> swapChainImages;
```

图像由Vulkan实例为交换链创建，且会在交换链销毁时被自动销毁，因此不需要我们来写销毁代码。

我在`createSwapChain`函数的末尾添加了这些代码，就在调用了`vkCreateSwapchainKHR`函数之后。取出这些图像的句柄就像取出其它Vulkan对象的数组一样。首先使用`vkGetSwapchainImagesKHR`函数来查询交换链中图片的数量，然后resize容器，最后再调用这个函数一次来取回句柄。

```c++
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
swapChainImages.resize(imageCount);
vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
```

注意，我们在创建交换链时通过`minImageCount`传递了我们希望创建的图像的数量。Vulkan实现有可能创建更多的图像，因此我们需要再显式查询一次。

最后，把我们为交换链中的图像所选择的格式与范围保存到成员变量中去。我们会在后续章节用到这些信息。

```c++
VkSwapchainKHR swapChain;
std::vector<VkImage> swapChainImages;
VkFormat swapChainImageFormat;
VkExtent2D swapChainExtent;

...

swapChainImageFormat = surfaceFormat.format;
swapChainExtent = extent;
```

我们现在有了一些可供绘制的图像，并且还可以显示到窗口上。下一章我们会涉及到如何把图像设为渲染目标，然后我们会开始接触真正的图形渲染管线以及绘制命令！

[C++代码](https://vulkan-tutorial.com/code/06_swap_chain_creation.cpp)