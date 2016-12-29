最近工作涉及到了很多animation相关的东西，频繁使用了`invalidate()`这个函数，所以花了点时间去理解这个函数背后到底发生了什么。

### Google doc
这个函数非常简单，一共三种形式

* `invalidate()` : Invalidate the whole view.
* `invalidate(Rect dirty)` : Mark the area defined by dirty as needing to be drawn.
* `invalidate(int l, int t, int r, int b)` : Mark the area defined by the rect (l,t,r,b) as needing to be drawn.

它的功能比较直白，就是重绘指定的区域，我只用到了第一种。

### Source Code
我看的是7.0.0_r24，源码在[这里](https://github.com/android/platform_frameworks_base/blob/android-7.0.0_r24/core/java/android/view/View.java)就能找到。

1. `invalidateInternal`
  
  可以发现三种形式的`invalidate`都指向了这个函数，所以它就是`invalidate`的起点。关于最后两个参数`boolean invalidateCache`是用来控制`view`的`drawingCache`相关的`flag`的，和`invalidate`关系不大。这里只关注`fullInvalidate`的情况。
