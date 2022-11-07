# Rust-Ash(Vulkan) 学习笔记

### 1. 窗口创建

Windows中创建窗口需要调用特定的系统API，Rust中一般使用`winit`包实现。

创建windows窗口遵循以下流程：

- 使用`Eventloop::new()`创建`Eventloop`实例

- 使用`WindowBuilder::new()`创建窗口实例

窗口中可能会产生各种各样的**窗口事件**（WindowEvent），这些窗口事件是由各种输入事件引起的，例如鼠标划过窗口、聚焦时的键盘输入。当然，各类设备也会产生**设备事件**（DeviceEvent），但设备事件通常不区分应用窗口。

可以通过`EventLoop::run`持续地运行并获取各类事件，直到该函数的控制流（control_flow）参数被设为`ControlFlow::Exit`，那么程序就会尝试结束运行。

### 2. 创建Vulkan实例

使用Ash创建Vulkan实例遵循以下流程：

- 通过`ash::Entry::linked()`加载Vulkan SDK（需要开启相关feature）

- 配置`app_info`和`create_info`

- 通过`entry.create_instance()`创建Vulkan实例

### 3. Validation layers

为了减少驱动负担，Vulkan中几乎没有错误检查。但Vulkan提供了一种优雅的机制来辅助开发，那就是`validation layers`。validation layers中通常包含的操作有：

- 根据规范检查参数值以检测是否存在误用

- 跟踪对象的创建和销毁以避免资源泄露

- 跟踪线程的发起端以保证线程安全

- 将每个调用及其参数记录到标准输出

- 跟踪Vulkan的调用以供分析和复现

启用validation layers一般分为以下几个步骤：

- 通过`entry.enumerate_instance_layer_properties()`获取所有layer

- 检查所有获取到的layer中是否存在所需的validation layers

- 将可用的validation layers添加到`create_info`中

虽然validation layers会将debug信息打印到标准输出中，但我们也完全可以通过一个回调来处理debug信息。自定义debug回调需要遵循以下步骤：

- 获取扩展信息

- 将扩展信息添加到`create_info`中

- 从系统引入并创建自定义的回调函数

- 将自定义的回调函数添加到`create_info`中

### 4. 物理设备和队列族

选取物理设备和队列族（queue family）一般遵循以下步骤：

- 获取所有可用的物理设备列表

- 检查各物理设备的适配性

- 检查物理设备的队列族的适配性

- 选取满足需要的设备

### 5. 逻辑设备和队列

选好物理设备后，需要为它设置一个（或多个）逻辑设备。

设置逻辑设备遵循如下步骤：

- 配置`DeviceQueueCreateInfo`

- 配置`DeviceCreateInfo`

- 通过实例的`create_device()`方法创建逻辑设备和队列

### 6. Window surface

为了建立Vulkan和窗口之间的连接,需要使用WSI扩展。Window surface应当在实例创建之后立即创建，且遵循以下步骤：

- 配置`Win32SurfaceCreateInfo`

- 创建`Win32Surface`实例

- 通过`Win32Surface`实例的`create_win32_surface`创建SurfaceKHR

- 创建`ash::extensions::khr::Surface`实例（通用上层抽象）

### 7. 交换链

为了展示图像序列，需要创建和使用交换链。交换链的创建遵循以下步骤：

- 检查交换链的支持性

- 在`DeviceCreateInfo`中启用`Swapchain`扩展

- 检查交换链细节属性的支持性（图片容量、图片格式、展现模式）

- 选取最佳配置（surface格式--颜色深度、展现模式--切换条件、交换规模--图片分辨率）

- 优先使用BGRA-24bit-Nonlinear图片格式

- Vulkan中的展现模式有4种：`IMMEDIATE`立即更新图片（可能会造成画面撕裂）、`FIFO`先进先出队列（类似于垂直同步）、`FIFO_RELAXED`队列为空时不再等待垂直空白而是直接应用（可能会导致画面撕裂）、`MAILBOX`使用新的图片替换队列中的图片而不是阻塞队列（三重缓冲）。

- 交换规模。绝大多数情况下，交换规模所指定的图片分辨率总是和窗口的像素分辨率一致。

- 配置`SwapchainCreateInfo`，创建交换链

### 8. 图片视图

为了使用图片，我们需要创建图片视图对象。创建图片视图对象遵循以下步骤：

- 遍历每张图片，配置`ImageViewCreateInfo`

- 根据`ImageViewCreateInfo`创建`ImageView`

### 9. 着色器与渲染阶段

Vulkan中的着色器必须使用一种被称为`SPIR-V`字节码格式。`SPIR-V`可用于编写图形着色器和计算着色器。一般通过`glslc.exe`（已包含于Vulkan SDK）将GLSL编译为`SPIR-V`。

在渲染管线中绑定着色器和配置渲染阶段一般遵循以下步骤：

- 读取着色器的`spv`字节码文件

- 配置`ShaderModuleCreateInfo`，创建`ShaderModule`

- 配置`PipelineShaderStageCreateInfo`

- 配置`PipelineVertexInputStateCreateInfo`

- 配置`PipelineInputAssemblyStateCreateInfo`

- 配置`PipelineViewportStateCreateInfo`

- 配置`PipelineRasterizationStateCreateInfo`

- 配置`PipelineMultisampleStateCreateInfo`

- 配置`PipelineDepthStencilStateCreateInfo`

- 配置`PipelineColorBlendStateCreateInfo`

- 配置`PipelineLayoutCreateInfo`

- 创建`PipelineLayout`

### 10. 渲染Pass

创建渲染Pass遵循以下步骤：

- 配置`AttachmentDescription`

- 配置`SubpassDescription`

- 配置`RenderPassCreateInfo`

- 创建`RenderPass`

### 11. 渲染管线

完成上述步骤后即可创建渲染管线。创建渲染管线遵循以下步骤：

- 配置`GraphicsPipelineCreateInfo`

- 创建`Pipeline`

### 12. 帧缓冲

为了从交换链中获取和显示图片，我们需要创建一个帧缓冲。

为每个`ImageView`创建帧缓冲遵循以下步骤：

- 配置`FramebufferCreateInfo`

- 创建`Framebuffer`

### 13. 命令缓冲

Vulkan中的命令需要通过命令缓冲对象传递。创建命令缓冲对象，首先需要创建一个命令池：

- 配置`CommandPoolCreateInfo`

- 创建`CommandPool`

然后就可以创建命令缓冲对象了：

- 配置`CommandBufferAllocateInfo`

- 创建`CommandBuffer`数组

对于每一个命令缓冲，遵循以下处理步骤：

- 创建`CommandBufferBeginInfo`

- 调用`begin_command_buffer`开始记录命令

- 配置`RenderPassBeginInfo`

- 调用`cmd_begin_render_pass`开启渲染pass

- 调用`cmd_bind_pipeline`绑定渲染管线

- 调用`cmd_draw`绘制

- 调用`cmd_end_render_pass`结束渲染pass

- 调用`end_command_buffer`结束记录命令

### 14. 渲染与画面呈现

渲染一帧图片通常包括以下阶段：

- 等待上一帧图片结束

- 向交换链请求一张新的图片

- 记录绘制图片的命令缓冲

- 提交命令缓冲

- 呈现交换链中的图片

处理同步问题通常使用Semaphores，但这并不会阻塞CPU端的执行；阻塞CPU端的执行需要使用Fences。等待上一帧图片结束应当使用fence，而交换链上的操作则只需要使用semaphores。

创建Sempahores遵循以下步骤：

- 配置`SemaphoreCreateInfo`

- 创建`Semaphore`对象

提交命令缓冲遵循以下步骤：

- 配置`SubmitInfo`

- 通过`queue_submit`提交命令缓冲

展示画面遵循以下步骤：

- 配置`PrensetInfoKHR`

- 通过`queue_present`提交展示指令

### 15. 重新生成交换链

当window surface发生改变时（调整窗口大小），需要重新生成交换链。

重新生成交换链遵循以下步骤：

- 等待逻辑设备进入闲置状态

- 销毁原来的交换链（释放命令缓冲、销毁帧缓冲、销毁渲染管线等）

- 创建新的交换链

### 16. 从管线中获取顶点信息

配置管线接受顶点信息，需要为管线的顶点输入信息配置绑定信息和属性信息。
