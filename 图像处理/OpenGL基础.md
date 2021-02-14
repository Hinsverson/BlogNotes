# OpenGL基础
#图像处理#


# 基本概念理解
## 上下文和状态机
上下文（Context）是一个openGL环境，它是一个状态机，本质上通过OpenGL API操作和设置都是对这个庞大的context状态机里的某个状态或者对象进行设置。
**状态机一般特点：**
1. 它有记忆的能力，能够记住自己当前的状态。
2. 它可以接收输入，根据输入的内容和自己的状态，修改自己的状态，并且可以得到输出。
3. 当它进入某个特殊的状态（停机状态）的时候，它不再接收输入，停止工作。

切换上下文往往意味着很大的开销，当不同的渲染模块需要完全独立的状态管理时，可以考虑在不同线程上使用不同的上下文，上下文间共享纹理、缓冲区等资源。

## 管线化
OpenGL的图形渲染流程类似于一条流水线，每个任务之间有先后顺序，因为显卡在处理数据的时候是严格按照固定顺序处理的，这种处理方式称做管线化。

## 顶点数组VAO）和顶点缓冲区（VertexBuffer）
顶点数组（VAO）：内存中存储绘制顶点的数组。
顶点缓冲区（VertexBuffer）：显存里存储绘制顶点的一块数据区域，因为是直接存在显存中，所以性能上比存在内存中好。

## 着色器程序（Shader）
整个OpenGL渲染流程里可编程的部分，OpenGL在绘制之前，需要着色器程序来告诉它如何处理顶点数据，如何进行逐像素的着色。
OpenGL本身自带一套编译环境，处理着色器程序Shader时也需要进行编译链接等步骤最终生成glProgram。glProgram同时就包含了顶点、片段着色器的处理逻辑。

OpenGL里：顶点、片段、像素、几何、细分着色器
OpenGL ES里：只有顶点、片段着色器。
![](OpenGL%E5%9F%BA%E7%A1%80/7E0D7D94-9263-48E2-92A3-0A028DD7CBC7.png)

### 顶点着色器（VertexShader）
用来处理图形中的每个顶点，常见操作就是旋转、平移、投影等。
特点是：
并行的逐顶点进行运算处理，也就是每个顶点数据都会执行一次顶点着色器里的处理逻辑。

### 片段着色器（FragmentShader）
用来处理图形中每个像素点的颜色并最终填充到像素点上。
特点：
并行的逐像素进行运算处理，也就是你需要为每一个像素依次指定填充的颜色，也就是说每个像素点的填充都要执行一次片段着色器里的上色处理逻辑。

## 纹理贴图（Texture Mapping）
将纹理图⽚附着到绘图的图像上，一个纹理就是一副用来贴到三角形或多边形上的图片。

## 缓冲区交换（SwapBuffer）
渲染缓冲区映射的是系统资源，比如屏幕，所以如果将图像直接渲染到窗口对应的缓冲区则可以将图像显示到屏幕上。
如果屏幕只有一个缓冲区，就会导致在绘制的过程中屏幕进行了刷新，显示了不完整的图像，所以一般都会有2个缓冲区，显示在屏幕上的称为屏幕缓冲区，没有显示的叫离屏缓冲区，每一个缓冲区完成渲染后通过交换实现图像显示。

## 视口
![](OpenGL%E5%9F%BA%E7%A1%80/E26EA3CB-25E8-4486-B458-FFF4FE90FF35.png)

## 坐标系
![](OpenGL%E5%9F%BA%E7%A1%80/51C4DB73-A136-44B7-8A77-7391252889CC.png)
![](OpenGL%E5%9F%BA%E7%A1%80/04EA3936-2668-48C6-B299-05A460AFD454.png)
### 物体坐标系
它与特定的物体相关联，每个物体都有自己特定的坐标系。不同物体之间的坐标系相互独立，当物体发生移动或者旋转，物体坐标系发生相同的平移或者旋转，物体坐标系和物体之间运动同步，相互绑定。（举个栗子：记得科目一有道题目是说人在运动的时候，方向的随意性。每一个人都有自己的一个坐标系，人在运动的过程中，每个人的方向都不一样，都是按照自己的物体坐标系运动，不受他人的影响。）
### 世界坐标系
世界坐标系是一个特殊的坐标系，它建立了描述其他坐标系所需要的参考系。也就是说，可以用世界坐标系去描述其他所有坐标系或者物体的位置。而且世界坐标系是固定不变的。
### 惯性坐标系
惯性坐标系是为了简化世界坐标系到惯性坐标系的转化而产生的。惯性坐标系的原点与物体坐标系的原点重合，惯性坐标系的轴平行于世界坐标系的轴。引入了惯性坐标系之后，物体坐标系转换到惯性坐标系只需旋转，从惯性坐标系转换到世界坐标系只需平移。、

## OpenGL渲染流程中的坐标转换
![](OpenGL%E5%9F%BA%E7%A1%80/1661FD3A-A5F5-421A-8973-7644457B41C6.png)
OpenGL只定义了裁剪坐标系、规范化设备坐标系和屏幕坐标系，然而物体坐标系)世界坐标系和摄像机坐标系都是为了方便用户设计而自定义的坐标系。 其中：
模型变换、视变换，投影变换，这些变换可以由用户根据需要自行指定，这些内容在顶点着色器中完成。
透视除法、视口变换，这两个步骤是OpenGL自动执行的，在顶点着色器处理后的阶段完成。 

## OpenGL中的变换
### 模型变换
我们将一个物体放在一个场景中，应该就相当于模型变换。模型变换是在世界坐标系中进行的，物体模型的中心定位在坐标系的中心处，通过对物体模型执行平移（glTranslate）、缩放（glScale）、旋转（glRotate）等操作，来调整物体模型在世界坐标系中的位置。
### 视变换
![](OpenGL%E5%9F%BA%E7%A1%80/6CBB1E45-9183-465C-8F12-245BECB07639.png)
将相机置于三角架上，让它对准三维物体，它相当于OpenGL中调整视点的位置，即视变换。经过模型变换，物体的坐标都处于世界坐标系中，视变换就是确定了场景中物体的视点位置和方向。在实际拍摄物体时，我们可以保持物体的位置不动，调整相机距离物体的距离和角度，这就相当于视点变换，我们也可以保持相机的固定位置，将物体远离相机，这就相当于模型转换。其实可以看出，在OpenGL中，以逆时针旋转物体就相当于以顺时针旋转相机。视变化就可以理解为：**根据摄像机的位置和角度，然后观察世界坐标系下的物体。 在上图的左边部分，是将摄像机放进世界坐标系，右边部分则是将世界坐标系内的物体模型变换到摄像机空间（即是视图空间）**。
### 投影变换
![](OpenGL%E5%9F%BA%E7%A1%80/9F5B9165-B647-4ED6-9567-5A15F9836C0A.png)
经过模型变换和视变换之后，物体已经处在了场景中所希望存在的位置上，此时我们选择相机镜头并调整相机焦距，使得三维物体投影在二维胶片上，它相当于OpenGL中把三维模型投影到二维屏幕上的过程，即OpenGL的投影变换。 OpenGL有2种投影方式：
**透视投影：**
即离视点近的物体大，离视点远的物体小，远到极点即为消失。
**正投影：**
又叫平行投影，无论物体距离相机多远，投影后的物体大小尺寸不变。

## OpenGL顶点、片元、图元转换和光栅化
### 顶点—> 图元
几何顶点被组合为图元（点，线段或多边形），然后图元被合成片元，最后片元被转换为帧缓存中的象素数据。
### 图元 —> 片元
![](OpenGL%E5%9F%BA%E7%A1%80/6586DBF0-863B-49AC-8391-BB0BBA8A4834.png)
不同类型的图元决定了对顶点的连接使用方式，最后呈现不同的图形。

**通常应该考虑GL_TRIANGLE_STRIP或者GL_TRIANGLE_FAN**
优点：提供运算性能并节省带宽，更少的顶点意味处理的顶点和顶点数据从内存传输到图形卡的速度更快。

![](OpenGL%E5%9F%BA%E7%A1%80/92B955D2-1364-4A38-879B-DD03912A5BE9.png)
图元转换为片元一般分为几步：图元被适当的裁剪，颜色和纹理数据也相应作出必要的调整，相关的坐标被转换为窗口坐标。最后，光栅化将裁剪好的图元转换为片元。

### 片元—> 像素
1. 象素所有权（ownership）检测
2. 裁剪检测
3. Alpha检测
4. 模版检测
5. 深度检测
6. 融合
7. 抖动
8. 逻辑操作
### 理解光栅化
将一个图元转变为一个二维图象（其实只是布满平面，没有真正的替换帧缓存区）的过程。二维图象上每个点都包含了颜色、深度和纹理数据。将该点和相关信息叫做一个 **片元（fragment）**。（这就是片元和像素之间的关键区别，虽然两者的直观印象都是的像素，但是片元比像素多了许多信息，在光栅化中纹理映射之后图元信息转化为了像素）

# OpenGL核心渲染流程
[OpenGL基础渲染 - 简书](https://www.jianshu.com/p/c0e71b4c1bf4)
![](OpenGL%E5%9F%BA%E7%A1%80/59E9EC56-3E14-4A4A-87DB-857D95C18D28.png)
![](OpenGL%E5%9F%BA%E7%A1%80/46ABCAA2-CCE6-4F39-93A0-044CCCC74E0A.png)
![](OpenGL%E5%9F%BA%E7%A1%80/1AE6E58B-E8EC-4BE8-B45F-57EC89CF4DF8.png)
OpenGL中可选步骤：**细分着色器（可选）****几何着色器（可选）**
OpenGL ES中没有上述步骤。

## OpenGL 内部渲染流程
![](OpenGL%E5%9F%BA%E7%A1%80/1AE6E58B-E8EC-4BE8-B45F-57EC89CF4DF8%202.png)

## 渲染客户端-服务端
管线上半部分是客户端，下半部分是服务端。就 **OpenGL** 而言，客户端是存储在 **CPU** 存储器中的，驱动程序将渲染命令与数据组合起来发给服务端执行。服务端和客户端在功能上是异步的。客户端不断的将数据和命令组合在一起送入缓冲区，缓冲区再发送到服务端执行。

## 着色器
上图中最大的框代表的是 **顶点着色器** 和 **片元着色器**。着色器是使用GLSL编写的程序。
### 顶点着色器
顶点着色器处理从客户端输入的数据，用数学运算来计算光照效果、位移、颜色值等，有几个顶点，顶点着色器就要执行几次。
上图中的 **图元组合（Primitive Assembly）**框图意在说明3个顶点已经组合在了一起。
### 片元着色器
片元着色器来计算片元的最终颜色（尽管在下一个阶段（逐片元的操作）时可能还会改变颜色一次）和它的深度值。在这里我们会使用纹理映射的方式，对顶点处理阶段所计算的颜色值进行补充。如果我们觉得不应该继续绘制某个片元，在片元着色器中还可以终止这个片元的处理，这一步叫做片元的**丢弃（discard）**。
### 顶点的着色器和片元着色器之间的区别：
**顶点着色（包括细分和几何着色）**决定了一个图元应该位于屏幕的什么位置，而**片元着色**使用这些信息来决定某个片元的颜色应该是什么。

## 三种向OpenGL 着色器传递渲染数据的⽅法
### 属性传值（Attributes）
就是对⼀个顶点都要作出改变的数据元素。实际上，顶点位置本身就是一个属性.。属性可以是浮点类型，整型，布尔类型等。
### Uniform值
通过设置 **Uniform** 变量就紧接着发送一个图元批次处理命令。**Uniform** 变量实际上可以无限次的使⽤。 设置一个应用于整个表⾯面的单个颜色值，还可也是一个时间值。
### 纹理（Textture）
对纹理进行采样和筛选。纹理数据的作用不仅仅是表现图形。很多图形文件格式都是以无符号字节形式对颜色分量进行存储的，但我们仍然可以设置浮点纹理。这就是说，任何大型浮点数据块（例如消耗资源很大的函数的大型查询表）都可以通过这种方式传递给着色器。

## 正背面剔除（解决渲染前后镂空问题）
![](OpenGL%E5%9F%BA%E7%A1%80/EBCD212A-F16B-4C35-8C4A-616EF22235C0.png)
在默认的情况下，**OpenGL**认为具有**逆时针方向**环绕的多边形是 **正面**的，可通过如下方式修改：
```c
/* 参数：GL_CW | GL_CCW
GL_CCW：表示传入的mode会选择逆时针为前向
GL_CW：表示顺时针为前向。
默认：GL_CCW。逆向时针为前向。
*/
glFrontFace(GL_CW);
```

![](OpenGL%E5%9F%BA%E7%A1%80/5D957060-D0C3-44CA-870E-771CF55EF852.png)
OpenGL内部进行正背面的判断，依据是由给的顶点顺序和设置绕序以及观察者的位置决定，比如当观察者在右侧，设置的绕序为逆时针，输入的顶点顺序为1、2、3，则右边三角形为正面，左边三角形为反面。开启和进行正背面剔除可以如下设置：
```c
// 可用值为 GL_FRONT、GL_BACK 或 GL_FRONT_AND_BACK
void glCullFace(GL_BACK); 
glEnable(GL_CULL_FACE); //开启
glDisable(GL_CULL_FACE); //关闭
```

## 深度测试（解决渲染点远近问题）
深度：在openGL坐标系中，像素点的 **Z** 坐标距离观察者的距离。观察者可能放在坐标系的任何位置，那么，就不能简单的说 **Z** 数值越大或越小，就是越靠近观察者。如果观察者在Z轴的正方向，**Z** 值大的靠近观察者，如果是在Z轴的反方向，则 **Z** 值小的更靠近观察者。

深度缓冲区：存储每个像素点的深度值的一块内存区域。
深度的概念主要给予3D物体层次感，如果没有深度，那么先绘制一个近处的物体，再绘制一个远处的物体，如果没有深度那么近处的物体就会被后绘制的远处物体覆盖。

深度缓冲区和颜色缓冲区是一一对应并关联的。
首先，使用glClear(GL_DEPTH_BUFFER_BIT)，把所有像素的深度值设置为最大值。
如果启用了深度缓冲区，在绘制每个像素之前，OpenGL会把它的深度值和已经存储在这个像素的深度值进行比较。如果，新像素深度值 **<** 原先像素深度值，则新像素值会取代原先的；反之，新像素值被遮挡，它的颜色值和深度将被丢弃。

```c
// 申请一个颜色缓冲区和一个深度缓冲区：
glutInitDisplayMode(GLUT_DOUBLE | GLUT_RGBA | GLUT_DEPTH | GLUT_STENCIL);

// 清除颜色缓冲区和深度缓冲区，默认值是1.0，表示最大的深度值
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// 开启和关闭
glEnable(GL_DEPTH_TEST);
glDisable(GL_DEPTH_TEST);

//指定深度测试判断规则
glDepthFunc(GLenum mode)
/* mode参数
GL_ALWAYS 总是通过测试 
GL_NEVER 总是不通过测试 
GL_LESS 当前深度值 < 存储的深度值时通过 
GL_EQUAL 当前深度值 = 存储的深度值时通过 
GL_LEQUAL 当前深度值 <= 存储的深度值时通过 
GL_GREATER 当前深度值 > 存储的深度值时通过 
GL_NOTEQUAL 当前深度值 != 存储的深度值时通过 
GL_GEQUAL 当前深度值 >= 存储的深度值时通过
*/
```

## z-fighting（z冲突、闪烁）问题
当深度值精确度很低时，引起ZFighting现象，说明两个靠的很近的物体某些点确定谁在前，谁在后时出现了歧义（屁游戏场景中经常出现的闪烁现象）。

**解决方法：**
1. 在第二次绘制时，插入一个少量的偏移。
2. 使用 **glPolygonOffset** 函数调节片段的深度值，使得深度值偏移而不产生重叠。
3. 使用更高位数的深度缓冲区，通常使用的深度缓冲区是24位的，现在有一些硬件使用使用32位的缓冲区，使精确度得到提高。

## 裁剪测试
一种提高渲染性能的方法，将 OpengGL渲染限制在窗口中一个较小的矩形区域（剪裁框）中，裁剪测试是片元可见性判断的第一个附加测试。默认情况下，剪裁框与窗口同样大小，并且不会进行裁剪测试。
![](OpenGL%E5%9F%BA%E7%A1%80/2F5C57FD-7C87-4433-81F8-D2EBABBD9F74.png)
``` c
// 1.先设置整体部分清屏颜色为蓝色
glClearColor(0.0f, 0.0f, 1.0f, 0.0f);
glClear(GL_COLOR_BUFFER_BIT);
    
// 2.现在剪裁为中间红色矩形部分   
glClearColor(1.0f, 0.0f, 0.0f, 0.0f);// (1)设置裁剪区颜色为红色
glScissor(50, 50, 300, 200);// (2)设置裁剪尺寸
glEnable(GL_SCISSOR_TEST);// (3)开启裁剪测试
glClear(GL_COLOR_BUFFER_BIT);// (4)开启清屏，执行裁剪
    
// 2.最后裁剪一个绿色的小矩形
// 设置清屏颜色为绿色
glClearColor(0.0f, 1.0f, 0.0f, 0.0f);
glScissor(100, 100, 200, 100);
glClear(GL_COLOR_BUFFER_BIT);
    
// 关闭裁剪测试
glDisable(GL_SCISSOR_TEST);
    
// 进行缓冲区交换并刷新
glutSwapBuffers();
// glFlush(void void) 立即刷新
```

## 混合
通常OpenGL渲染时会把颜色值放在颜色缓冲区，每个像素的深度值放在深度缓冲区。
**如果深度测试被关闭：**
新的颜色值会简单地覆盖颜色缓冲区中已经存在的其他值。
**当深度测试被打开**
``` c
glEnable(GL_BLEND); //打开混合
```
保留2者中深度值Z更小的，同时但如果打开了 **混合功能**，那么下层的颜色值就不会被直接清除而是一个由新旧颜色混合的值，源颜色和新的目标颜色的组合方式是由混合方程式控制，默认情况如下：

最终颜色 = (源颜色 * 源混合因子S) + (目标颜色 * 目标混合因子D)

### 统一设置RGBA混合因子
```c
glBlendFunc(GLenum s, GLenum D); // s是源混合因子，D是目标混合因子
```
![](OpenGL%E5%9F%BA%E7%A1%80/C20FB7A2-AA07-4DA3-9FCD-ED7F8D60C3CE.png)
> 其中f=min(As, 1 - Ad)  

**注意：**
**GL_CONSTANT_COLOR**，**GL_ONE_MINUS_CONSTANT_COLOR**，**GL_CONSTANT_ALPHA**，
**GL_ONE_MINUS_CONSTANT_ALPHA**都允许在混合方程式中引入一个常量混合颜色，初始默认为黑色（0.0f, 0.0f, 0.0f, 0.0f），可以用下面函数修改：
```c
void glBlendColor(GLclampf red ,GLclampf green ,GLclampf blue ,GLclampf alpha );
```

### 分开设置RGB和A的混合因子
```c
void glBlendFuncSeparate(GLenum srcRGB, GLenum dstRGB, GLenum srcAlpha, GLenum dstAlpha);
```

### 设置混合方程式
```c
void glBlendEquation(GLenum mode); 
```
![](OpenGL%E5%9F%BA%E7%A1%80/914CF7F0-F244-4CF1-BA8D-70260F4DDF88.png)

## 抗锯齿
消除图元之间的锯齿状边缘，使用混合功能来混合片段的颜色，也就是把像素的目标颜色与周围像素的颜色进行混合。所以在启用混合功能并选择正确的混合函数以及混合方程式之后，可以选择调用 **glEnable** 函数对点、直线和多边形进行抗锯齿处理。
```c
glEnable(GL_POINT_SMOOTH);       // Smooth out points
glEnable(GL_LINE_SMOOTH);          // Smooth out lines
glEnable(GL_POLYGON_SMOOTH); // Smooth out polygon edges
```

# 综合案例
[OpenGL基础变化综合练习实践总结 - 简书](https://www.jianshu.com/p/ce3b51b8f168)

# 纹理相关
https://www.jianshu.com/p/23af8d5e427e
[OpenGL 图形库使用（五） —— 纹理 - 简书](https://www.jianshu.com/p/d861577901c8)

## 设置像素打/解包方式
```c
glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
```
全局设置状态，和纹理对象无关。一般用GL_UNPACK_ALIGNMENT，值为1，标示按1字节边界对齐，紧密打包。

GL_UNPACK开头的影响调用glTexImage2D、glTexImage3D、glTexSubImage2D、glTexSubImage3D时数据从内存中解包的方式。
GL_PACK开头的影响调用glReadPixels时数据打包到内存的方式，对纹理的载入没有任何影响。

**注意：如果不设置的话glTexImage2D载入纹理数据时默认是GL_UNPACK_ALIGNMENT=4。**

## 生成纹理对象
```c
void glGenTextures (GLsizei n, GLuint *textures);
```
指定纹理对象的数量n 和一个指针*textures，这个指针指向一个无符号整型数组（由纹理对象标识符填充）。

## 绑定/删除纹理到纹理目标
```c
// 参数target： GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D
// 参数texture：需要绑定的纹理对象
void glBindTexture (GLenum target, GLuint texture);
// 删除
void glDeleteTextures (GLsizei n, const GLuint *textures);
```
一旦纹理对象绑定到一个纹理目标（比如GL_TEXTURE_2D）之后所有目标对应的纹理加载和纹理参数设置只影响当前绑定的纹理对象。

## 测试纹理对象有效性
```c
// 如果texture是一个以前已经分配的纹理对象名，则返回 GL_TRUE，否则返回 GL_FALSE。
GLboolean glIsTexture(GLuint texture)
```

## 设置纹理参数
### 纹理过滤方式
根据一个拉伸或收缩的纹理贴图计算颜色片段的过程称为**纹理过滤（Texture Filitering）**。使用 **OpenGL** 的纹理参数函数，可以同时设置放大和缩小过滤器。这两种过滤器的参数名分别是 ：**GL_TEXTURE_MAG_FILTER** 
 **GL_TEXTURE_MIN_FILTER**。
我们可以为它们从两种基本的纹理过滤器： 
**GL_NEAREST（邻近过滤）** 
**GL_LINEAR（线性过滤）** 

**邻近过滤（GL_NEAREST）和线性过滤（GL_LINEAR）的区别**
当在一个很大的物体上应用一张低分辨率的纹理时（纹理被放大了，每个纹理像素都能看到）
![](OpenGL%E5%9F%BA%E7%A1%80/7E510A7F-1954-41E6-81C5-C5762622C487.png)
```c
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_HEAREST);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_HEAREST);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MAG_FILTER,GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_MIN_FILTER,GL_LINEAR);
```

### 纹理环绕方式
正常情况下，是在 0.0到1.0的范围内指定纹理坐标，使它与纹理贴图中的纹理单元形成映射关系。如果纹理坐标落在这个范围之外，**OpenGL** 则根据当前纹理环绕模式（Wrapping Mode）处理这个问题，不同环绕方式的效果如下：
![](OpenGL%E5%9F%BA%E7%A1%80/1651F20C-3C4C-4CC2-A95F-E1BE793DF138.png)
```c
/* 纹理环绕方式
参数1：纹理维度。
GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D
参数2：为S/T坐标设置模式。
GL_TEXTURE_WRAP_S、GL_TEXTURE_T、GL_TEXTURE_R,针对s,t,r坐标
参数3：wrapMode，环绕模式。
GL_REPEAT、GL_CLAMP、GL_CLAMP_TO_EDGE、GL_CLAMP_TO_BORDER
(1) GL_REPEAT： OpenGL 在纹理坐标超过1.0的⽅向上对纹理理进⾏重复;
(2) GL_CLAMP：所需的纹理单元取自纹理边界或TEXTURE_BORDER_COLOR.
(3) GL_CLAMP_TO_EDGE：环绕模式强制对范围之外的纹理坐标沿着合法的纹理单元的最后一⾏或者最后一
列来进行采样。
(4) GL_CLAMP_TO_BORDER：在纹理坐标在0.0到1.0范围之外的只使⽤边界纹理单元。边界纹理单元是作为围绕基本图像的额外的行和列，并与基本纹理图像⼀起加载的。
*/
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAR_S,GL_CLAMP_TO_EDGE);
```

## 多级渐远纹理（Mipmap）
假设有一个包含着上千物体的大房间，每个物体上都有纹理。有些物体会很远，但其纹理会拥有与近处物体同样高的分辨率。由于远处的物体可能只产生很少的片段，OpenGL ES 从高分辨率纹理中为这些片段获取正确的颜色值就很困难，因为它需要对一个跨过纹理很大部分的片段只拾取一个纹理颜色。在小物体上这会产生不真实的感觉，更不用说对它们使用高分辨率纹理浪费内存的问题了。OpenGL ES 使用一种叫做多级渐远纹理(Mipmap)的概念来解决这个问题，它简单来说就是一系列的纹理图像，后一个纹理图像是前一个的二分之一。多级渐远纹理背后的理念很简单：距观察者的距离超过一定的阈值，OpenGL ES会使用不同的多级渐远纹理，即最适合物体的距离的那个。由于距离远，解析度不高也不会被用户注意到。同时，多级渐远纹理另一加分之处是它的性能非常好。
![](OpenGL%E5%9F%BA%E7%A1%80/F6C31BEF-AE17-415F-BB06-3F595AF5EBD6.png)
**常见的错误：将放大过滤的选项设置为多级渐远纹理过滤选项之一**
这样没有任何效果，因为多级渐远纹理主要是使用在纹理被缩小的情况下的：纹理放大不会使用多级渐远纹理，为放大过滤设置多级渐远纹理的选项会产生一个GL_INVALID_ENUM错误代码。
```c
/* 只有minFilter 等于以下四种模式，才可以生成Mip贴图
   GL_NEAREST_MIPMAP_NEAREST具有非常好的性能，并且闪烁现象非常弱
   GL_LINEAR_MIPMAP_NEAREST常常用于对游戏进行加速，它使用了高质量的线性过滤器
   GL_LINEAR_MIPMAP_LINEAR 和GL_NEAREST_MIPMAP_LINEAR 过滤器在Mip层之间执行了一些额外的插值，以消除他们之间的过滤痕迹。
   GL_LINEAR_MIPMAP_LINEAR 三线性Mip贴图。纹理过滤的黄金准则，具有最高的精度。
 */
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, minFilter);
if(minFilter == GL_LINEAR_MIPMAP_LINEAR ||
   minFilter == GL_LINEAR_MIPMAP_NEAREST ||
   minFilter == GL_NEAREST_MIPMAP_LINEAR ||
   minFilter == GL_NEAREST_MIPMAP_NEAREST)
    
    //加载Mip,纹理生成所有的Mip层
    //参数：GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D
    glGenerateMipmap(GL_TEXTURE_2D);

```

## 纹理加载/替换
载入纹理数据到绑定的纹理对象上。
```c
/* 载入纹理到绑定的纹理对象上。
target：纹理维度GL_TEXTURE_1D、GL_TEXTURE_2D、GL_TEXTURE_3D。
Level：指定所加载的mip贴图层次。⼀一般我们都把这个参数设置为0。
internalformat：每个纹理理单元中存储多少颜⾊色成分。（从读取像素图时获得）
width、height、depth 参数：指加载纹理理的宽度、⾼高度、深度。==注意!==这些值必须是 2的整数次⽅方。(这是因为OpenGL 旧版本上的遗留留下的⼀一个要求。当然现在已经可以⽀支持不不是 2的整数次⽅方。但是开发者们还是习惯使⽤用以2的整数次⽅方去设置这些参数。)
border 参数：允许为纹理理贴图指定⼀一个边界宽度。
format 参数：像素数据的数据类型（GL_UNSIGNED_BYTE，每个颜色分量都是一个8位无符号整数）
type 参数：
data 参数：指向纹理图像二进制数据的指针。
*/
void glTexImage1D (GLenum target, GLint level, GLint internalformat, GLsizei width, GLint border, GLenum format, GLenum type, const GLvoid *pixels);

void glTexImage2D (GLenum target, GLint level, GLint internalformat, GLsizei width, GLsizei height, GLint border, GLenum format, GLenum type, const GLvoid *pixels);

void glTexImage3D (GLenum target, GLint level, GLint internalformat, GLsizei width, GLsizei height, GLsizei depth, GLint border, GLenum format, GLenum type, const GLvoid *pixels);
```

替换一个纹理图像要比直接使用 glTexImage 重新加载一个新纹理快得多。
```c
/*
绝大部分参数都与 glTexImage 函数的参数准确地对应。
xOffset、yOffset 和 zOffset 
参数指定了在原来的纹理贴图中开始替换纹理数据的偏移量。
width、height 和 depth 
参数指定了“插入”到原来那个纹理中的新纹理的宽度、高度和深度。
*/
void glTexSubImage1D(GLenum target,GLint level,GLint xOffset,GLsizei width,GLenum
  format,GLenum type,const GLvoid *data);

void glTexSubImage2D(GLenum target,GLint level,GLint xOffset,GLint yOffset,GLsizei
  width,GLsizei height,GLenum format,GLenum type,const GLvoid *data);

void glTexSubImage3D(GLenum target,GLint level,GLint xOffset,GLint yOffset,GLint
  zOffset,GLsizei width,GLsizei height,GLsizei depth,Glenum type,const GLvoid * data);
```

## 从颜色缓冲区直接读取转化为纹理
一维和二维纹理也可以从指定的颜色缓冲区加载数据，而不需要外部图片二进制数据。可以从颜色缓冲区读取一幅图像，并通过下面这两个函数将它作为一个新的纹理使用。
```c
// 读取哪个颜色源缓冲区是通过 glReadBuffer 函数设置的
void glReadBuffer(GLenum mode); //指定读取的缓存
void glWriteBuffer(GLenum mode) //指定写⼊的缓存

// 读取颜色缓冲区数据到绑定的纹理对象中
// 并不存在 glCopyTexImage3D，因为我们无法从 2D 颜色缓冲区获取3D数据
void glCopyTexImage1D(GLenum target,GLint level,GLenum
  internalformt,GLint x,GLint y,GLsizei width,GLint border);
void glCopyTexImage2D(GLenum target,GLint level,GLenum
  internalformt,GLint x,GLint y,GLsizei width,GLsizei
  height,GLint border);
```

