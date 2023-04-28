早期实现采用稀疏八叉树在GPU上存储场景，但GPU上查询稀疏八叉树的效率比较低，更高效的方案是采用三维纹理存储。

先通过Geometry Shader将三角形变换到top, left, front其中一个面上（根据面法线信息）。

接着过一遍片元着色器，在三维纹理中存储漫反射颜色、法线和自发光信息。

为了显示体素效果，需要在屏幕空间内还原三维纹理的数据。先通过深度图重建各像素的世界坐标；然后将相机转换到体素空间中；最后从相机发出一条射线，沿着射线不断步进来读取三维纹理中的颜色。

计算光照则需要从光源的反射阴影贴图（RSM）开始，将RSM中可见的纹素转换到体素空间中，以步进形式采样光照（cone tracing）。采样diffuse受光信息是对球面进行16次采样，采样reflect信息则是沿着反射方向采样。对于每次采样，射线沿着采样方向步进42次，如果没有碰到任何物体，则采样SkyColor。

各向异性的体素存储了随方向变化的体素颜色。具体而言，从上一级纹理中获取相邻的8个颜色值，并沿着正负XYZ方向积分得到。