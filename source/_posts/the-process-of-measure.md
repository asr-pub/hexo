---
title: Android measure流程
date: 2016-12-18
tags:
  - Android
---

### 概述

View 的绘制过程，与艺术家作画的过程有点类似。艺术家首先确定作品的尺寸（measure），然后决定在画布上的什么位置作画（layout），最后拿起画笔作画（draw），当然我只是把该过程给简单化了。

当 Activity 获得焦点时，那么就要绘制它的布局了。Android Framework 将会处理绘制的过程，但是 Activity 需要提供布局的根节点。我们新建一个 Hello World 工程，用 Hierarchy View 来查看详细的布局。我们可以看到，Android 会为我们额外的增加一些布局，图中最左边的就是根节点了，传说中的 DecorView，关于绘制的一切都是从它开始的。

![此处输入图片的描述][1]

DecorView 继承自 FrameLayout，FrameLayout 继承自 ViewGroup，ViewGroup 实现了 ViewParent 接口，而 ViewRootImpl 实现了 ViewParent 接口，我们的切入点就是 ViewRootImpl 这个类，View 绘制的代码就是从该类开始的。
<!--more-->

![此处输入图片的描述][2]

在 ViewRootImpl 中关于绘制的部分是从 performTraversals() 开始的。

### performTraversals

刚看到这个函数的时候，感觉很长，相当长，我数了一下，有 657 行，这对于初学者来说，并不友好。不过结合网上的资料和自己的分析能力，我们总能标出一下关键的代码，所以接下来，我会把其他无关、冗余的代码给注释掉，只保留与绘制有关的代码。

```jva
private void performTraversals() {
	...
	if (!mStopped) {
		int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);  // 1
		int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);
		performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);       
	}
  
	if (didLayout) {
		performLayout(lp, desiredWindowWidth, desiredWindowHeight);
		...
	}

	if (!cancelDraw && !newSurface) {
		if (!skipDraw || mReportNextDraw) {
			if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
				for (int i = 0; i < mPendingTransitions.size(); ++i) {
					mPendingTransitions.get(i).startChangingAnimations();
				}
				mPendingTransitions.clear();
			}

			performDraw();
		}
	} 
	...
}
```

在 performTraversals() 内部分别执行了 performMeasure(), performLayout() 和 performDraw()，在这3个方法中，又分别调用 measure(), layout(), draw()。

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

我们看到 `performMeasure()`，里面执行了 `mView.measure`，mView 就是 DecorView 了，所以我们可以得知，View 的测量过程是从 DecorView 开始的，方法`measure()`在 View 中。`performMeasure()`的参数是类型为`MeasureSpec`的`childWidthMeasureSpec`与`childHeightMeasureSpec`。`MeasureSpec`类型的值由两部分组成，分别是 `mode`和`size`，关于`MeasureSpec`的知识点比较基础，网上的相关资料也很多，这里不再赘述。在 Android 中，每个 View 都有两个 `MeasureSpec` 类型的参数，这两个参数表示该 View 的父亲对它测量大小的一个约束，那么这两个参数是如何计算出来的呢？计算公式是这样的：`ParentView.MeasureSpec + ChildView.LayoutParams = ChildView.MeasureSpec`，该公式只是具体代码的一个抽象而已。那么顶级 DecorView 是由谁决定的呢？

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

ViewRootImpl.java 文件中的该方法决定 DecorView 的 MeasureSpec，其中 windowSize 是屏幕的尺寸。

### measure 

在这里我给大家介绍一款 Chrome 插件 ———— [Android SDK Search][3]，它能在我们浏览 Android 官方文档的时候，可以让我们直接跳到该文档相关的源码中。

#### View measure

我们看到 View 中的 `measure`方法，我只留下了关键代码：

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
	······
	if (forceLayout || needsLayout) {
		······
		if (cacheIndex < 0 || sIgnoreMeasureCache) {
			······
			onMeasure(widthMeasureSpec, heightMeasureSpec);
			······
		} else {
			·····
		}
		······
	}
	······
}
```

我们从`measure()`的注释中可以得到以下信息：
1. 该方法是`final`类型的，所以子类不能重写该方法
2. View 实际的测量工作在`onMeasure(int, int)`进行，所以要在子类中重写该方法

当然，在 View 中也有默认的实现：一般情况下默认使用 background 的宽高作为 View 的宽高。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}
```

```java
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

在`getDefaultSize()`中，`UNSPECIFIED`一般用于系统内部测量中，所以 View 的宽高取决于`specSize`，而该数值来自于父亲，在下面我会介绍到，此时的`specSize`就是父亲剩余的可用宽高，那么这不就是`match_parent`的效果吗？由此我们得知，View 的默认实现是不支持`wrap_content`的，为了支持`wrap_content`，我们得在自定义 View 中重写`onMeasure()`。

```java
// 代码来自 任玉刚 《Android开发艺术探索》
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSpecSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);
    int heightSpecSize = MeasureSpec.getSize(heightMeasureSpec);

    if (widthSpecMode == MeasureSpec.AT_MOST && heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWidth, mHeight);
    } else if (widthSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(mWidth, heightSpecSize);
    } else if (heightSpecMode == MeasureSpec.AT_MOST) {
        setMeasuredDimension(widthSpecSize, mHeight);
    }
}
```
在上述代码中，我们只处理`wrap_content`中的部分，非`wrap_content`的部分，使用父亲的值就行，其中`mWidth`与`mHeight`是在代码中灵活计算的，具体的实现可以参考 TextView 与 ImageView 中的相关代码。在`onMeasure()`中必须要调用`setMeasuredDimension()`来设置视图的测量宽高。

#### ViewGroup measure

```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    final int size = mChildrenCount;
    final View[] children = mChildren;
    for (int i = 0; i < size; ++i) {
        final View child = children[i];
        if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
        }
    }
}
```
在`measureChildren()`中，遍历所有的子视图，对属性不为`GONE`的视图调用`measureChild()`，传递的3个参数分别是子视图，父亲的`widthMeasureSpec`和`heightMeasureSpec`，我们下面看看`measureChild`做了哪些操作。

```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```

在该函数中体现了我们之前写的一个公式`ParentView.MeasureSpec + ChildView.LayoutParams = ChildView.MeasureSpec`，这个函数不就是计算出父 View 传递给子 View 的 `MeasureSpec`么，然后把计算结果扔给子 View，让子 View 自己进行测量。子 View`MeasureSpec`的计算过程中，`getChildMeasureSpec`是做了最多工作的一步，该函数略长。

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
该函数根据文档中的注释，可以很容易的看出这个过程是如何计算出来的，如下图所示：

![|center](/images/the-process-of-drawing-view-3.png)

### 参考资料

1. https://android.googlesource.com/platform/frameworks/base.git/+/master/core/java/com/android/internal/policy/DecorView.java


  [1]: http://odc50o546.bkt.clouddn.com/the-process-of-drawing-view-1.png
  [2]: http://odc50o546.bkt.clouddn.com/the-process-of-drawing-view-2.png
  [3]: https://chrome.google.com/webstore/detail/android-sdk-search/hgcbffeicehlpmgmnhnkjbjoldkfhoin