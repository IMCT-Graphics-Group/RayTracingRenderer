## 1. 前言

### 1.1 关于Validation Layers
需要注意的是，Validation Layers并不能捕获像未初始化内存和错误指针这样的bug。对于这样的bug，推荐使用Address Sanitizer（MSVC已支持）和Valgrind这样的工具。

### 1.2 基本思想
使用Vulkan开发，几乎所有的内容都是围绕手动创建和使用的对象设计的。这意味着不仅需要手动创建各类GPU资源，例如图片、纹理、缓冲数据（顶点数据等），也需要创建许多用于内部配置的结构体。

对于一些GPU固定函数，比如光栅化模式，就被存储在管线对象中，该对象还包含shader和其他配置信息。如果是在OpenGL和DX11中，这些都将在运行中计算。但使用Vulkan时，就需要考虑提前缓存这些对象还是留到运行时。像管线这类对象的创建是开销很大的，因此最好在加载场景时创建，或是启用一个后台线程创建。像描述符集这样的资源，创建比较轻松，留到渲染时创建也没有问题。

由于Vulkan中的很多内容都已经提前准备好，这意味着GPU中大部分的状态校验都会在创建对象时就完成，那么渲染本身所需要担负的工作就将减少，效率也得到提升。

当执行GPU指令时，实质上是将所有的GPU指令都录制到CommandBuffer（指令缓冲）中，然后将CommandBuffer提交到指令队列中。我们首先需要分配一个指令缓冲，然后向其中编码内容，最后将其添加到指令队列中执行。当我们将指令缓冲添加到指令队列后，这些指令就开始在GPU端执行。我们可以通过一些工具来控制它们何时结束执行。如果向不同的指令队列中提交不同的指令缓冲，它们就可能并行执行。

Vulkan中并没有”帧“这样的概念，这意味着所有的渲染都由开发者来把控。唯一需要注意的是何时将一帧通过交换链显示到屏幕上。渲染并通过网络发送图像，或者将图像保存到文件中，或者通过交换链将其显示到屏幕上之间没有本质上的区别。

### 1.3 Vulkan工作流

Vulkan中的主要术语：
- `VkInstance`：Vulkan的上下文实例，用于沟通驱动。
- `VkPhysicalDevice`：物理GPU，可以通过它请求物理GPU的详细信息，例如支持的特性（features）、性能、显存大小等。
- `VkDevice`：逻辑GPU，各种指令是交给逻辑GPU执行的。
- `VkBuffer`：（分配出来的）一块显存。
- `VkImage`：一张可以读写的纹理。
- `VkPipeline`：保存GPU绘制所需要的各种状态信息，比如Shader、光栅化选项、深度设置等。
- `VkRenderPass`：保存渲染图像相关的各种信息。所有绘制指令都必须在一次renderpass中执行完。
- `VkFrameBuffer`：保存renderpass的目标图像。
- `VkCommandBuffer`：编码GPU指令。所有需要在GPU上执行的内容都必须编码进`VkCommandBuffer`。
- `VkQueue`：指令的执行“端口”。GPU具有一组属性不同的队列，有些只允许传递图形指令，有些只允许内存指令……
- `VkDescriptorSet`：保存着关联shader输入和具体数据（`VkBuffer`资源、`VkImage`纹理等）的绑定信息。可以当作是GPU侧的指针。
- `VkSwapchainKHR`：保存着输往屏幕的图像。通过交换链可以将渲染的内容显示到窗口上。`KHR`后缀意味着该内容来自于API扩展，也就是来自于`VK_KHR_swapchain`扩展。
- `VkSemaphore`：用于同步CPU端和GPU端执行的指令。用于按顺序同步多个命令缓冲区的提交。
- `VkFence`：用于同步CPU端和GPU端执行的指令。用于了解某个指令缓冲是否在GPU端被执行完成。

Vulkan的上层工作流

1. **引擎初始化阶段**：首先，所有东西都需要初始化。初始化Vulkan是从创建`VkInstance`开始的，它要求传递进一组`VkPhysicalDevice`。如果电脑同时具备一张独立显卡和一张集成显卡，那么它们就分别对应一个`VkPhysicalDevice`。可以通过`VkPhysicalDevice`查询显卡所支持的特性，然后从符合需求的物理显卡创建一个逻辑设备`VkDevice`。有了`VkDevice`，就可以获取它的指令队列`VkQueue`，通过在指令队列中添加指令缓冲即可执行各种指令。和`VkQueue`同时出现的一般还有指令池`VkCommandPool`，用于分配录制指令所使用的指令缓冲`VkCommandBuffer`。另一个需要初始化的是交换链`VkSwapchainKHR`。
2. **资源初始化**：Vulkan的核心组件都初始化后，就可以开始初始化渲染所需要的各种资源。载入材质后，可以创建一组`VkPipeline`对象用于存储Shader绑定以及材质参数。对于网格模型，需要将顶点数据传递进`VkBuffer`中；贴图纹理则是传递进`VkImage`中，并且需要确保`VkImage`是在“readable” layout中。还需要创建多个`VkRenderPass`对象来执行渲染，比如一个`VkRenderPass`用于主渲染，另一个`VkRenderPass`则用于阴影渲染。在实际引擎中，上述很多操作都是在后台线程中并行执行的，特别是像管线创建这样的昂贵操作。
3. **渲染循环**：