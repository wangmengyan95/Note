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

通俗的讲第一级缓存主要负责缓存还在屏幕上的并且对应数据并没有发生变化的ViewHoler，以LinearLayoutManager为例，ViewHoler被加入第一级缓存的一种流程为
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

当我们重新layout RecyclerView时，如果使用LinearLayoutManager，我们会发现LinearLayoutManager依次讲屏幕上所有的ViewHolder（准确的说RecyclerView所有子View对应的ViewHolder）都加入了第一级缓存。

从第一级缓存中取出ViewHolder的逻辑比较简单，其中最核心的一条判定标准就是holder.getLayoutPosition() == position，即如果原来ViewHolder用来显示某个位置的信息，当数据源没发生变化，我们仍然可以继续用这个ViewHolder显示这个位置的信息。

继续上述流程，在detachAndScrapAttachedViews(...)后LinearLayoutManager依次将屏幕上所有的ViewHolder都加入了第一级缓存之后
- LinearLayoutManager.fill(...)
- LinearLayoutManager.layoutChunk(...)
- LayoutState.next(...)
- Recycler.getViewForPosition(...)

在getViewForPosition(..)中，Recycler通过第一级缓存比对position，很快将对应的ViewHolder返回，快速完成了整个measure的流程。

综上所述，第一级缓存逻辑的核心就是当数据没有发生变化时（即我们不需要重新去adapter bindViewHolder），我们可以尽量用原来相同位置的ViewHolder显示一样的数据。
