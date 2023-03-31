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

Vulkan的工作流

1. **引擎初始化阶段**：首先，所有东西都需要初始化。初始化Vulkan是从创建`VkInstance`开始的，它要求传递进一组`VkPhysicalDevice`。如果电脑同时具备一张独立显卡和一张集成显卡，那么它们就分别对应一个`VkPhysicalDevice`。可以通过`VkPhysicalDevice`查询显卡所支持的特性，然后从符合需求的物理显卡创建一个逻辑设备`VkDevice`。有了`VkDevice`，就可以获取它的指令队列`VkQueue`，通过在指令队列中添加指令缓冲即可执行各种指令。和`VkQueue`同时出现的一般还有指令池`VkCommandPool`，用于分配录制指令所使用的指令缓冲`VkCommandBuffer`。另一个需要初始化的是交换链`VkSwapchainKHR`。
2. **资源初始化**：Vulkan的核心组件都初始化后，就可以开始初始化渲染所需要的各种资源。载入材质后，可以创建一组`VkPipeline`对象用于存储Shader绑定以及材质参数。对于网格模型，需要将顶点数据传递进`VkBuffer`中；贴图纹理则是传递进`VkImage`中，并且需要确保`VkImage`是在“readable” layout中。还需要创建多个`VkRenderPass`对象来执行渲染，比如一个`VkRenderPass`用于主渲染，另一个`VkRenderPass`则用于阴影渲染。在实际引擎中，上述很多操作都是在后台线程中并行执行的，特别是像管线创建这样的昂贵操作。
3. **渲染循环**：经过初始化，现在可以开始渲染了。首先向`VkSwapchainKHR`请求一张图像用于渲染写入。接着从`VkCommandBufferPool`中分配一块`VkCoomandBuffer`来录入指令（或者重复利用已经执行结束的`VkCommandBuffer`）。然后开始执行renderpass来进行渲染，`VkRenderPass`可以指定渲染写入的图像（之前从`VkSwapchainKHR`中请求的），接着创建循环来给renderpass绑定`VkPipeline`、绑定`VkDescriptorSet`、绑定顶点缓冲，然后执行绘制指令（draw call）。renderpass录制之后，如果没有额外需要渲染的内容，可以结束`VkCommandBuffer`的指令录制并将指令提交给GPU执行。如果最后需要将渲染的图像呈现到屏幕上，则需要通过同步指令（semaphore）等待渲染出图像。

**渲染循环**的流程用伪代码表示为：
```Cpp
// Ask the swapchain for the index of the swapchain image we can render onto
int image_index = request_image(mySwapchain);

// Create a new command buffer
VkCommandBuffer cmd = allocate_command_buffer();

// Initialize the command buffer
vkBeginCommandBuffer(cmd, ... );

// Start a new renderpass with the image index from swapchain as target to render onto
// Each framebuffer refers to a image in the swapchain
vkCmdBeginRenderPass(cmd, main_render_pass, framebuffers[image_index] );

// Rendering all objects
for(object in PassObjects){

    // Bind the shaders and configuration used to render the object
    vkCmdBindPipeline(cmd, object.pipeline);
    
    // Bind the vertex and index buffers for rendering the object
    vkCmdBindVertexBuffers(cmd, object.VertexBuffer,...);
    vkCmdBindIndexBuffer(cmd, object.IndexBuffer,...);

    // Bind the descriptor sets for the object (shader inputs)
    vkCmdBindDescriptorSets(cmd, object.textureDescriptorSet);
    vkCmdBindDescriptorSets(cmd, object.parametersDescriptorSet);

    // Execute drawing
    vkCmdDraw(cmd,...);
}

// Finalize the render pass and command buffer
vkCmdEndRenderPass(cmd);
vkEndCommandBuffer(cmd);


// Submit the command buffer to begin execution on GPU
vkQueueSubmit(graphicsQueue, cmd, ...);

// Display the image we just rendered on the screen
// renderSemaphore makes sure the image isn't presented until `cmd` is finished executing
vkQueuePresent(graphicsQueue, renderSemaphore);
```

## 2. 开发

### 2.1 VkInstance
`VkInstance`表示Vulkan API的上下文，创建`VkInstance`时可以选择开启校验层（validation layers）、选择需要的扩展（例如`VK_KHR_surface`）以及关联日志。

整体来说，一个应用程序通常只需要一个`VkInstance`贯穿全局，它也充当着全局的Vulkan上下文。

### 2.2 VkPhysicalDevice
通过`VkInstance`可以获取系统中的GPU，这些GPU的信息都可以通过相应的`VkPhysicalDevice`访问，每个`VkPhysicalDevice`都指向一个实际存在的GPU设备。`VkPhysicalDevice`还用于查询一些API特性（feature）、显卡的显存以及对API扩展的支持。

### 2.3 VkDevice
当选定了需要使用的`VkPhysicalDevice`后，可以由它创建一个`VkDevice`。`VkDevice`事实上是GPU驱动，也就是硬件上的程序，是应用程序沟通GPU的对象。Vulkan除调试工具和初始化操作外的大部分指令都需要通过`VkDevice`完成。

`VkDevice`在创建时需要指定启用的API扩展，最好只启用实际需要用到的扩展，否则会浪费额外的时间。Vulkan在设计时一个重要目标是手动处理多GPU运算，这可以通过分别创建`VkDevice`实现，`VkDevice`之间可以共享数据。例如，可以使用独立显卡运行重要的图形渲染程序，同时通过集成显卡运行一些物理计算或数据处理。

### 2.4 Swapchain
如果需要将渲染的结果输出到屏幕上，就需要通过交换链（swapchain）实现。交换链并不包含在Vulkan的核心规范中，是一个可选的实现，而且通常因平台而异。如果使用Vulkan做一些离屏渲染，比如直接将渲染结果存储到本地磁盘，或是做一些离线计算，那么完全没有必要创建交换链，交换链唯一的作用就是将图像发送到屏幕上。

交换链在创建时需要指定画面大小，如果窗口的尺寸发生了变化，就必须重新创建交换链。

不同平台、不同GPU的交换链可能会需要不同格式的图像，所以最好使用交换链所需要的图像格式，否则可能会在其他的运行环境下出现画面异常或崩溃。

交换链实质上是一组图像，操作系统会获取交换链并将图像显式到屏幕上。具体实现时我们可以自由指定图像的数量，但一般情况下只创建2~3张图像，也就是双重缓冲或三重缓冲。

创建交换链时还有一项重要的工作是指定**显示模式**（Present Mode），也就是指定交换链的工作模式：
- `VK_PRESENT_MODE_IMMEDIATE_KHR`：交换链不做任何等待，将获取到的每一张图像都立刻刷新到屏幕上。该模式下很可能造成画面撕裂，一般不推荐使用该模式。
- `VK_PRESENT_MODE_FIFO_KHR`：交换链使用先进先出队列按一定间隔刷新屏幕。当队列满列时，就需要等待队列空出位置再执行渲染。该模式是一种强制垂直同步的显示模式，可以固定画面刷新率（FPS）。
- `VK_PRESENT_MODE_FIFO_RELAXED_KHR`：大部分情况下表现得和垂直同步相同，但是自适应的。当应用程序的渲染效率低于画面刷新率时，会立刻使用获取的每一张图像，可能造成画面撕裂。举例来说，屏幕刷新率为60Hz，而渲染效率为55Hz时，该模式并不会像垂直同步那样将画面刷新率降低到30Hz，而是采用55Hz的刷新率，但可能会引起画面撕裂。
- `VK_PRESENT_MODE_MAILBOX_KHR`：该模式下缓存一组图像，交换链总是选取当前最新的图像进行显式。当想要实现没有强制垂直同步的三缓冲效果时，可以使用该模式。

实际开发中很少使用`VK_PRESENT_MODE_IMMEDIATE_KHR`模式，通常使用MAILBOX模式或者两种FIFO模式之一。

### 2.5 Commands
