Recycler是RecyclerView中负责管理ViewHolder缓存和复用的内部类，它是RecyclerView非常重要的组成部分。

RecyclerView.Recycler的源代码在[这里](https://github.com/android/platform_frameworks_support/blob/master/v7/recyclerview/src/android/support/v7/widget/RecyclerView.java#L5144)。

Recycler内部实现了一个多级缓存系统，负责存取ViewHoler。我们分层讨论Recycler是怎样存取ViewHolder的。

##第一级缓存
第一级缓存对应的数据结构是mAttachedScrap和mChangedScrap这两个链表。

```
private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    if (viewHolder.isInvalid() && !viewHolder.isRemoved() &&
            !mRecyclerView.mAdapter.hasStableIds()) {
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        // 存入第一级缓存
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}

void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
            throw new IllegalArgumentException("Called scrap view with an invalid view."
                    + " Invalid views cannot be reused from scrap, they should rebound from"
                    + " recycler pool.");
        }
        // 存入mAttachedScrap
        holder.setScrapContainer(this, false);
        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        // 存入mChangedScrap
        holder.setScrapContainer(this, true);
        mChangedScrap.add(holder);
    }
}

ViewHolder getChangedScrapViewForPosition(int position) {
    // If pre-layout, check the changed scrap for an exact match.
    final int changedScrapSize;
    if (mChangedScrap == null || (changedScrapSize = mChangedScrap.size()) == 0) {
        return null;
    }
    // find by position
    for (int i = 0; i < changedScrapSize; i++) {
        final ViewHolder holder = mChangedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }

    ......
}

ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();

    // Try first for an exact, non-invalid match from scrap.
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }

    ......
}
```

通俗的讲第一级缓存主要负责缓存还在屏幕上的ViewHoler，以LinearLayoutManager为例，ViewHoler被加入第一级缓存的一种流程为
- RecyclerView.onMeasure()
- RecyclerView.dispatchLayoutStep2()
- LinearLayoutManager.onLayoutChildren(...)
- LayoutManager.detachAndScrapAttachedViews(...)

 ```
 public void detachAndScrapAttachedViews(Recycler recycler) {
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);
        scrapOrRecycleView(recycler, i, v);
    }
}
 ```

当我们刷新RecyclerView时，如果使用LinearLayoutManager，我们会发现LinearLayoutManager依次讲屏幕上所有的ViewHolder（准确的说RecyclerView所有子View对应的ViewHolder）都加入了第一级缓存。
