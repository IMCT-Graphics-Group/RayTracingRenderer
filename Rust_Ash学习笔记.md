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

### 9. 着色器

Vulkan中的着色器必须使用一种被称为`SPIR-V`字节码格式。`SPIR-V`可用于编写图形着色器和计算着色器。一般通过`glslc.exe`（已包含于Vulkan SDK）将GLSL编译为`SPIR-V`。

在渲染光线中绑定着色器一般遵循以下步骤：

- 读取着色器的`spv`字节码文件

- 配置`ShaderModuleCreateInfo`，创建`ShaderModule`

- 配置`PipelineShaderStageCreateInfo`
