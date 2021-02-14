# Metal总结
#图像处理#

[Metal入门教程总结 - 阅读清单 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/inventory/1366/article/1192048)
[GitHub - loyinglin/LearnMetal: Metal 入门教程](https://github.com/loyinglin/LearnMetal)
Metal2研发笔录_Mr_厚厚的博客-CSDN博客
[](https://blog.csdn.net/cordova/category_9467887.html)

[LearnOpenGL-CN](https://learnopengl-cn.readthedocs.io/zh/latest/)

### MTLRenderPassDescriptor
[【Metal2研发笔录：RenderPassDescriptor和VertexDescriptor】 - 知乎](https://zhuanlan.zhihu.com/p/92840318)

一个MTLRenderPassDescriptor对象包含一组attachments，作为rendering pass产生的像素的目的地。MTLRenderPassDescriptor还可以用来设置目标缓冲来保存rendering pass产生的可见性信息（光照可见性等用法，也就是自定义GBuffer的用法）。

另外除了用于保存颜色信息的color attachments，MTLRenderPassDescriptor还分别包含一个depthAttachment和stencilAttachment，用于保存深度缓冲数据和模板缓冲数据。

MTLRenderPassDescriptor的colorAttachments、depthAttachment、stencilAttachment对应的类都是继承自MTLRenderPassAttachmentDescriptor。
![](Metal%E6%80%BB%E7%BB%93/50C2C67F-0110-4F5D-BF38-D9712F84C667.png)

![](Metal%E6%80%BB%E7%BB%93/2AA15DE5-8D20-4C61-9F3F-2E93841821CA.png)