layout: post
title: "Scroll和OffsetTopAndBottom的区别"
excerpt: "用一种类比的方式来理解Android的绘制原理"
tags: 
- Scroll,offSetTopAndBottom
- Android
categories:
- Android
---

在Android的滑动中,有画布和内容两个概念，分别对应了不同的方法和参数，比如scrollTo()，移动的是View中的内容，而View对应的画布(显示区域)没有发生变化。而offsetTopAndBottom()，移动的是画布，而内容的位置(相对于画布的坐标)并没有发生变化。

还有一点需要特别注意，子视图的画布，仅仅在父视图的画布范围内才可以显示。

{% img /images/scroll_and_offset/scroll_and_offset_01.png  [简单的类比理解]%}

相框就好比是一块画布，相框里面的内容是可以被看见的。

{% img /images/scroll_and_offset/scroll_and_offset_02.png  [简化的下拉刷新]%}

对于经典的下拉刷新控件，我们可以使用offSetTopAndBottom()，实现，也就是通过操作View的画布来实现。

思路大致如下，写一个自定义的控件PullToRefreshLayout,继承自ViewGroup,他包含两个子控件，头部header和内容ListView。

作为原生的ViewGroup，必须要确定子视图的大小---重写onMeasure()方法，确定子视图画布的大小---重写onLayout()方法。

**onMeasure的理解**

在PullToRefreshLayout中，你不仅需要测量出该控件的大小，还需要为其各个子控件测量出大小。


    @override
	protected void onMeasure(int widthMeasureSpec,int heightMeasureSpec){
		/**
		*这里的widthMeasureSpec和heightMeasureSpec是由父控件传递过来的，
		*/
	}

{% img /images/scroll_and_offset/scroll_and_offset_03.png  [典型ViewGroup的重写]%}

不同ViewGroup的继承类，会有不同的测量方法，如上图所示，对于bottom控件，在布局文件中的高度都是`android:layout_height="match_parent"`，但显示出不同效果。

对于LinearLayout，在测量完top之后，top的高度已经确定了，在测量bottom的时候，在确定其高度的时候，会将top的高度考虑进去，所以虽然bottom的高度模式是match_parent，但却不是全屏显示。

问题：如果说我并没有重写PullToRefreshLayout的onMeasure()方法会怎样？

	 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec,heightMeasureSpec);
	}
	该方法调用到View的onMeasure()方法，最后这个控件的测量高度和宽度都为0.


关于View的onMeasure()理解

ScrollView和RelativeLayout测量子控件时的不同方式。
假设这两个父控件的高度只有30px，而里面ImageView的高度模式是wrap_content,那么该图片在两种父控件中的显示效果是不同的。

对于RelativeLayout，图片可以完全显示，但是会缩小。

对于ScrollView，图片只显示30px的高度，但是不会缩放。

这是由于父控件在测量时会选择不同的测量模式，比如ScrollView，在测量时会将模式修改为`MeasureSpec.UNSPECIFIED`，也就是表明，不对子孩子的高度做任何限制，结合该ImageView，那么最终ImageView的测量高度就由图片的高度决定，所以，会比30px高。

而对于RelativeLayout，他在测量时做了限制，wrap_content和match_content都不会超过父控件的高度，除非明确指定了子控件的高度。

如果你是自定义的View，那么就必须重写onMeasure()方法，不然测量出来的宽高都为0。








**onLayout的理解**

如前所述，onLayout()的作用在于确定画布的大小,在正常情况下，画布的尺寸和内容尺寸是一致的，但是，好奇的你也可以将onLayout()重写得与onMeasure()无关，但一般不会这么做，因为你会把自己弄晕。

假设这样一种极端的情况。







