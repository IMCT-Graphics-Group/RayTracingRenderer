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

- 通过`ash::Entry::linked()`加载Vulkan SDK

- 配置`app_info`和`create_info`

- 通过`entry.create_instance()`创建Vulkan实例


