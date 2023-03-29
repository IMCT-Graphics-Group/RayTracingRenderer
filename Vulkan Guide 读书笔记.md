# 1. 什么是Vulkan
Vulkan是新一代的图形和计算API，它提供了对现代图形硬件的跨平台高效访问。

### 1.1 核心概念
就本质而言，Vulkan是一套API规范。相比于OpenGL，Vulkan并不是其直接替代，而是一种允许更明确地调用GPU的显式API。

### 1.2 OpenGL ES vs. Vulkan
- **状态管理**：OpenGL ES使用一个全局状态来描述，因而每次draw call都需要重新创建必要的渲染状态和资源绑定表。由于状态绑定只在绘制阶段被确定，所以一些优化算法无法适用。Vulkan则使用状态对象（描述符）来描述渲染状态，这就允许上层应用提前将一些状态绑定打包起来。从而能采用一些基于Shader的优化方案，降低运行期开销。状态管理上的优化显著降低了CPU端的调度开销，代价是需要上层应用提前确定好状态。
- **API执行模型**：OpenGL ES采用同步渲染执行模型，也就是各API调度必须按顺序执行。实际上，现代GPU不会这么处理，各种GPU资源调度仍然是异步进行的。但为了确保API层面上的同步执行，硬件驱动必须追踪各资源的读写来确保各渲染操作的正确执行。Vulkan则使用异步渲染执行模型，与现代GPU的底层模式相合。上层应用将渲染指令填充进指令队列中，使用显式的调度依赖来控制执行顺序，并且使用显式的异步原语来对齐CPU和GPU的处理。
- **API线程模型**：OpenGL ES使用单线程渲染模型，这大大制约了多核处理器在渲染任务中的发挥。Vulkan则使用多线程渲染模型。值得一提的是，ARM的处理器通常使用了”异构多核技术“（也就是所谓的”大小核“），该技术将高性能的”大核“与节能的”小核“结合使用，该技术可以大幅减少处理器的功耗，并能及时释放渲染所需的资源。
- **API错误检查**：OpenGL ES是一个严格的API规范，有着广泛的错误检查。有些错误仅限于开发过程中会发生，运行时并不能有效处理，但OpenGL ES的错误检查仍然会进行，这增加了驱动和发布版本应用的负担。Vulkan则使用一个精简的核心规范，不要求驱动实现运行时错误检查。对API的无效使用可能会引发渲染中断或应用崩溃。作为全周期错误检查的替代，Vulkan提供了一个框架，允许在应用程序和本机Vulkan驱动程序之间插入层驱动。这些层可用于实现错误检查、debug，以及任何发布版本应用中可以移除的工作。
- **渲染通道的抽象**：OpenGL ES中并没有渲染通道（render pass）对象这样的概念，但这样的概念对于像Mali这样的tile-based渲染器尤为重要。驱动需要在运行时推断哪些渲染指令构成了一个渲染通道，这类任务通常会占用一些时间，而启发式方法也并不准确。Vulkan则是围绕渲染通道概念构建的，并且还额外引入了子通道（subpass）的概念用于自动转译一个渲染通道中分片内（in-tile）的着色操作。这些显式编码可以完全去除运行时的启发式方法，也可以通过预构建来进一步降低运行时开销。
- **内存分配**：OpenGL ES使用客户端-服务器（C/S）内存模型。这种模型显式地划分了客户端（CPU端）资源和服务器（GPU）资源，并提供相应的函数实现数据在两者之间的转移。但这种模型有两种明显的缺点：1. 应用层无法直接分配和管理后台服务端的资源；2. 在客户端和服务器之间同步资源需要额外的开销，特别是API同步执行与实际的异步处理上存在冲突时。Vulkan是为现代硬件设计的，假定了CPU和GPU上某种程度的内存一致性。Vulkan API允许应用层直接控制一些内存资源（分配、更新等）。内存一致性使得缓冲的地址映射持久有效，避免了OpenGL ES中地址映射与取消映射的循环。
- **内存使用**：OpenGL ES使用一种重度的类型对象模型，将逻辑资源与物理内存紧密耦合。这使得用起来简便，但意味着存在大量的中间存储。Vulkan将不同的资源概念和其背后的物理内存做了区分，例如图片。这使得内存调度可以重复利用同一块物理内存来存储渲染管线中不同阶段的不同资源。这种区分设计可以通过循环使用同一块内存来降低上层应用在每帧中的内存占用。区分设计和内存可变性也会对驱动端优化施加一些限制，特别是像帧缓冲压缩这种改变内存布局的优化。

### 1.3 Vulkan的目标
虽然Vulkan提供了大量的优化手段，但同时也意味着将许多开发和维护的责任转交到了应用层开发者的手中。在使用Vulkan开发之前，需要思考清楚Vulkan带来的收益以及需要为此付出的代价。
Vulkan并不能带来直接的性能提升，暴露给Vulkan和OpenGL ES的硬件是相同的，它们所使用的渲染方法也大致相同。但Vulkan可以降低CPU的负载，间接支持了更高的GPU频率。
Vulkan最大的优势在于降低渲染逻辑中的CPU负载，这是通过简化API接口以及支持多线程实现的。Vulkan的第二个优势在于降低了应用的内存占用，这是通过帧内资源回收利用实现的，对RAM受限的设备有很大价值。
Vulkan最大的缺点在于将大量的职责转变到了应用层，包括内存分配、依赖管理、CPU-GPU同步。这是一把双刃剑，在细化控制粒度的同时，增加了风险。Vulkan的另一个巨大的弊端在于其极小的抽象层很容易暴露硬件之间的差异。例如OpenGL ES的依赖完全由驱动把控，所以能假定执行正确的操作；但Vulkan由应用层控制，有些在传统管线中运行良好的渲染通道可能在tile-base渲染中显得过于保守。

# 2. Vulkan可以做什么
Vulkan实际上是一组工具，可以通过很多方式完成开发任务。

### 2.1 图形
2D和3D图形是Vulkan API设计上最主要的用途，Vulkan被设计用于开发和构建硬件加速的图形应用。

### 2.2 计算
由于GPU的天然并行性，因而可利用GPU来执行各种GPGPU计算任务。

### 2.3 光线追踪
光追是一项可选的渲染技术，跨厂商的光追支持以Vulkan 1.2.162的扩展规范体现。主要由`VK_KHR_ray_tracing_pipeline`、`VK_KHR_ray_query`和`VK_KHR_acceleration_structure`组成。

### 2.4 视频
在Vulkan 1.2.175规范中有一个关于Vulkan Video的临时规范。Vulkan Video坚持Vulkan的理念，即为应用程序提供对视频处理调度、同步和内存利用的灵活、细粒度的控制。

### 2.5 机器学习
目前，Vulkan工作组正在研究如何使Vulkan成为展现现代GPU机器学习计算能力的一流API。

# 3. Vulkan规范
Vulkan 规范（通常称为 Vulkan 规范）是对 Vulkan API 如何工作的官方描述，最终用于决定什么是有效的 Vulkan 用法，什么是无效的。乍一看，Vulkan 规范似乎是一大堆枯燥乏味的文本，但它通常是开发时最有用的材料。

Khronos Group的[Vulkan Spec Registry](https://registry.khronos.org/vulkan/specs/)

# 4. 什么是SPIR-V
[SPIR-V Guide](https://github.com/KhronosGroup/SPIRV-Guide)
SPIR-V是图形着色阶段和计算内核之间的二进制中间表示。使用Vulkan时，应用层仍然可以使用高级着色语言（HLSL、GLSL）编写Shader，但当使用`vkCreateShaderModule`时需要传递其SPIR-V二进制表示。

### 4.1 SPIR-V接口和功能
Vulkan规范中有一整章定义如何使用SPIR-V接口。SPIR-V具有许多其他功能，而不仅仅是用于Vulkan。

### 4.2 SPIR-V编译器
- glslang：Khronos的参考前端，用于生成GLSL、HLSL和ESSL的SPIR-V。
- Shaderc：由 Google 托管的用于 Vulkan 着色器编译的工具、库和测试的集合。
- DXC：DirectXShaderCompiler也支持将HLSL编译为SPIR-V。

### 4.3 生态和工具
- SPIRV-Tools：Khronos SPIRV-Tools 项目提供了 C 和 C++ API 以及一个命令行界面来与 SPIR-V 模块一起工作。
- SPIRV-Cross：Khronos SPIRV-Cross 项目是一个实用的工具和库，用于在 SPIR-V 上执行反射并将 SPIR-V 反汇编回所需的高级着色语言。
- SPIRV-LLVM：Khronos SPIRV-LLVM 项目是一个支持 SPIR-V 的 LLVM 框架。它旨在包含 LLVM 和 SPIR-V 之间的双向转换。它还作为编译 SPIR-V 的 LLVM 前端编译器的基础。

# 5. 开发工具
### 5.1 Vulkan Layers
Layer 是增强 Vulkan 系统的可选组件。它们可以在从应用程序到硬件的过程中拦截、评估和修改现有的 Vulkan 功能。层作为库实现，可以使用 Vulkan Configurator 启用和配置。

- Khronos Layers：`VK_LAYER_KHRONOS_validation`是Khronos验证层。这是每个开发人员在调试 Vulkan 应用程序时的第一道防线。该验证层包括了“同步验证”、“GPU辅助验证”、“着色器打印”和“最佳实践警告”。
- Vulkan SDK layers：除了Khronos Layers，Vulkan SDK还包括一些额外的有用的独立layer，比如`VK_LAYER_LUNARG_api_dump`、`VK_LAYER_LUNARG_gfxreconstruct`等。
- 第三方layers

### 5.2 Debugging
- Arm Graphics Analyzer
- GAPID
- NVIDIA Nsight
- PVRCarbon
- RenderDoc
- GFXReconstruct

### 5.3 性能分析
- AMD Radeon GPU Profiler
- Arm Streamline Performance Analyzer
- Intel GPA
- OCAT
- PVRTune
- Qualcomm Snapdragon Profiler
- VKtracer

# 6. Vulkan Validation
### 6.1 Valid Usage
VU的官方定义是：为了在应用程序中实现明确定义的运行时行为而必须满足的一组条件。
Vulkan作为显式API的一个重要优势是，驱动不需要浪费时间检查输入的有效性。

### 6.2 Undefined Behavior
如果应用层提供了无效输入，则根据规范中的VU，其结果可能是未定义的行为。

### 6.3 Valid Usage ID
VUID 是为每个有效使用提供的唯一 ID。这提供了一种快速标示规范中有效用法的途径。

### 6.4 Validation Error Message
```
Validation Error: [ VUID-vkBindBufferMemory-memory-parameter ] Object 0: handle =
0x20c8650, type = VK_OBJECT_TYPE_INSTANCE; | MessageID = 0xe9199965 | Invalid
VkDeviceMemory Object 0x60000000006. The Vulkan spec states: memory must be a valid
VkDeviceMemory handle (https://registry.khronos.org/vulkan/specs/1.1-extensions/
html/vkspec.html#VUID-vkBindBufferMemory-memory-parameter)
```
- 首先需要注意的是开头的VUID：`VUID-vkBindBufferMemory-memory-parameter`
- `The Vulkan spec states`是对规范中VUID内容的引用
- `VK_OBJECT_TYPE_INSTANCE`告知了Vulkan对象类型（`VkObjectType`）
- `Invalid VkDeviceMemory Object 0x60000000006`告知了哪个句柄引发了错误

# 7. Vulkan开发
### 7.1 Loader
加载器负责将应用程序映射到 Vulkan Layer 和 Vulkan 驱动程序 (ICD)。

加载器有两种链接方式：直接和间接。
- 直接链接（编译期）：需要有一个已经构建的Vulkan Loader（静态库或动态库）。
- 间接链接（运行期）：使用动态符号查找（通过 `dlsym` 和 `dlopen` 等系统调用），应用程序可以初始化自己的调度表。

### 7.2 Layers
Layers作为共享库，由loader动态加载于loader和应用之间。启用Layer的两个要素是二进制文件位置和需要启用的Layer。要使用的Layer可以由应用程序显式启用，也可以通过告诉加载程序来隐式启用。

Vulkan SDK 包含一个Layer配置文档，该文档非常具体地说明了如何在每个平台上发现和配置Layer。Windows，Linux和macOS上可以使用Vulkan配置器 vkconfig 启用显式层和禁用隐式层。

### 7.3 物理设备信息查询
Vulkan中可查询的物理设备信息包括：设备属性（只读数据）、可用扩展（[Registry](https://registry.khronos.org/vulkan/#repo-docs)）、可用特性、限制和支持格式。

可用扩展：虽然默认情况下所有这些扩展项目都在 Vulkan 标头中找到，但如果未启用扩展，则使用扩展是未定义的行为。

限制：限制是具体实现所依赖的最小值、最大值和应用程序可能需要注意的其他设备特性。

支持格式：Vulkan 提供了许多具有多个 `VkFormatFeatureFlags` 的 `VkFormat`，每个 `VkFormatFeatureFlags` 都包含不同的 `VkFormatFeatureFlagBits` 的位掩码。

### 7.4 启用扩展
Vulkan中的扩展有两种：实例扩展和设备扩展。