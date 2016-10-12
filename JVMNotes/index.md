> 关于 Java 虚拟机的学习内容


<!-- toc orderedList:0 -->

- [缘由](#缘由)
- [内容结构](#内容结构)
	- [JVM Memory Moduel & GC](#jvm-memory-moduel-gc)
	- [JVM Execution Subsystem](#jvm-execution-subsystem)
	- [Compile & Code Optimization](#compile-code-optimization)
	- [Efficient Concurrence](#efficient-concurrence)

<!-- tocstop -->

# 缘由

近段时间一直在研究 Android 插件化项目的开发。在整个尝试的过程中遇到各种各样预料中的难题，类加载器的问题算是其中比较头疼的地方。所以在被折磨的过程中就萌生了系统学习 Android虚拟机 这一块内容的想法，虽然它和 Java虚拟机 并不对等，但是仍然打算从 Java虚拟机入手，在保证对虚拟机设计实现有一定的理解和接触之后再去学习Dalvik和ART的内容。

先写这篇导读的目的是为了保证整个学习过程条理清晰地稳步执行，同时也是为了方便日后进行快速的回顾。整篇导读的内容会伴随学习的深入不断地完善。

# 内容结构

![JVM 思维导图](<./Imgs/Java Virtual Machine.png>)

该部分的内容专注于 JVM 的学习，学习资料来源于[《深入理解Java虚拟机》](<http://item.jd.com/11252778.html>)。上面的结构图参考本书的目录对于整个将要学习的内容进行划分和总结，在学习的过程会不断的进行完善。

## JVM Memory Moduel & GC

这部分的内容包含内存模型和垃圾回收两个部分的内容。JVM 启动之后划分出不同功能职责的内存，这些内存有着各自的特点和特定的服务为对象以及对应的生命周期，这些东西都会在这个部分进行详细的介绍。另外一点，JVM 为我们提供了内存分配管理的同时，也同样接管了垃圾回收的工作。所以这里同样会涉及到垃圾回收部分的内容。

## JVM Execution Subsystem

## Compile & Code Optimization

## Efficient Concurrence
