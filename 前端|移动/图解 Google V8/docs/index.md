V8 是 Google 基于 C++ 编写的开源高性能 JavaScript 与 WebAssembly 引擎，主要的应用包括Chrome浏览器以及Node.js。得益于Chrome浏览器的市场占有率以及Chromium阵营的不断强大，V8已经成为了当今最主流的JavaScript引擎。

但很多前端开发人员对 V8 的理解还停留在表面，只是单纯地使用 JavaScript 和调用 Web API，并不了解 V8 这个“黑盒”内部是如何工作的，项目出现问题时，也只能是“头疼医头，脚疼医脚”，没有系统的解决策略；想要系统学习V8时，也不知道从何处着手，不能迅速抓住V8的核心知识要点。

因此，我们邀请了李兵，带来第二季课程《图解 Google V8》。在这个课程中，他将完整地梳理V8的核心知识体系，通过大量图片演示，深入浅出地讲解 V8 执行 JavaScript 代码的底层机制和原理。

通过学习这门课程，你不仅可以了解完整的 V8 编译流水线，还能通过对 V8 工作机制的学习，搞懂JavaScript语言的核心特性，进而从根源解决程序上的问题，加快 JavaScript 的执行速度。

# V8知识图谱

![](https://static001.geekbang.org/resource/image/0e/10/0ea66b6ea8045a7b5eb80b38fa2d3b10.jpg)

# 模块介绍

本课程包括三个模块，分别是 JavaScript 设计思想篇、V8 编译流水线篇、事件循环和垃圾回收篇。

**JavaScript 设计思想篇**，关注 JavaScript 的设计思想，讨论它背后的核心特性，以及V8是是怎么实现这些特性的。

**V8 编译流水线篇**，带你分析 V8 的编译流水线所涉及到的具体知识点，同时也会穿插讲解一些内存分配相关的内容，因为函数调用、变量声明、参数传递或者函数返回数值都涉及到了内存分配。

**事件循环和垃圾回收篇**，深入到 V8 的心脏事件循环系统中，学习 V8 是如何实现JavaScript 单线程执行的。同时，关注垃圾回收问题，打通 V8 分配内存和回收数据的整个链路，掌握系统排查问题的方法。