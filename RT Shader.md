## 1. 工程配置

启用DX12的配置步骤：
- Project Settings --> Player --> Other Settings
- Other Settings --> Rendering --> 取消勾选`Auto Graphics API for Windows`
- Other Settings --> Rendering --> Graphics API for Windows --> 添加DX12并上调至列表首位 --> 依据提示重启Editor
- Other Settings --> Rendering --> 取消勾选`Static Batching`

## 2. 思维流程

流程为：
- 初始化光追加速结构
- 初始化或更新RenderTexture用于存储输出画面
- 判断硬件是否支持光追
- 判断相机和参数是否发生变动，归零收敛计数
- 判断场景是否变动，构建光追加速结构
- 配置RayTracing Shader参数
- 派发RayTracing Shader
- 拷贝渲染结果
- 将当前参数标记为“上一帧”

## 3. 详细流程

#### 3.1 初始化光追加速结构
1. 创建`RayTracingAccelerationStructure.Settings`。
2. 配置settings的`rayTracingModeMask`属性，该掩码表明哪些类型的场景物体将被纳入到光追加速结构中。选项包括`Nothing`、`Static`、`DynamicTransform`、`DynamicGeometry`和`Everything`，我们希望将所有类型的场景物体都纳入加速结构，所以我们配置该属性为`RayTracingModeMask.Everything`。
3. 配置settings的`managementMode`属性，该属性决定了光追加速结构的更新方式，它的两个选项为`Manual`和`Automatic`，我们希望由Unity自动维护场景的光追加速结构，所以我们将其设置为`ManagementMode.Automatic`。
4. 配置settings的`layerMask`属性，该掩码表明哪些层级的场景物体将被纳入到光追加速结构中。我们希望将所有场景物体都添加到加速结构中，所以将掩码设置为前8位全1，也就是255。
5. 由settings创建`RayTracingAccelerationStructure`。

#### 3.2 初始化或更新RenderTexture用于存储渲染结果
1. 如果相机分辨率发生变动，则重新创建RenderTexture
2. 创建`RenderTextureDescriptor`。
3. 配置descriptor，我们需要配置`dimension`、`width`、`height`、`depthBufferBits`、`volumeDepth`、`msaaSamples`、`vrUsage`、`graphicsFormat`和`enableRandomWrite`。我们不需要深度信息，所以`depthBufferBits`设置为0、`volumeDepth`设置为1；我们也不需要MSAA，所以`msaaSamples`设置为1；最后，我们需要能并行地写入光追结果，所以需要启用随机写入，将`enableRandomWrite`设置为`true`。
4. 由descriptor创建RenderTexture。
5. 归零收敛计数。

#### 3.3 判断硬件是否支持光追
Unity提供了`SystemInfo.supportsRayTracing`属性用于检查当前平台是否支持硬件光追。如果当前设备不支持硬件光追，则打印一条提示信息并回退到默认渲染。

#### 3.4 构建光追加速结构
通过之前创建的`RayTracingAccelerationStructure`，可以直接调用其上的`Build()`方法构建场景的光追加速结构。实际上，如果场景并没有变动，可以不每帧重新构建加速结构。方便起见，我们先选择每帧都重新构建，对于不涉及动画形变的场景来说，构建光追加速结构的开销并不大。

#### 3.5 配置RayTracing Shader的参数
我们需要为RayTracing Shader做以下配置：
- 指定光线命中物体时所执行的pass。
- 配置光线弹射次数。
- 配置光追加速结构。
- 配置相机属性，包括镜头缩放和画面宽高比。
- 配置环境光照贴图。
- 配置收敛计数和帧计数。
- 配置用于存储渲染结果的RenderTexture。

#### 3.6 RayTracing Shader
我们需要在RayTracing Shader中处理光线生成（ray generation  shader）和光线未命中（ray miss shader）。

**Ray generation shader**：通过调用`TraceRay()`生成任意数量的光线。调用时需要同时指定一个用户自定义的ray payload结构作为参数。也可以在该着色器中调用`CallShader()`来调用一些callable shader。

#### 4.1 RayTracing Shader中的Mesh采样
传统的光栅化管线中，顶点着色器（vertex shader）通过专门的顶点语义（POSITION，TEXCOORD0等）定义输入的顶点信息，GPU将顶点输入流转换为顶点属性存入GPU的寄存器中。整个过程都是不透明的，由硬件厂商实现。但对于光追来说，我们需要手动从Mesh中读取顶点信息并解码顶点属性。

光线和三角形的求交由硬件加速结构完成，根据交点的位置不同，`any hit shader`和`closest hit shader`会被调用。HLSL提供的`PrimitiveIndex()`可用于获取当前三角形在BLAS中的编号，使用该编号可以读取相应三角形的三个顶点的数据，并根据交点位置对顶点属性进行插值。

需要注意的是：
- Mesh采样只可用于材质shader中的`any hit shader`和`closest hit shader`。
- 光栅化过程和光追过程的Mesh采样是一致的，不存在额外的数据拷贝。

**Unity中的参数访问**
在Unity内的HLSL中，可以通过引入`UnityRayTracingMeshUtils.cginc`来获取顶点流数组和Mesh的索引缓冲及属性缓冲数据。其中有一些帮助函数用于方便地获取顶点数据：
- `uint3 UnityRayTracingFetchTriangleIndices(uint primitiveIndex)`：读取指定三角形的三个顶点的索引。
- `floatN UnityRayTracingFetchVertexAttributeN(uint vertexIndex, uint attributeType)`：其中N=2,3,4，读取指定顶点的顶点属性。需要注意的是，如果实际数据的维数小于函数签名中的维数（N），则多出来的维度会自动补零。
- `bool UnityRayTracingHasVertexAttribute(uint attributeType)`：检查指定的顶点属性是否存在。

**顶点数据插值**
调用`any hit shader`或`closest hit shader`后，硬件固定函数会将相应的ray payload信息和交点的重心坐标值作为着色器的输入数据。`float2 barycenterics`的两个值分别对应v1和v2的权重，因此三个顶点的权重系数应为：
`float3 barycentricCoords = float3(1.0 - attribs.barycentrics.x - attribs.barycentrics.y, attribs.barycentrics.x, attribs.barycentrics.y);`
