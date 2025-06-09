---
title: OpenGL入门
tags:
  - OpenGL
---

## 1. 基础知识 ##

OpenGL 是一个图形API， 包含了一系列可以操作图形、图像的函数。然而，OpenGL本身并不是一个API，它仅仅是一个由Khronos组织制定并维护的规范(Specification)。

### 立即渲染模式/核心模式 ###
早期使用立即渲染模式，绘制方便，但不够灵活，效率也高。
现如今使用核心模式，更灵活，效率更高，但学习更难。

### 扩展 ###
OpenGL的一大特性就是对扩展(Extension)的支持，通过扩展可以对未来某些特性进行支持，不支持的仍然可以使用旧的方式实现。

### 状态机 ###
OpenGL自身是一个巨大的**状态机**(State Machine)：一系列的变量描述OpenGL此刻应当如何运行。OpenGL的状态通常被称为OpenGL上下文(Context)。我们通常使用如下途径去更改OpenGL状态：**设置选项**，**操作缓冲**。最后，我们使用当前OpenGL上下文来渲染。

### 对象 ###
OpenGL库是c写的，但c的一些语言结构不易被翻译到其它的高级语言，因此OpenGL开发的时候引入了一些抽象层。“对象(Object)”就是其中一个。简单来说就是结构体，结构体成员与函数指针，一同构成对象的属性和方法。
```c
struct object_name {
    float  option1;
    int    option2;
    char[] name;
};
```

OpenGL Context是一个大的结构体：
```c
// OpenGL的状态
struct OpenGL_Context {
    ...
    object* object_Window_Target;
    ...     
};
```
```c
// 创建对象
unsigned int objectId = 0;
glGenObject(1, &objectId);
// 绑定对象至上下文
glBindObject(GL_WINDOW_TARGET, objectId);
// 设置当前绑定到 GL_WINDOW_TARGET 的对象的一些选项
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH, 800);
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// 将上下文对象设回默认
glBindObject(GL_WINDOW_TARGET, 0);
```

创建对象，保存引用（`objectId`），上下文绑定，设置对象。。。