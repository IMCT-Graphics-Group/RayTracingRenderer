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

## 2. 基础渲染

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

### 2.5 指令执行
与DX11和OpenGL不同，Vulkan中的所有GPU指令都需要通过指令缓冲（command buffer）传递。指令缓冲从指令池（Command Pool）中申请，在队列中执行。

指令执行的一般流程为：
- 从`VkCommandPool`中分配一块`VkCommandBuffer`
- 通过`VkCmdXXXXX`函数将指令录制到指令缓冲中
- 将指令缓冲提交到`VkQueue`中执行

同一块指令缓冲是可以多次使用的，有些案例会在录入一次指令缓冲后，每帧重复提交。但实际应用中一般每帧都需要重新录制指令缓冲。

录制指令缓冲的开销很小，绝大多数工作发生在调用`VkQueueSubmit`时，这时需要检查指令的有效性并转译为真正的GPU指令。

指令缓冲是可以并行录制的，可以在不同的线程中同时向不同的指令缓冲录入指令。但这需要在不同的线程中分别使用`VkCommandPool`和`VkCommandBuffer`，并确保该线程只会使用自己线程的指令池和指令缓冲。最后需要在一个线程中分别提交这些指令缓冲，因为`vkQueueSubmit`并不是线程安全的，同一时间只能有一条线程提交指令。

### 2.6 VkQueue
队列是GPU的执行端口，每个GPU都有多个队列可获取，可以同时使用这些队列执行不同的指令流。提交到不同队列的指令也可能同时执行，这对于执行一些不需要和主线程同步的后台工作来说非常有用。比如，可以创建一个`VkQueue`专门用于某些后台工作，从而与主渲染任务区分开来。

Vulkan中的所有队列都来自于队列族（Queue Family）。队列族实际上就是队列的“类型”，标示着该队列支持哪些指令。例如，支持图形和显示的队列就可以用于渲染和刷新屏幕。

不同的GPU支持不同的队列族，例如某个GPU可能支持两种队列族，其中一种队列族支持所有的特性并提供了16个队列，而另一种仅支持传输且只提供1个队列。仅支持传输的队列族在Vulkan中比较特殊，常被用于在后台加载各种资源，一般可以独立于渲染流程而完全地异步执行。因此，如果需要在后台线程中加载GPU资源，使用仅支持传输的队列是一个很好的选择。

### 2.7 VkCommandPool
指令池`VkCommandPool`是从`VkDevice`中创建的，而且在创建时需要指定相应的队列族索引。可以将指令池看作指令缓冲`VkCommandBuffer`的后台分配器，可以从指令池中分配任意多的指令缓冲，但同时只能录制来自一个线程的指令。如果需要多线程同时录制指令，就需要创建多个`VkCommandPool`对象。

指令缓冲通常分配得很少，而且每次都需要重置。最快捷的方式是重置整个指令池，这样就能一起重置所有分配的指令缓冲。也可以直接重置指令缓冲，但如果从指令池中分配了多个指令缓冲，那么重置指令池会更快。

### 2.8 VkCommandBuffer
所有送往GPU的指令都需要录制到指令缓冲中。所有需要GPU执行的函数都需要等到指令缓冲被提交到GPU时才会执行。

指令缓冲分配后一般会处于就绪状态（Ready），这时可以调用`vkBeginCommandBuffer()`来将指令缓冲切换到录制状态（Recording）。这时便可以将各种`vkCmdXXXXX`函数录制到指令缓冲中。当指令录制完毕，可以调用`vkEndCommandBuffer()`来结束录制并将指令缓冲切换至可执行状态（Executable），这时就可以准备提交到GPU中了。提交指令缓冲是通过`vkQueueSubmit()`实现的，需要指定提交的指令缓冲和提交到的队列。提交后的指令缓冲将变成挂起状态（Pending）。

提交后的指令缓冲仍然处于活跃状态，可能会被GPU使用，所以这时就重置指令缓冲是不安全的。需要确认GPU已经执行完指令缓冲的全部指令之后再重置该指令缓冲。重置指令缓冲通过`vkResetCommandBuffer()`实现。

### 2.9 渲染通道（Renderpass）
Vulkan中的所有渲染都发生在`VkRenderPass`中。无法在渲染通道外执行渲染指令，但可以执行计算指令。

`VkRenderPass`是对渲染目标、渲染图像状态的一个封装对象。渲染通道（renderpass）是一个仅存在于Vulkan中的概念，可以给驱动提供更多有关渲染目标图像的状态信息。渲染通道会将结果渲染进帧缓冲（Framebuffer）中，帧缓冲链接着渲染目标的图像，当渲染通道开始执行时，帧缓冲会将该图像设置为渲染目标。

渲染通道的一般指令流程为：
```C++
vkBeginCommandBuffer(cmd, ...);

vkCmdBeginRenderPass(cmd, ...);

//rendering commands go here

vkCmdEndRenderPass(cmd);

vkEndCommandBuffer(cmd)
```

启动渲染通道时需要设置目标帧缓冲和clear color。

### 2.10 子通道（Subpasses）
一个渲染通道包含多个子通道，有点类似于渲染过程的“步进”。子通道对于移动端GPU来说非常有用，它可以给驱动留下很大的优化空间。对于桌面端GPU来说，子通道则相对不那么重要。每个渲染通道最少需要一个子通道。

### 2.11 图像布局（Image Layouts）
渲染通道的一项非常重要的工作在于，进入和退出渲染通道时修改图像布局。

GPU中的图像未必是我们需要的格式。出于优化考虑，GPU会对图像执行大量的转换和组合，将其变成内部不透明格式。例如，有些GPU会尽可能压缩各类纹理并重新排列像素以利于生成mipmap。Vulkan中不需要我们控制格式转换，但需要控制图像布局，这能使得驱动将图像转变为内部优化的格式。

常用图像布局为：
- `VK_IMAGE_LAYOUT_UNDEFINED`：不关心图像布局，可以是任意一种布局。
- `VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL`：优化写入的图像布局。
- `VK_IMAGE_LAYOUT_PRESENT_SRC_KHR`：可用于屏幕显示的图像布局。
- `VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`：优化Shader读入的图像布局。

### 2.12 同步
Vulkan为CPU端和GPU端的执行同步提供了显式的同步结构。并且可以控制GPU的执行顺序。所有执行的Vulkan指令都将进入队列，并以未定义的顺序“不间断”地执行。

有时需要确认某个操作是否已经执行完成，虽然队列中的指令通常是顺序执行的，但是多个队列之间的先后顺序是无法保证的。为此，Vulkan提供了`VkFence`和`VkSemaphore`。

- `VkFence`：用于GPU->CPU的通信。许多像`vkQueueSubmit`这样的操作允许传递进一个可选的fence参数，如果传递了fence参数，就可以在CPU端知晓GPU是否执行完该操作（阻塞CPU直到GPU执行完成）。
- `VkSemaphore`：用于GPU和GPU之间的同步。Sempahore可用于定义GPU指令之间的操作顺序，部分操作允许设置发射信号或是等待信号。如果设置该操作发射信号，则在操作执行完之前都会锁定，直到执行完毕再释放。如果设置该操作等待信号，则该操作将一直等到信号被释放才执行。

Semaphore示例伪代码：
```C++
VkSemaphore Task1Semaphore;
VkSemaphore Task2Semaphore;

VkOperationInfo OpAlphaInfo;
// Operation Alpha will signal the semaphore 1
OpAlphaInfo.signalSemaphore = Task1Semaphore;

VkDoSomething(OpAlphaInfo);

VkOperationInfo OpBetaInfo;

// Operation Beta signals semaphore 2, and waits on semaphore 1
OpBetaInfo.signalSemaphore = Task2Semaphore;
OpBetaInfo.waitSemaphore = Task1Semaphore;

VkDoSomething(OpBetaInfo);

VkOperationInfo OpGammaInfo;
//Operation gamma waits on semaphore 2
OpGammaInfo.waitSemaphore = Task2Semaphore;

VkDoSomething(OpGammaInfo);
```

### 2.13 渲染管线
完整的Vulkan图形管线非常复杂，本部分只介绍一种简化版。

数据（data）---> 顶点着色器（vertex shader）---> 光栅化（rasterization）---> 片元着色器（fragment shader）---> 渲染输出

- 数据指任何渲染所需要的数据，包括不限于纹理、3d模型、材质参数。
- 顶点着色器是逐顶点运行的一小段程序，可以将各顶点的位置和参数信息转换为光栅化阶段所需要的形式。
- 光栅化阶段基于顶点信息找到三角形所覆盖的像素，这些像素将作为片元着色器的输入信息。
- 片元着色器是逐像素运行的一段程序，将像素绘制为最终颜色。
- 片元着色器的绘制结果将被送往帧缓冲中，如果有透明像素，则透明混合操作也在该阶段进行。

Vulkan中的`VkPipeline`是一个复杂的对象，包含GPU渲染所需要的全部配置信息。构建`VkPipeline`对象是一个非常昂贵的操作，这个过程中需要将着色器模块转译为GPU指令并校验配置信息。

### 2.14 管线布局
除了各种状态结构，渲染管线还需要一个`VkPipelineLayout`对象。这是一个Vulkan中的对象，需要在管线之外单独创建。

管线布局包含管线中着色器的输入信息，需要在管线布局中配置描述符集。

## 3. 模型渲染

### 3.1 顶点缓冲
Vulkan中可以分配出对GPU可见的内存，并使shader读取其中的信息。这类内存有两种形式：图像和缓冲。图像通常是2d或3d的数据，比如纹理。缓冲则是GPU可以直接读写的内存块。缓冲有几种不同的类型，顶点缓冲是其中最常见的类型之一。

顶点缓冲允许GPU读取其中的数据，并将数据发送给顶点着色器（vertex shader）。着色器中读取顶点缓冲，需要在管线中设置顶点输入状态，该状态告诉Vulkan如何将缓冲数据解释为顶点数据。

顶点输入布局（vertex input layout）可用于描述顶点数据的结构。

### 3.2 常量传递（Push Constants）
常量传递可以通过指令缓冲将一小部分数据直接发送到shader中（各个阶段都行），性能表现也很不错。

使用常量传递，需要在各个着色器阶段都指定常量传递的大小，这些信息在创建VkPipelineLayout时指定。然后使用`vkCmdPushConstants`指令将数据封装进指令缓冲中传递。

常量传递允许最小128字节的信息传递，足够传递一个4x4的矩阵和一些参数。

常量传递按区域（Range）描述，这意味着可以在不同的区域内存储不同的常量。例如将一个64字节的矩阵存储到顶点着色器上，然后从offset=64的位置开始存储一些片元着色器中才需要用到的参数。

### 3.3 深度图
交换链中的图像不包含深度图，因而需要单独创建深度图用于深度测试。

### 3.4 资源描述符集（Descriptor Sets）
虽然可以使用常量传递来将一些数据从CPU端发送到GPU端，但是这种方式有很大的限制，比如不能用于传递数组，也不能传递资源缓冲的指针。后者就需要使用资源描述符集，这是联系CPU到GPU数据的主要方式。

一个资源描述符相当于一个指向具体资源的指针。具体资源可以是一个缓冲或是一张图像，其中也包含其他信息，比如缓冲的大小或图像的采样器。

资源描述符集就是这样一组绑定在一起的指针。Vulkan不允许单独绑定某一个资源到shader上，所以需要通过资源描述符集实现资源绑定。但这在一些硬件上并不高效，比如Intel的集成显卡只支持每根管线最多4个资源描述符集。常用的一种解决方法是按绑定频率对资源描述符集分组。0号资源描述符集用于全局资源，每一帧都会重新绑定；1号资源描述符集用于逐pass的资源；2号资源描述符集用于材质资源；3号资源描述符集则用于逐物体的资源。这样，渲染循环中一般只需要绑定2号和3号资源描述符集。

资源描述符集必须通过`VkDescriptorPool`分配。资源描述符集会是从GPU显存中分配的一块区域。分配完资源描述符集后就需要使其指向资源文件（buffers/textures）。当资源描述符集被用于`vkCmdDraw()`时就不可以再进行修改，除非设置过`VK_DESCRIPTOR_POOL_CREATE_UPDATE_AFTER_BIND_BIT`标记。

创建资源描述符池后，需要指定资源描述符集的数量，以及资源的数量。默认一般都会设置一个较大的值，比如1000。

实际引擎中每帧都用一组描述符池，一旦描述符分配失败，就在列表中再新建一个描述符池。指令提交后会重置这一组描述符池。

刚从GPU显存中分配的资源描述集并不占用什么空间，接下来就需要通过`vkUpdateDescriptorSets()`来更新资源绑定。如果设置了Update After Bind的标记，那么可以在指令缓冲中绑定资源后（提交指令缓冲之前）。

资源描述符集是存储在Vulkan管线的一个特定位置的。当创建管线时，需要指定每个资源描述符集绑定到管线的布局（layout）。这一般是通过反射shader来自动绑定的。资源描述符集的布局描述了资源描述符集的形状，例如一个布局可能是由2个缓冲和1张图像构成。如果描述符集布局相同，即使它们是在两个不同的地方创建的，它们也可以兼容。

假设我们有两根管线，其中一根管线的资源描述符集0绑定到一个缓冲上，资源描述符集1绑定到4张图像上；另一根管线的资源描述符集0绑定到同一个缓冲上，但资源描述符集1绑定到3张图像上。那么在绑定第二根管线的时候，资源描述符集0是绑定好的，而资源描述符集1就需要重新绑定。这也是为什么需要根据资源描述符集的更新频率来分组。

多个资源描述符可以指向同一个缓冲的不同区域，而且这通常是一个更好的实现方式。有许多技术可以做到这一点，例如动态描述符，它可以重复使用同一个资源描述集，但每次会分配不同的缓冲偏移。唯一需要注意的是数据上的对齐，GPU通常不能访问任意地址，它有一个最小对齐尺寸。可以通过查询GPU信息获取该尺寸，该属性是`minUniformBufferOffsetAlignment`。

如果是每帧都需要更新的信息，例如相机矩阵，那么使用一个常规资源描述符来表示是最合适的；但如果是逐物体的数据，那么可以考虑使用动态描述符。

出于动态描述符的灵活性，许多游戏引擎甚至全程采用动态描述符。使用动态描述符可以在运行中分配并写入新的缓冲数据。

### 3.5 存储缓冲（Storage buffer）
统一缓冲（Uniform Buffer）一般用于较小的只读数据，如果数据量不能确定，或是数据要求可写入，则一般使用存储缓冲（Storage Buffer）。存储缓冲会更慢一点，但是支持存储大量数据，如果需要将整个场景存入缓冲中，就必须使用存储缓冲。

使用存储缓冲可以向shader中传递一个尺寸不定的数据块，最常见的用法就是存储场景中的所有物体。

### 3.6 内存转换（Memory transfer）
有时候需要从CPU端传入数据，但传完之后为了寻求更高的访问效率而需要将其转变为GPU_ONLY，这就需要使用内存转换。

### 3.7 图像（Image）
`VkImage`存有实际的图像数据，包括像素信息等，但不能直接读取。从效率上考虑，通常创建具有Optimal布局的图像格式。

`VkImageView`是对`VkImage`的包裹，含有图像的各种辅助信息，例如如何解释图像数据。与`VkImage`不同，`VkImageView`不一定非得分配在GPU显存中，因而可以直接通过API创建。可以将`VkImageView`看作是指向一张`VkImage`的指针。

`VkSampler`是着色器实际访问纹理时所使用的。它控制着多个固定管线状态，例如mimap blending以及最近邻滤波。它也可用于控制各向异性过滤，或是图像采样越界的处理方案。采样器的创建不依赖与特定的图像，因而引擎中常常是提前创建好并通过hashmap缓存起来。

图像总是从`VK_IMAGE_LAYOUT_UNDEFINED`布局开始的，因此需要将图像布局转为其他布局，例如`VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL`。转换图像布局的时候需要使用Pipeline Barrier来控制着色器的资源访问同步。

## 4. GPU Driven Rendering
受益于MultiDrawIndirect等几个类似的特性，可以用compute shader完成很多渲染工作。

### 4.1 Draw Indirect
GPU Driven Rendering的核心是围绕Draw Indirect的。该技术出现在了几乎所有的API中，但在DX12和Vulkan中表现得最好，因为这两个现代API支持对底层内存管理和计算阻塞的控制。

Draw Indirect技术是指drawcall不再是从调用中获取参数，而是从GPU缓冲中获取。使用draw indirect时，只需指定绘制所需的GPU缓冲位置，则GPU会直接执行该缓冲上的绘制指令。可以先指定好缓冲，再写入指令参数，而且可以从不同的线程、计算着色器中写入。

由于参数都存在特定的缓冲上，因此可以使用计算着色器来写入，乃至于在计算着色器中做一些裁剪、选择LOD等操作。得益于GPU强大的算力，在计算着色器中做裁剪可以在半毫秒内完成数百万物体的处理。《龙腾世纪：审判》和《彩虹六号：围攻》中则更进一步，对网格的三角形面片也做了裁剪。

设计GPU-driven渲染器的核心理念是，所有的场景都应存储在GPU上，包括场景物体的各种信息。如果渲染器能在GPU缓冲中获取所有需要的信息，那么就不再需要常量传递和资源描述符集。

由于我们希望将尽可能多的信息存储在GPU上，那么结合使用“无绑定”技术就不再需要为每种材质指定资源描述符集或是修改顶点缓冲。在《毁灭战士：永恒》的引擎中全面使用了无绑定技术，最终每一帧只需要很少的几个drawcall。

使用无绑定技术的渲染器也可以大幅加速光追。

### 4.2 无绑定设计
无绑定的设计可以使得CPU的工作量大幅减少，同样得益于单次更多工作的绘制指令，GPU的使用率也得到了大幅提升。

将顶点缓冲和索引缓冲改为无绑定设计，需要先将它们合并为一个大缓冲，将每个缓冲存一个顶点缓冲和一个索引缓冲改为在一个缓冲中存储场景中所有的顶点缓冲。一些引擎甚至从管线上移除了顶点属性，直接由顶点着色器从缓冲中抓取顶点数据。这样做也有利于使用一些高级压缩算法，这也是Mesh Shader的主要用例。

将纹理改为无绑定设计需要使用纹理数组。通过合理扩展，着色器中的纹理数组大小可以不受限制，就像使用SSBO一样。着色器访问纹理时，可以直接根据索引从缓冲中抓取。不使用资源描述符索引扩展也可以使用纹理数组，但数组大小会受到限制。

将材质改为无绑定设计需要将所有的材质信息存进SSBO中，并通过一个ubershader来获取。《毁灭战士：永恒》的引擎中使用了很少的几根渲染管线（不到500），而虚幻游戏经常动辄10000+管线。如果使用ubershader来大幅减少单独的管线数量，则可以大幅提升效率，因为管线绑定是一个非常耗时的操作。

常量传递和动态描述符可以正常使用，但会用于全局数据。对相机位置这样的数据使用常量传递仍然十分理想。

整体的思路是将数据尽可能都放进缓冲中，然后就不再需要每次drawcall都绑定一遍。