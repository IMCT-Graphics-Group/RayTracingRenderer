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


