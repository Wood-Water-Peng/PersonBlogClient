---
layout: post
title: "View绘制流程"
excerpt: "基于问题驱动"
tags: 
- View
- Android
categories:
- Android
comments: true
share: true
---

<img src="/images/view/View绘制流程_02.png">

在支持硬件加速后，View的绘制逻辑多了一条分支，这条分支主要是对DisplayList的处理。

注意View的`draw(canvas,parent,drawingTime)`在递归调用中所处的位置。ViewGroup调用该方法完成child的绘制，同时这里也是处理硬件加速场景的地方。根据流程图中所示，如果走硬件加速流程，那么只是递归的去更新DisplayList。如果View需要重建DisplayList，那么会执行他的draw()，即真正的绘制。

>场景一：如下图：假设我点击tv_02后更新tv_02的文本。

<img src="/images/view/demo_01.jpg">

```
//View.updateDisplayListIfDirty()

  if (renderNode.hasDisplayList()
                    && !mRecreateDisplayList) {
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;
                dispatchGetDisplayList();

                return renderNode; // no work needed
            }
```
对于tv_01来说，这段代码就直接return，因为它自己的缓存是valid的。

对于tv_02来说，会跳过，因为setText()会invalidate(),`PFLAG_INVALIDATED`会生效，在tv_02的parent调用`draw(canvas,parent,drawingTime)`时，会走这段流程

```
//View.draw(canvas,parent,drawingTime)

if (hardwareAcceleratedCanvas) {
            // Clear INVALIDATED flag to allow invalidation to occur during rendering, but
            // retain the flag's value temporarily in the mRecreateDisplayList flag
            mRecreateDisplayList = (mPrivateFlags & PFLAG_INVALIDATED) != 0;
            mPrivateFlags &= ~PFLAG_INVALIDATED;
        }
```

所以，`mRecreateDisplayList=true`

最终tv_02会调用自己的`draw(canvas)`方法。



参考链接：

[View的硬件渲染](https://sharrychoo.github.io/blog/android-source/graphic-choreographer)

[Android应用程序UI硬件加速渲染环境初始化过程分析](https://blog.csdn.net/luoshengyang/article/details/45769759)

