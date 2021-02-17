---
layout: post
title: "View硬件加速知识点"
excerpt: "基于问题驱动"
tags: 
- View
- Android
categories:
- Android
comments: true
share: true
---

**1.有一个说法是RenderThread在渲染帧信息时，MainThread必须等待，这是什么原因？**

```
 //ThreadedRender.java
 
 int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
```
>该方法是一个jni调用，会唤醒RenderThread去同步帧信息，并绘制帧信息。在同步的过程中，MainThread肯定是会等待的，同步完成后是否唤醒MainThread，取决于同步的结果。如果同步完成后MainThread被唤醒了，那么它可以并行的去准备下一帧的信息。
>
>什么时候同步信息会返回false呢(false则MainThread等待)?
>
>RenderThread在同步帧信息的时候，会将DisplayList引用的Bitmap生成Open GL纹理上传到GPU。但是一个应用进程可以创建的Open GL纹理是有限制的，如果超出这个限制，某些Bitmap就不能作为Open GL纹理上传到GPU，那么就会返回false。
>
>
>
```
// If prepareTextures is false, we ran out of texture cache space   缓存使用完了
    return info.prepareTextures;
```
>在后续的过程中，会使用LRU算法删除旧的纹理，然后生成新的Open GL纹理上传到GPU。

[老罗的博客](https://blog.csdn.net/Luoshengyang/article/details/46281499)

**2.parent持有所有child的DisplayList，怎么理解？**

> DisplayList可以理解为一块缓存区，它里面的保存的是绘制数据，最终会交给底层。如果开启了硬加速,ViewRootImpl会首先发起一次数据更新操作，这是一个递归操作，从DecorView开始一直到叶节点。更新完成之后，DisplayList也完成了更新，DecorView节点中的DisplayList会被作为最终的绘制数据，层层解析。
> 

DisplayList结构图  

<img src="/images/view/DisplayList结构.png">

> DecorView持有的RenderNode会作为RootRenderNode的内容被
> ThreadRender使用。
> 假如更新了某个子View的DisplayList数据，那么他的直系Parent的会将View更新后的DisplayList写入自己的RenderNode内部缓存起来。

```
//draw(canvas,view,time)
//...
((RecordingCanvas) canvas).drawRenderNode(renderNode);

```
> parent通过将canvas交给child去完成绘制，然后将child的displayList写入自己的RenderNode中。这样parent和child的displayList就关联起来了。

[参考博客](https://www.jianshu.com/p/7bf306c09c7e)

**3.硬件加速对View的translation和scale等属性是如何实现快速支持的。**



13040949078

