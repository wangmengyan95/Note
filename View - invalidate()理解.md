最近工作涉及到了很多animation相关的东西，频繁使用了`invalidate()`这个函数，所以花了点时间去理解这个函数背后到底发生了什么。

### Google doc
这个函数非常简单，一共三种形式

* `invalidate()` : Invalidate the whole view.
* `invalidate(Rect dirty)` : Mark the area defined by dirty as needing to be drawn.
* `invalidate(int l, int t, int r, int b)` : Mark the area defined by the rect (l,t,r,b) as needing to be drawn.

它的功能比较直白，就是重绘指定的区域，我只用到了第一种。

### Source Code
我看的是7.0.0_r24，源码在[这里](https://github.com/android/platform_frameworks_base/blob/android-7.0.0_r24/core/java/android/view/View.java)就能找到。

1. `view.invalidateInternal`
  
  可以发现三种形式的`invalidate`都指向了这个函数，所以它就是`invalidate`的起点。关于参数`boolean invalidateCache`是用来控制`view`的`drawingCache`相关的`flag`的，和`invalidate`关系不大。这里只关注`fullInvalidate`的情况。
  ```
  void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
      // Check whether we need to skip invaldiate
      ...

      if (Check view flags) {
          // Update view flags
          ...

          // Key part
          final AttachInfo ai = mAttachInfo;
          final ViewParent p = mParent;
          if (p != null && ai != null && l < r && t < b) {
              final Rect damage = ai.mTmpInvalRect;
              damage.set(l, t, r, b);
              // p is the parent of the view, normally it is a ViewGroup class
              p.invalidateChild(this, damage);
          }

          // Damage view receivew if necessary
          ...
      }
  }
  ```
    可以看到函数的逻辑比较清晰，在判断需要`invalidate`后，会更新相关的`flag`, 并且调用`p.invalidateChild(this, damage);`。

1. `viewGroup.invalidateChild`

  ```
  @Override
  public final void invalidateChild(View child, final Rect dirty) {
      ViewParent parent = this;

      final AttachInfo attachInfo = mAttachInfo;
      if (attachInfo != null) {
          // Animation and opaque related codes
          ...

          // Update view invalidate flags if necessary
          ...

          final int[] location = attachInfo.mInvalidateChildLocation;
          location[CHILD_LEFT_INDEX] = child.mLeft;
          location[CHILD_TOP_INDEX] = child.mTop;

          // Transform related codes 
          ...

          do {
              View view = null;
              if (parent instanceof View) {
                  view = (View) parent;
              }

              // Update animation and opaque flags
              ...

              parent = parent.invalidateChildInParent(location, dirty);


              // Transform related codes 
              ...

          } while (parent != null);
      }
  }

  ```
  在`invalidateChild`中，核心部分是一个do while的循环。我们会发现当前的`ViewGroup`不断调用`parent`的`invalidateChildInParent`。对于一般的`ViewGroup`来讲，它的`parent`还是`ViewGroup`。所以先来看`ViewGroup`的`invalidateChildInParent`。
    
