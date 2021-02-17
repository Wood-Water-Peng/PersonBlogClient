---
layout: post
title: "View相关的一些问题"
excerpt: "基于问题驱动"
tags: 
- View
- Android
categories:
- Android
comments: true
share: true
---

**问题：ViewRootImpl在执行`scheduleTraversals()`时，会调用`mHandler.getLooper().getQueue().postSyncBarrier()`插入一个同步消息屏障，这样做的目的是什么？**

> 同步屏障消息存在的目的是为了快速的去获取MessageQueue中的下一个异步消息。
> 
> ```
> MessageQueue.java
> next()方法
> if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
    ```            

<img src="/images/view/View绘制流程_invalidate流程.png">

在引入了vSync信号之后，traversal工作由Choreographer负责，他的内部有一个`FrameDisplayEventReceiver`用于接收底层的vSync信号

```
  @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame) {
        //收到底层的更新信号，步骤5
            //...
            Message msg = Message.obtain(mHandler, this);
          
            msg.setAsynchronous(true);
            //发送一个异步消息，步骤6
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        }
        
         @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame);
        }
        
         @UnsupportedAppUsage
    void doFrame(long frameTimeNanos, int frame) {
 		  //...
 		              //发送一个异步消息，步骤7
    		 doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    		  //...
    }
```
前提知识：

vSync的信号意思是GPU要准备刷新下一帧了，App是时候将最新的View信息刷新到缓存了，所以App在`doTraversals()`中会依次`onMeasure()->onLayout()->onDraw()`将新的数据刷新到GPU的缓存，后面由显示器显示出来。如果此时View的信息没有更新，那么显示器还是显示缓存的内容。注意：一般情况下显示器是16.6ms刷新一帧，那么假设Choreographer也是16.6ms收到一次vSync信号，那么`doTraversals()`就需要在16.6ms内完成（暂不考虑双缓存机制）。

同步消息屏障的作用？

```
tv_02.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                tv_02.setText("哈哈哈哈多");
                 Message message = Message.obtain();
//              message.setAsynchronous(true);
                mHandler.sendMessage(message);
            }
        });
         @Override
        public void handleMessage(@NonNull Message msg) {
	         //耗时操作
        }
```
当执行完`tv_02.setText("哈哈哈哈多");`后，实际上是走到了步骤3，Choreographer的callBack中多了一个`mTraversalRunnable`任务。这个任务并不会立刻执行，而是在等待下一个vSync信号。

然后你继续向队列中添加耗时的同步任务，假设此时MessageQueue中一共有两个消息，同步屏障消息和这个同步消息A。UI线程等待被唤醒。

此时vSync信号来临，走到步骤6，发送一个异步消息B,唤醒UI线程，MessageQueue执行`next()`方法，过滤掉A，直接执行B，然后走到步骤7，8,View的数据正常被刷新。如果没有这个屏障消息，会先执行A，他是一个耗时操作，那么`ViewRootImpl`的`doTraversals()`就会延迟进行，最新的View数据就不能及时刷新到GPU缓存。

这里需要注意的是，如果是耗时的异步消息，该消息会正常执行。




