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

2. `viewGroup.invalidateChild`

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
  
3. `viewGroup.invalidateChildInParent`
  ```
  @Override
  public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
      // Normally we go to this branch sicne PFLAG_DRAWN is set
      if ((mPrivateFlags & PFLAG_DRAWN) == PFLAG_DRAWN ||
              (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID) {
          if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE)) !=
                      FLAG_OPTIMIZE_INVALIDATE) {
              // We change the coordinate from child index to parent index
              dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                      location[CHILD_TOP_INDEX] - mScrollY);

              if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                  dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
              }

              final int left = mLeft;
              final int top = mTop;

              if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                  if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                      dirty.setEmpty();
                  }
              }
              mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;

              location[CHILD_LEFT_INDEX] = left;
              location[CHILD_TOP_INDEX] = top;

              if (mLayerType != LAYER_TYPE_NONE) {
                  mPrivateFlags |= PFLAG_INVALIDATED;
              }

              return mParent;

          } else {
              mPrivateFlags &= ~PFLAG_DRAWN & ~PFLAG_DRAWING_CACHE_VALID;

              location[CHILD_LEFT_INDEX] = mLeft;
              location[CHILD_TOP_INDEX] = mTop;
              if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                  dirty.set(0, 0, mRight - mLeft, mBottom - mTop);
              } else {
                  // in case the dirty rect extends outside the bounds of this container
                  dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
              }

              if (mLayerType != LAYER_TYPE_NONE) {
                  mPrivateFlags |= PFLAG_INVALIDATED;
              }

              return mParent;
          }
      }

      return null;
  }
  ```
  首先`PFLAG_DRAWN`这个flag是在`view.draw()`里被设置的，对于一般`invalidate`的情况，`PFLAG_DRAWN`为1所以我们进入第一个分支。`FLAG_CLIP_CHILDREN`这个flag是通过设置`clipChildren`这个属性设置的。它用来控制`parent view`是否裁剪`child view`。默认情况这个flag的值为0。
  
  以这样一个例子说明计算的流程
  ```
  <FrameLayout
      android:id="@+id/parent"
      android:layout_marginLeft="20px"
      android:layout_marginTop="20px"
      android:layout_width="100px"
      android:layout_height="100px">
      <FrameLayout
          android:id="@+id/child"
          android:layout_marginLeft="30px"
          android:layout_marginTop="30px"
          android:layout_width="50px"
          android:layout_height="50px" />
  </FrameLayout>
  ```
  则调用 `invalidateChildInParent`, 开始时`location`为30，30, `dirty`为0，0，50，50，经过`offset`计算后，`location`为20，20, `dirty`为30，30，80，80。相当于将`dirty`区域的坐标由`child`的坐标系坐标转换为了以`parent`的坐标系坐标。
  根据android的`view hierarchy`，最后我们会调用`viewRootImpl.invalidateChildInParent()`方法。
  
4. `viewRootImpl.invalidateChildInParent`
  ```
  @Override
  public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
      checkThread();
      if (DEBUG_DRAW) Log.v(mTag, "Invalidate child: " + dirty);

      // Dirty area check
      if (dirty == null) {
          invalidate();
          return null;
      } else if (dirty.isEmpty() && !mIsAnimating) {
          return null;
      }

      // Update dirty area location
      if (mCurScrollY != 0 || mTranslator != null) {
          mTempRect.set(dirty);
          dirty = mTempRect;
          if (mCurScrollY != 0) {
              dirty.offset(0, -mCurScrollY);
          }
          if (mTranslator != null) {
              mTranslator.translateRectInAppWindowToScreen(dirty);
          }
          if (mAttachInfo.mScalingRequired) {
              dirty.inset(-1, -1);
          }
      }

      invalidateRectOnScreen(dirty);

      return null;
  }

  private void invalidateRectOnScreen(Rect dirty) {
      final Rect localDirty = mDirty;
      if (!localDirty.isEmpty() && !localDirty.contains(dirty)) {
          mAttachInfo.mSetIgnoreDirtyState = true;
          mAttachInfo.mIgnoreDirtyState = true;
      }

      // Add the new dirty rect to the current one
      localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
      // Intersect with the bounds of the window to skip
      // updates that lie outside of the visible region
      final float appScale = mAttachInfo.mApplicationScale;
      final boolean intersected = localDirty.intersect(0, 0,
              (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
      if (!intersected) {
          localDirty.setEmpty();
      }
      if (!mWillDrawSoon && (intersected || mIsAnimating)) {
          scheduleTraversals();
      }
  }
  ```
  这段代码比较直白，做了一些必要的检查和update dirty的坐标后，将dirty和现有的dirty取并集，如果发现dirty区域和`viewRootImpl`的位置有重叠的话，则通过`scheduleTraversals()`来开始重绘过程。
