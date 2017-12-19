---
title: LinearLayoutManager StaggeredGridLayoutManager源码阅读
date: 2017-12-19 15:45:56
tags: 
    - RecyclerView LayoutManager
    - Android

---


实现自定义的通用的LayoutManager，但是卡住了，遂看下Android 官方的几种LayoutManager是如何实现的，大致的以及一些细节都看懂了，但是还是没找到什么好办法解决自己的问题，不如趁着热度把自己的分析过程写下来，也给其他需要的Androider.也不细分章节了，就按照滚动的流程来写。至于为什么从滚动开始分析，是因为看源码还是讲究切入点，从RecyclerView的滑动开始是最佳切入点，很直观。

由于自定的LayoutManager如果要(肯定要，不然还定义啥)支持滚动都必须至少重写以下两个方法中的一个，并且返回true，分别表示支持垂直滚动和水平滚动
```java
 public boolean canScrollHorizontally() {
            return true;
        }
 public boolean canScrollVertically() {
            return true;
        }
```
在发生滚动的时候，会在以下两个方法回调滚动的距离dy/dx
```java
    public int scrollVerticallyBy(int dy, Recycler recycler, State state) {
            return 0;
        }

    public int scrollHorizontallyBy(int dx, Recycler recycler, State state) {
            return 0;
        }
```
这里我们从LinearLayoutManager的垂直滚动分析起，进入scrollBy方法,
```java
    int scrollBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (getChildCount() == 0 || dy == 0) {
            return 0;
        }
        //滚动发生时，是需要回收View的
        mLayoutState.mRecycle = true;
        ensureLayoutState();
        //手指向上滑动时dy>0
        final int layoutDirection = dy > 0 ? LayoutState.LAYOUT_END : LayoutState.LAYOUT_START;
        final int absDy = Math.abs(dy);
        //更新LayoutState
        updateLayoutState(layoutDirection, absDy, true, state);
        final int consumed = mLayoutState.mScrollingOffset
                + fill(recycler, mLayoutState, state, false);
        if (consumed < 0) {
            if (DEBUG) {
                Log.d(TAG, "Don't have any more elements to scroll");
            }
            return 0;
        }
        final int scrolled = absDy > consumed ? layoutDirection * consumed : dy;
        //layout view结束，所有View整体平移；这里需要注意的是LLM并非是从头到尾一个个layout view，而是先根据偏移把需要回收的view回收掉，会显示的view显示出来，最后进行整体的平移。想一想这样效率确实要高
        mOrientationHelper.offsetChildren(-scrolled);
        if (DEBUG) {
            Log.d(TAG, "scroll req: " + dy + " scrolled: " + scrolled);
        }
        mLayoutState.mLastScrollDelta = scrolled;
        return scrolled;
    }
```
写了部分注释，具体分析一下updateLayoutState方法(不喜欢贴太长串的代码。。。)
```java
     private void updateLayoutState(int layoutDirection, int requiredSpace,
            boolean canUseExistingSpace, RecyclerView.State state) {
        // If parent provides a hint, don't measure unlimited.
        mLayoutState.mInfinite = resolveIsInfinite();
        mLayoutState.mExtra = getExtraLayoutSpace(state);
        mLayoutState.mLayoutDirection = layoutDirection;
        int scrollingOffset;
        if (layoutDirection == LayoutState.LAYOUT_END) {
            mLayoutState.mExtra += mOrientationHelper.getEndPadding();
            // get the first child in the direction we are going
            final View child = getChildClosestToEnd();
            // the direction in which we are traversing children
            mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                    : LayoutState.ITEM_DIRECTION_TAIL;
            mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
            mLayoutState.mOffset = mOrientationHelper.getDecoratedEnd(child);
            // calculate how much we can scroll without adding new children (independent of layout)
            scrollingOffset = mOrientationHelper.getDecoratedEnd(child)
                    - mOrientationHelper.getEndAfterPadding();

        } else {
            final View child = getChildClosestToStart();
            mLayoutState.mExtra += mOrientationHelper.getStartAfterPadding();
            mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                    : LayoutState.ITEM_DIRECTION_HEAD;
            mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
            mLayoutState.mOffset = mOrientationHelper.getDecoratedStart(child);
            scrollingOffset = -mOrientationHelper.getDecoratedStart(child)
                    + mOrientationHelper.getStartAfterPadding();
        }
        mLayoutState.mAvailable = requiredSpace;
        if (canUseExistingSpace) {
            mLayoutState.mAvailable -= scrollingOffset;
        }
        mLayoutState.mScrollingOffset = scrollingOffset;
    }
```
分析这个方法前，先看下LayoutState这个类，了解一下几个重要参数的含义

mRecycle 表示是否需要回收View
mOffset   layout View时候的起始坐标(垂直方向的LinearLayoutManager 表示y值)，e.g.比如发生滑动后，下一个item需要显示出来，那么mOffset的值就等于最后一个可见item的bottom值(不考虑margin，向上滑动)
mAvailable 表示可用距离，在layout View的时候用到

mCurrentPosition  表示获取View的起始索引，在layout View的时候循环取View的时候用到

mItemDirection  获取item 数据的方向，是从前到后（值为1），还是从后往前（值为-1），本篇分析的是正序情况

mExtra 有些情况下用到，表示距离信息，用于某些情况下的滚动正常手指滚动值是0

再来看updateLayoutState方法