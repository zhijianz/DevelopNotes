> 资源框架学习笔记，记录学习过程中的一些想法和感悟，学习完之后再进行系统的梳理

# Android 应用程序资源管理器创建过程分析

> 来源于老罗的博客[《Android应用程序资源管理器（Asset Manager）的创建过程分析》](http://blog.csdn.net/luoshengyang/article/details/8791064)

就像老罗在学习计划中介绍的那样，可以将Android的资源体系分成Resource/Asset两种类型，而实际上就算是Resource访问资源的操作也是通过桥接到AssetManager上实现的大体结构如[资源类结构](/ResourcesClassDiagram.pu)

在资源管理结构中比较重要的几个类的关系图如[AssetManager Structure Class Diagram](/AssetManagerStructure.pu)
