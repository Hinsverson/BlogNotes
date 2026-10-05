# Metal总结

[Metal入门教程总结 - 阅读清单 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/inventory/1366/article/1192048)很好的总结
[GitHub - loyinglin/LearnMetal: Metal 入门教程](https://github.com/loyinglin/LearnMetal)
Metal2研发笔录_Mr_厚厚的博客-CSDN博客
[](https://blog.csdn.net/cordova/category_9467887.html)

[LearnOpenGL-CN](https://learnopengl-cn.readthedocs.io/zh/latest/)

# 重要对象理解
![](Metal%E6%80%BB%E7%BB%93/2AA15DE5-8D20-4C61-9F3F-2E93841821CA.png)
MTLDevice：代表GPU，通常使用MTLCreateSystemDefaultDevice获取默认的GPU。
MTLCommandQueue：由device创建，用于创建和组织MTLCommandBuffer，保证指令（MTLCommandBuffer）有序地发送到GPU， 每一帧都会产生一个MTLCommandBuffer对象，用于填放指令。
MTLCommandBuffer会提供一些encoder，对于一个commandBuffer，只有调用encoder的结束操作，才能进行下一个encoder的创建，同时可以设置执行完指令的回调。GPUs的类型很多，每一种都有各自的接收和执行指令方式，在MTLCommandEncoder把指令进行封装后，MTLCommandBuffer再做聚合到一次提交里。有如下的Encoder：
	* 编码绘制指令的MTLRenderCommandEncoder
	* 编码计算指令的MTLComputeCommandEncoder
	* 编码缓存纹理拷贝指令的MTLBlitCommandEncoder。
MTLRenderPassDescriptor：是一个轻量级的临时对象，里面存放较多属性配置，供MTLCommandBuffer创建MTLRenderCommandEncoder对象用。

### MTLRenderPassDescriptor
[【Metal2研发笔录：RenderPassDescriptor和VertexDescriptor】 - 知乎](https://zhuanlan.zhihu.com/p/92840318)
MTLRenderPassDescriptor和MTLVertexDescriptor是Metal引擎框架中比较重要的两个类，分别用来配置渲染末期渲染结果的去向和渲染初期顶点数据的映射传送。

一个MTLRenderPassDescriptor对象包含一组attachments，作为rendering pass产生的像素的目的地。MTLRenderPassDescriptor还可以用来设置目标缓冲来保存rendering pass产生的可见性信息。另外除了用于保存颜色信息的color attachments，MTLRenderPassDescriptor还分别包含一个depthAttachment和stencilAttachment，用于保存深度缓冲数据和模板缓冲数据。

MTLRenderPassDescriptor的colorAttachments、depthAttachment、stencilAttachment对应的类都是继承自MTLRenderPassAttachmentDescriptor。
![](Metal%E6%80%BB%E7%BB%93/50C2C67F-0110-4F5D-BF38-D9712F84C667.png)
