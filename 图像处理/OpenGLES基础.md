# OpenGLES基础
#图像处理#

# 可编程着色器
## 顶点着色器
用来处理图形中的每个顶点，常见操作就是旋转、平移、投影等。
**特点：**
并行的逐顶点进行运算处理，也就是每个顶点数据都会执行一次顶点着色器里的处理逻辑。
**输入：**
1. 着色器程序：  描述顶点上执⾏操作的顶点着色器程序源代码（GLSL代码）或者可执行⽂文件。
2. 属性Attributes： 常常通过它输入顶点数据。
3. 统一变量uniform：不变数据类型，可以理解为常量。
4. 采样器：代表顶点着色器使用纹理的特殊统一变量类型。
![](OpenGLES%E5%9F%BA%E7%A1%80/58859DA6-E3B7-4055-8FA7-9C8E7D6131AF.png)

## 片段着色器
用来处理图形中每个像素点的颜色并最终填充到像素点上。
**特点：**
并行的逐像素进行运算处理，也就是你需要为每一个像素依次指定填充的颜色，也就是说每个像素点的填充都要执行一次片段着色器里的上色处理逻辑。
**输入：**
1. 着色器程序：描述⽚段上执⾏操作的片元着⾊器程序源代码/可执行⽂件。
2. 可变变量varying：从顶点着色器传过来的变量。
3. 统一变量uniform：不变数据类型，可以理解为常量。
4. 采样器：代表⽚元着色器使⽤纹理的特殊统一变量类型。
![](OpenGLES%E5%9F%BA%E7%A1%80/97BDE71C-32C9-46C7-B779-22A1B501D60A.png)

# OpenGL ES工作流程
![](OpenGLES%E5%9F%BA%E7%A1%80/0C4B0277-C454-4FF6-B0D9-9ED7A1E8772C.png)
同OpenGL类似，OpenGL ES 是 OpenGL的简化版本，它消除了冗余功能，我们能编程操作的就是顶点和片元着色器的输入和处理逻辑，片段着色器着色后下一阶段就是，逐片段的一系列操作，如下：
![](OpenGLES%E5%9F%BA%E7%A1%80/5232FC13-F0A0-4BFE-8736-CD6A6D3F5088.png)
**像素归属测试：**
这个测试确定帧缓存区中位置 **(Xw, Yw)** 的像素目前是不是归 **OpenGL ES** 所有。例如，如果⼀个显示 **OpenGL ES** 帧缓存区窗口的窗口被另外一个窗口所遮蔽，则窗⼝系统可以确定被遮蔽的像素不属于 **OpenGL ES** 上下文。从⽽完全不显示这些像素。虽然像素归属测试是 **OpenGL ES** 的⼀部分，但它不由开发人员控制，⽽是在 **OpenGL ES** 内部进行。
**裁剪测试：**
裁剪测试确定 **(Xw, Yw)** 是否位于作为 **OpenGL ES** 状态的一部分的裁剪矩形范围内。如果该⽚段位于裁剪区域之外，则被抛弃。
**深度测试：**
输⼊片段的深度值比较，确定⽚段是否应该被拒绝。
**混合：**
混合将新生成的⽚段颜⾊值与保存在帧缓冲区 **(Xw, Yw)** 位置的颜⾊值组合起来。
**抖动：**
抖动可用于最⼩化，因为使用有限精度在帧缓存区中保存颜色⽽产生的伪像。

# GLES语法
[OpenGL 着色语言 GLSL 语法介绍 - 简书](https://www.jianshu.com/p/43aaff0b6226)

# OpenGL ES处理图像示例
## 准备流程
### 1. 配置渲染图层CAEAGLLayer
![](OpenGLES%E5%9F%BA%E7%A1%80/D944EEA3-0170-4E49-9D86-5A3701A89F70.png)
### 2. 配置OpenGL ES上下文环境
![](OpenGLES%E5%9F%BA%E7%A1%80/E55E4768-EA13-41D0-9A5C-82FBEED43266.png)
### 3. 配置帧缓冲区和渲染缓冲区
![](OpenGLES%E5%9F%BA%E7%A1%80/5B5FA937-B273-49DC-B0FE-DDBD4F93C16A.png)
## 绘制流程
![](OpenGLES%E5%9F%BA%E7%A1%80/C9EEAAEA-5512-4872-BB65-2551FB55912E.png)
### 1. 配置着色器程序并传递数据
![](OpenGLES%E5%9F%BA%E7%A1%80/475F5016-A585-4372-9120-75783B444FA0.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/35C32A6C-2896-48F9-A0A3-5F67CECA8A39.png)
纹理坐标原点在左下角，openGL处理原点是在中心。
### 2. 渲染并输出到屏幕上
![](OpenGLES%E5%9F%BA%E7%A1%80/8F756B49-67C1-42A4-B42E-F0EDC5D9FED8.png)

## 相关数据结构
### BaseVertex
![](OpenGLES%E5%9F%BA%E7%A1%80/993E6515-65FE-414F-A525-1A48702CDCD3.png)
### Filter Shader Base
![](OpenGLES%E5%9F%BA%E7%A1%80/6656D335-9E35-45E6-9024-FE4E90A89BF9.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/1D566580-EA24-4A3B-9965-C62F72FF90C2.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/9432290E-39C9-4A69-B88E-AF70D8FB4EB1.png)
### Filter Shader Gray
![](OpenGLES%E5%9F%BA%E7%A1%80/2C54EB4B-133A-4237-8860-FB274114BE3E.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/DA918263-24C8-43E0-8104-32D24A276720.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/038A9FC2-3850-4589-A789-8949DCF10781.png)
### Filter Shader Mosaci
![](OpenGLES%E5%9F%BA%E7%A1%80/F21FB751-1F3E-42FD-BC8B-E3AB33D82B35.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/E874578D-0819-40CE-9590-6A5B5C4014D7.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/1B9B6B4C-6BD2-4208-8A93-CF1B1AD4FDAD.png)
### ShareProgram
![](OpenGLES%E5%9F%BA%E7%A1%80/6C157AB7-A187-4364-A672-A5F44DBAEAC0.png)
## GLHepler工具类
### 加载并编译着色器代码
![](OpenGLES%E5%9F%BA%E7%A1%80/44150B9E-B9A2-42FD-A10C-FB1E5DAC8489.png)
### 链接编译后的着色器并生成glProgram
![](OpenGLES%E5%9F%BA%E7%A1%80/5D50E68F-38A0-4294-BA2D-9490C5941B99.png)
### 输入顶点和纹理数据到glProgram的VSH对应Attribute上
![](OpenGLES%E5%9F%BA%E7%A1%80/63A7F0DA-1A98-4805-987D-6F5D4B7F6FDF.png)
### 输入纹理数据到glProgram的FSH对应纹理采样器上
![](OpenGLES%E5%9F%BA%E7%A1%80/8C5BDD1E-59E9-40CC-910A-F37ED2C3F8E2.png)
![](OpenGLES%E5%9F%BA%E7%A1%80/1C8382CF-9CE0-4D00-AE13-82A3CA984884.png)
