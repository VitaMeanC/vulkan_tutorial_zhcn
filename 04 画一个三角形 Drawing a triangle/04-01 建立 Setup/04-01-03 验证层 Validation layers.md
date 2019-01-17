# 验证层

## 验证层是什么？

Vulkan API是基于最小化驱动负担的思想设计的，这个目标的一个体现形式就是，在默认情况下，这套API中的错误检查十分受限。哪怕是一点小问题，比如枚举值传错了，或者在必需参数上传了一个空指针，通常都没办法被显式处理，并且很容易导致崩溃或者未定义行为。由于Vulkan要求你在使用时显式设置每样东西，就会很容易导致许多小毛病的发生：比如使用了新的GPU特性却没有在创建逻辑设备的时候请求它。

然而，这并不意味着不能给这套API加上错误检查。Vulkan使用了一个非常优雅的系统来进行错误检查，这就是“验证层”（validation layers）。验证层是一些连接在Vulkan函数上的可选组件，用来进行一些额外的操作。一般来说，验证层有以下用途：
* 根据规范检测参数值，以避免误用
* 追踪对象的创建和析构过程，以发现资源泄露
* 从一个线程被调用的源头追踪线程，以检查线程安全性
* 把每个调用及其参数都记录在标准输出上
* 追踪Vulkan函数的调用，以进行性能分析和重放

以下是诊断验证层（diagnostics validation layer）的一个函数实例：

```cpp
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}

```

这些验证层可以自由地组合起来，以实现你感兴趣的所有调试功能。你可以简单地在调试时开启验证层，然后在发布时彻底关掉验证层，这样两全其美。

Vulkan没有任何内置的验证层，但是LunarG Vulkan SDK提供了一套验证层来检查普通的错误。这些验证层是完全[开源](https://github.com/KhronosGroup/Vulkan-ValidationLayers)的，所以你可以看到它们都检查和贡献的错误类型。使用验证层是避免你的应用程序因不小心依赖未定义行为而在不同的驱动上崩溃的最佳方式。

验证层只有在安装在系统上之后才能使用。举个例子，LunarG验证层只能在装了Vulkan SDK的电脑上使用。

Vulkan中曾经有两种不同类型的验证层：实例验证层和基于特定设备的验证层。这种想法是实例层只检查与全局Vulkan对象有关的调用，比如实例；而基于特定设备的验证层则只检查与某种特定GPU有关的调用。基于特定设备的验证层现在已经被弃用，这意味着实例验证层可以作用于所有Vulkan调用。规范文档仍然推荐你同时在设备层面启用验证层以提高兼容性，这是某些实现所需要的。我们将简单地在逻辑设备层面启用一些和实例层面相同的验证层，我们一会儿会讨论这个。

## 使用验证层

在这一节我们会看看如何启用一个Vulkan SDK提供的标准诊断层。和扩展一样，验证层也需要通过指定名字的方式启用。不需要显式指定所有可用的层，SDK允许你请求`VK_LAYER_LUNARG_standard_validation`层来隐式启用大部分可用的诊断层。

首先在程序里加两个配置变量来指定要启用的层，以及是否启用它们。我选择了让这个值基于调试模式是否开启。`NDEBUG`宏是C++标准的一部分，代表着“没有进行调试”。

```c++
const int WIDTH = 800;
const int HEIGHT = 600;

const std::vector<const char*> validationLayers = {
    "VK_LAYER_LUNARG_standard_validation"
};

#ifdef NDEBUG
    const bool enableValidationLayers = false;
#else
    const bool enableValidationLayers = true;
#endif
```

我们添加了一个新函数`checkValidationLayerSupport`来检查是否所有被请求的层都可用。首先使用[`vkEnumerateInstanceLayerProperties`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkEnumerateInstanceLayerProperties.html)函数列出所有可用的层。这个函数的使用方式与之前讲解创建实例的时候使用的[`vkEnumerateInstanceExtensionProperties`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkEnumerateInstanceExtensionProperties.html)函数相同。

```c++
bool checkValidationLayerSupport() {
    uint32_t layerCount;
    vkEnumerateInstanceLayerProperties(&layerCount, nullptr);

    std::vector<VkLayerProperties> availableLayers(layerCount);
    vkEnumerateInstanceLayerProperties(&layerCount, availableLayers.data());

    return false;
}
```

接下来，检查`validationLayers`中的层是否都存在于`availableLayers`列表里。你可能需要引入`<cstring>`头文件来使用`strcmp`。

```c++
for (const char* layerName : validationLayers) {
    bool layerFound = false;

    for (const auto& layerProperties : availableLayers) {
        if (strcmp(layerName, layerProperties.layerName) == 0) {
            layerFound = true;
            break;
        }
    }

    if (!layerFound) {
        return false;
    }
}

return true;
```

我们现在可以在`createInstance`中使用这个函数了：

```c++
void createInstance() {
    if (enableValidationLayers && !checkValidationLayerSupport()) {
        throw std::runtime_error("validation layers requested, but not available!");
    }

    ...
}
```

现在在调试模式下运行这个程序并且确保没有任何错误。如果出错了，确认一下你是否正确安装了Vulkan SDK。如果只报告了几个或者根本没有可用的层，那么你可能需要处理一下[这个问题](https://vulkan.lunarg.com/app/issues/578e8c8d5698c020d71580fc)（需要一个LunarG 账号来查看）。查看这个页面来获得解决错误的帮助。

最后，修改[`VkInstanceCreateInfo`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/VkInstanceCreateInfo.html)结构体实例来引入可用的验证层名字：

```c++
if (enableValidationLayers) {
    createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
    createInfo.ppEnabledLayerNames = validationLayers.data();
} else {
    createInfo.enabledLayerCount = 0;
}
```

如果检查成功了，那么[`vkCreateInstance`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkCreateInstance.html)不应该返回`VK_ERROR_LAYER_NOT_PRESENT`错误，不过你应该运行一下程序来保证。

## 信息回调函数

不幸的是，仅仅启用验证层的话没什么帮助，因为它们现在没有办法把错误信息发送回我们的程序。为了接收这些信息，我们需要设置一个回调函数，这需要`VK_EXT_debug_utils`插件。

首先我们创建一个`getRequiredExtensions`函数，这个函数将根据启用的验证层返回我们需要的插件列表：

```c++
std::vector<const char*> getRequiredExtensions() {
    uint32_t glfwExtensionCount = 0;
    const char** glfwExtensions;
    glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

    std::vector<const char*> extensions(glfwExtensions, glfwExtensions + glfwExtensionCount);

    if (enableValidationLayers) {
        extensions.push_back(VK_EXT_DEBUG_UTILS_EXTENSION_NAME);
    }

    return extensions;
}
```

由GLFW指定的插件通常来说都是必需的，不过调试报告插件是有条件地加入的。注意一下此处我使用了`VK_EXT_DEBUG_UTILS_EXTENSION_NAME`宏，它等同于字面字符串"VK_EXT_debug_utils"。使用这个宏可以让你避免打错字。

现在我们可以在`createInstance`里面使用这个函数了：

```c++
auto extensions = getRequiredExtensions();
createInfo.enabledExtensionCount = static_cast<uint32_t>(extensions.size());
createInfo.ppEnabledExtensionNames = extensions.data();
```

运行程序并且确保没有收到`VK_ERROR_EXTENSION_NOT_PRESENT`错误。我们实际上不用检查这个插件是否存在，因为验证层可用的话，它就是存在的。

现在我们来看看回调函数应该长什么样。加入一个名为`debugCallback`的新的静态成员函数并且使用`PFN_vkDebugUtilsMessengerCallbackEXT`函数原型。`VKAPI_ATTR`和`VKAPI_CALL` 确保了这个函数拥有正确的修饰符，以使Vulkan能够调用它。

```c++
static VKAPI_ATTR VkBool32 VKAPI_CALL debugCallback(
    VkDebugUtilsMessageSeverityFlagBitsEXT messageSeverity,
    VkDebugUtilsMessageTypeFlagsEXT messageType,
    const VkDebugUtilsMessengerCallbackDataEXT* pCallbackData,
    void* pUserData) {

    std::cerr << "validation layer: " << pCallbackData->pMessage << std::endl;

    return VK_FALSE;
}
```

第一个参数指明了消息的严重性，其值是下列值之一：

* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT`：诊断消息
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`：信息性消息，例如一个资源被创建
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT`：有关此消息的行为不一定是一个错误，但很有可能是应用程序中的一个bug（警告）
* `VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT`：有关此消息的行为是非法的，并且可能导致程序崩溃（错误）

这个枚举类型中的值是按照如上方式设置的，所以可以使用一个比较操作来检查一条消息是否与某个严重性相等或更加严重，例如：

```c++
if (messageSeverity >= VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT) {
    // Message is important enough to show
}
```

`messageType`参数可以是以下值：

- `VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT`：发生了一个与规范或性能无关的事件
- `VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT`：发生了违反规范的行为或者有可能发生的错误
- `VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT`：潜在的Vulkan非最佳使用方式

`pCallbackData`参数是一个`VkDebugUtilsMessengerCallbackDataEXT`类型的结构体，其中包含了这个信息的细节，其中最重要的成员有：

- `pMessage`：调试信息，是一个没有终止符的字符串
- `pObjects`：有关此消息的Vulkan对象句柄数组
- `objectCount`：数组中的对象数量

最后，`pUserData`参数包含了一个在设置回调函数时指定的指针，允许你传入自己的数据。

回调函数返回一个布尔值指示当验证层消息被Vulkan函数调用触发时是否应该退出程序。如果回调函数返回了真值，这个调用就会以`VK_ERROR_VALIDATION_FAILED_EXT`错误退出。这通常只用于测试验证层本身，因此你应该始终返回`VK_FALSE`。

现在只剩下告知Vulkan有关这个回调函数的信息。说起来或许会有些令人惊讶，就连Vulkan中的调试回调函数也由一个需要显式创建和销毁的句柄来管理。这种回调函数被称为“messenger”，并且你可以根据需要想设置多少个就设置多少个。在`instance`下方添加一个类成员来保存这个句柄：

```c++
VkDebugUtilsMessengerEXT callback;
```

现在添加一个`setupDebugCallback`函数，然后在`initVulkan`中的`createInstance`之后调用：

```c++
void initVulkan() {
    createInstance();
    setupDebugCallback();
}

void setupDebugCallback() {
    if (!enableValidationLayers) return;

}
```

我们现在需要用这个回调函数的细节来填充一个结构体：

```c++
VkDebugUtilsMessengerCreateInfoEXT createInfo = {};
createInfo.sType = VK_STRUCTURE_TYPE_DEBUG_UTILS_MESSENGER_CREATE_INFO_EXT;
createInfo.messageSeverity = VK_DEBUG_UTILS_MESSAGE_SEVERITY_VERBOSE_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_WARNING_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_SEVERITY_ERROR_BIT_EXT;
createInfo.messageType = VK_DEBUG_UTILS_MESSAGE_TYPE_GENERAL_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_VALIDATION_BIT_EXT | VK_DEBUG_UTILS_MESSAGE_TYPE_PERFORMANCE_BIT_EXT;
createInfo.pfnUserCallback = debugCallback;
createInfo.pUserData = nullptr; // Optional
```

`messageSeverity`字段允许你指定你的回调函数在何种严重等级下被触发。我在此指定了除`VK_DEBUG_UTILS_MESSAGE_SEVERITY_INFO_BIT_EXT`以外的所有等级来接收所有可能的错误信息并忽略一般的调试信息。

类似地，`messageType`字段允许你过滤回调函数的消息类型。我在这里简单地开启了所有类型，你可以关闭那些对你来说没什么用的。

最后，`pfnUserCallback`指定了回调函数的指针。你可以给`pUserData`传递一个指针，这个指针会通过`pUserData`参数传递到回调函数中。比如你可以用这个来传递`HelloTriangleApplication`类的指针。

注意，配置验证层消息和调试回调函数还有很多不同的方法，不过这里给出的是一个很适合入门的方法。关于其它方法，参阅[extension specification](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/vkspec.html#VK_EXT_debug_utils)（扩展规范）以获取更多信息。

这个结构体应该被传递到`vkCreateDebugUtilsMessengerEXT`函数中来创建`VkDebugUtilsMessengerEXT`对象。不幸的是，因为这个函数是一个扩展函数，所以它不会被自动加载。我们必须自己用[`vkGetInstanceProcAddr`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetInstanceProcAddr.html)函数来查找它的地址。我们要创建一个我们自己的代理函数，帮助我们在后天完成这一切。我在`HelloTriangleApplication`类定义的上面添加了这个函数：

```c++
VkResult CreateDebugUtilsMessengerEXT(VkInstance instance, const VkDebugUtilsMessengerCreateInfoEXT* pCreateInfo, const VkAllocationCallbacks* pAllocator, VkDebugUtilsMessengerEXT* pCallback) {
    auto func = (PFN_vkCreateDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkCreateDebugUtilsMessengerEXT");
    if (func != nullptr) {
        return func(instance, pCreateInfo, pAllocator, pCallback);
    } else {
        return VK_ERROR_EXTENSION_NOT_PRESENT;
    }
}
```

如果这个函数没有被加载，[`vkGetInstanceProcAddr`](https://www.khronos.org/registry/vulkan/specs/1.0/man/html/vkGetInstanceProcAddr.html)函数则返回`nullptr`。现在，如果该函数可用，我们就可以调用这个函数来创建这个扩展对象了：

```c++
if (CreateDebugUtilsMessengerEXT(instance, &createInfo, nullptr, &callback) != VK_SUCCESS) {
    throw std::runtime_error("failed to set up debug callback!");
}
```

倒数第二个参数依然是那个被我们设置成`nullptr`的可选的分配器回调函数，其余的参数含义都很明了。由于调试回调函数特定于我们的Vulkan实例和它的验证层，它们需要被显式设置为第一个参数。一会你还会看到这种模式用在其它“子”对象上。让我们看看它是否正常工作……运行程序，然后在你看腻了那个空白窗口之后关掉它。你应该看到如下消息被打印在命令提示符上：

![](https://vulkan-tutorial.com/images/validation_layer_test.png)

哎呀，在我们的程序中已经发现了一个bug！`VkDebugUtilsMessengerEXT`对象需要被`vkDestroyDebugUtilsMessengerEXT`函数清除。与`vkCreateDebugUtilsMessengerEXT` 类似，这个函数需要被显式加载。注意一下，消息被打印很多是正常的，因为有复数个验证层在检查调试messenger是否被删除。

在`CreateDebugUtilsMessengerEXT`下面添加另外一个代理函数：

```c++
void DestroyDebugUtilsMessengerEXT(VkInstance instance, VkDebugUtilsMessengerEXT callback, const VkAllocationCallbacks* pAllocator) {
    auto func = (PFN_vkDestroyDebugUtilsMessengerEXT) vkGetInstanceProcAddr(instance, "vkDestroyDebugUtilsMessengerEXT");
    if (func != nullptr) {
        func(instance, callback, pAllocator);
    }
}
```

确保这个函数是一个静态成员函数，或者是一个类外面的函数。这样我们可以在`cleanup`函数中调用它：

```c++
void cleanup() {
    if (enableValidationLayers) {
        DestroyDebugUtilsMessengerEXT(instance, callback, nullptr);
    }

    vkDestroyInstance(instance, nullptr);

    glfwDestroyWindow(window);

    glfwTerminate();
}
```

当你运行程序的时候，你将不会看到任何错误消息。如果你想看到哪个调用触发了消息，你可以在消息回调函数里打一个断点，然后看看堆栈跟踪。

## 配置

除了在`VkDebugUtilsMessengerCreateInfoEXT`结构体中指定标志之外，还有很多设置验证层行为的方法，浏览Vulkan SDK中的`Config`目录，`vk_layer_settings.txt` 文件解释了如何设置这些验证层。

要为你的应用程序设置验证层，把这个文件复制到你工程的`Debug`和`Release`文件夹里然后照着上面的说明来设置你想要的行为。然而，在此教程的余下部分，我假设你用的是默认设置。

在此教程中，我会故意犯几个错误来让你看看验证层对于捕获这些错误有多大的帮助，并且告诉你清楚地知道你在用Vulkan做什么有多重要。现在是时候看看系统中的Vulkan设备了。

[C++代码](https://vulkan-tutorial.com/code/02_validation_layers.cpp)