# Instrument使用笔记


[关于Instruments-Leaks工具使用的归纳总结   - 简书](https://www.jianshu.com/p/895bfbce4bf4)
1. 先Analyze静态分析可疑点，再使用Instruments的Leaks动态具体分析泄露点。
2. 开启并runapp，leak部分泄露点会标红，然后点击面板下方可以查看引用计数详细信息，以及引用环。使用calltree定位代码位置。、


[Instruments学习之Allocations - 简书](https://www.jianshu.com/p/b617f16acb7f)
Allocations显示了内存分配的整体情况，看这里分析内存爆增。

[instrument Time Profiler总结 - 简书](https://www.jianshu.com/p/21d29be26479)
分析运行时的函数执行时间，但应当注意要跑真机并且release环境

[Instruments学习之Core Animation - 简书](https://www.jianshu.com/p/7f128f03081f?from=timeline&isappinstalled=0)
[Instruments学习之Core Animation学习 -  开发者知识库](https://www.itdaan.com/blog/2017/06/02/f8539eae7e7014063bdb870e79689b72.html)
主要就是用来定位离屏渲染，包括：
	* 定位并减少Color Blended Layers，方法是少使用透明性颜色，不要关闭UI控件的opaque 属性为false（默认关闭）。
	* 定位是否正确使用layer的shouldRasterize，对于对于需要经常变动的内容不要开启这个属性，比如tableviewcell，对于耗费资源比较多的静态内容（设置阴影）开启后可以缓存提高性能。
	* 定位图片大小是否正确显示，也就是image size和imageView size不匹配的情况，在展示高清大图时可以先解压缩绘制一个可view大小相同的图片，再异步渲染，避免大图的解压缩内存开销。