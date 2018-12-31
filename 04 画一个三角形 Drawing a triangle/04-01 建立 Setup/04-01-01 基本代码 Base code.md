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

int main() { HelloTriangleApplication app; try { app.run(); } catch (const  std::exception& e) { std::cerr << e.what() << std::endl; return EXIT_FAILURE; } return EXIT_SUCCESS; }
```
