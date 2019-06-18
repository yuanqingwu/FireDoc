
# 渲染流程
顶点数据(Vertices) > 顶点着色器(Vertex Shader) > 图元装配(Assembly) > 几何着色器(Geometry Shader) > 光栅化(Rasterization) > 片元着色器(Fragment Shader) > 逐片元处理(Per-Fragment Operations) > 帧缓冲(FrameBuffer)。

再经过双缓冲的交换(SwapBuffer)，渲染内容就显示到了屏幕上。
图元装配：往土讲就是把图形放置到坐标系中。
光栅化：将图形投影到屏幕上，把图形栅格化成一个个的像素点。一个像素点也就是一个片元。
逐片元处理：填充颜色，在渲染的时候是基于像素分片处理的，会涉及到纹理和光影处理等
