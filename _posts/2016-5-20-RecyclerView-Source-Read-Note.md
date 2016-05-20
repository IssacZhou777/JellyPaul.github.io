---
layout: post
title:  "RecyclerView 源码阅读"
date:   2016-05-20 15:54:10 +0800
categories: android
---


## ViewHolder
源码注释：

>
    /**
     * A ViewHolder describes an item view and metadata about its place within the RecyclerView.
     *
     * <p>{@link Adapter} implementations should subclass ViewHolder and add fields for caching
     * potentially expensive {@link View#findViewById(int)} results.</p>
     *
     * <p>While {@link LayoutParams} belong to the {@link LayoutManager},
     * {@link ViewHolder ViewHolders} belong to the adapter. Adapters should feel free to use
     * their own custom ViewHolder implementations to store data that makes binding view contents
     * easier. Implementations should assume that individual item views will hold strong references
     * to <code>ViewHolder</code> objects and that <code>RecyclerView</code> instances may hold
     * strong references to extra off-screen item views for caching purposes</p>
     */
     
对于传统的AdapterView，需要在实现的Adapter类中手动加ViewHolder，RecyclerView直接将ViewHolder内置，并在原来基础上功能上更强大。ViewHolder描述RecylerView中某个位置的itemView和元数据信息，属于Adapter的一部分。其实现类通常用于保存findViewById的结果。 主要元素组成有：

```java 
	public final View itemView;
    int mPosition = NO_POSITION;
    int mOldPosition = NO_POSITION;
    long mItemId = NO_ID;
    int mItemViewType = INVALID_TYPE;
    int mPreLayoutPosition = NO_POSITION;
    // The item that this holder is shadowing during an item change event/	animation
    ViewHolder mShadowedHolder = null;
    // The item that is shadowing this holder during an item change event/	animation
    ViewHolder mShadowingHolder = null; 
```


***ViewHolder的几个状态标记(mFlag)：***
>static final int FLAG_BOUND = 1 << 0;

表示ViewHolder已绑定某个位置的数据。mPosition, mItemId and mItemViewType都可用。

> static final int FLAG_UPDATE = 1 << 1;

表示该ViewHolder的映射关系不是最新的，需要通过adapter重新绑定。其mPosition和mItemId是一致的。

> static final int FLAG_INVALID = 1 << 2;

表示该ViewHolder不可用，必须要重新绑定新的数据。

> static final int FLAG_REMOVED = 1 << 3;

表示该ViewHolder指向的数据已经从数据源中移除，但是它的View仍旧被某些操作使用，比如移除动画等。

> static final int FLAG_NOT_RECYCLABLE = 1 << 4;

表示该ViewHolder不能被重用， 目的是为了在动画中保持view， 该状态通过setIsRecyclable()设置

> static final int FLAG_RETURNED_FROM_SCRAP = 1 << 5;

这个状态的ViewHolder会加到scrap list被复用。

> static final int FLAG_CHANGED = 1 << 6;

表示内容有变化，通常用于表明有ItemAnimator动画。

> static final int FLAG_IGNORE = 1 << 7;

ViewHolder完全由LayoutManager管理，不能回收， 重用， 移除

> static final int FLAG_TMP_DETACHED = 1 << 8;

ViewHolder从父RecyclerView临时分离的标志，便于后续移除或添加回来  

> static final int FLAG_ADAPTER_POSITION_UNKNOWN = 1 << 9;

ViewHolder不知道对应的Adapter的位置，直到绑定到一个新位置 

> static final int FLAG_ADAPTER_FULLUPDATE = 1 << 10;

方法addChangePayload(null)调用时设置
## RecycledViewPool
源码注释：

>
    /**
     * RecycledViewPool lets you share Views between multiple RecyclerViews.
     * <p>
     * If you want to recycle views across RecyclerViews, create an instance of RecycledViewPool
     * and use {@link RecyclerView#setRecycledViewPool(RecycledViewPool)}.
     * <p>
     * RecyclerView automatically creates a pool for itself if you don't provide one.
     *
     */

RecycledViewPool可以让你在多个RecyclerView之间共享View， 即ViewHold。
如果你想跨多个RecyclerView重用view， 则需要创建一个RecycledViewPool实例， 并在RecyclerVew中调用setRecycledViewPool(RecycledViewPool)方法。如果你创建RecycledViewPool， RecyclerView会自动创建一个pool。

**成员变量**：

* ***mScrap***: SparseArray< ArrayList< ViewHolder>>:  一个< viewType , List>映射。 存储可复用的View(ViewHolder), 按viewType存储该类型下所有ViewHolder
* ***mMaxScrap***: SparseIntArray :一个<viewType: Max>映射， 反应没个类型(viewType)下缓存的最大数量。
* ***mAttachCount*** 统计依附于RecyclerView的View的数量
* ***DEFAULT_MAX _SCRAP*** = 5; mMaxScrap的默认值

**成员方法**：

* ***clear()***  清空RecycledPool中的View（ViewHolder）
* ***setMaxRecycledViews(int viewType, int max)*** 设置对应viewType中的最大数量限制，  如果当前ViewHolder数量超过max, 则循环移除最后一项ViewHolder， 直到数量等于max。
* ***getRecycledView（int viewType）*** 获取指定ViewType的list中最后一个ViewHolder, 没有则返回null
* ***size()***  返回RecycledPool的大小， 即每一个viewType对应的ViewHolder数量总和
* ***putRecycledView（ViewHolder scrap）*** 添加ViewHolder。 如果数量已经超过最大值，则不添加。
* ***attach（Adapter adapter）*** mAttachCount++
* ***detach()*** mAttachCount--
* ***onAdapterChanged(Adapter oldAdapter, Adapter newAdapter,boolean compatibleWithPrevious)  ***与adapter的关联发生变化时会调用。  如果RecycledPool仅关联一个adapter, 新关联的adapter对ViewHolder的使用不同，即viewType 与ViewHolder不同， 则RecycledPool会清空缓存。 compatibleWithPrevious在新老adapter都是用相同的viewType和ViewHolder时为true.
* ***getScrapHeapForType(int viewType)*** 获取对应viewType的ViewHolder缓存，如果不存在，则初始化一个空的缓存。
     
## ViewCacheExtension

源码注释：

>   /**
     * ViewCacheExtension is a helper class to provide an additional layer of view caching that can
     * ben controlled by the developer.
     * <p>
     * When {@link Recycler#getViewForPosition(int)} is called, Recycler checks attached scrap and
     * first level cache to find a matching View. If it cannot find a suitable View, Recycler will
     * call the {@link #getViewForPositionAndType(Recycler, int, int)} before checking
     * {@link RecycledViewPool}.
     * <p>
     * Note that, Recycler never sends Views to this method to be cached. It is developers
     * responsibility to decide whether they want to keep their Views in this custom cache or let
     * the default recycling policy handle it.
     */
     
ViewCacheExtension是一个由开发者控制的可以作为View缓存的帮助类。调用Recycler.getViewForPosition(int)方法获取View时，Recycler先检查attached scrap和一级缓存，如果没有则检查ViewCacheExtension.getViewForPositionAndType(Recycler, int, int)，如果没有再检查RecyclerViewPool。注意：Recycler不会在这个类中做缓存View的操作，是否缓存View完全由开发者控制。

## Recycler

源码注释：

>   /**
     * A Recycler is responsible for managing scrapped or detached item views for reuse.
     *
     * <p>A "scrapped" view is a view that is still attached to its parent RecyclerView but
     * that has been marked for removal or reuse.</p>
     *
     * <p>Typical use of a Recycler by a {@link LayoutManager} will be to obtain views for
     * an adapter's data set representing the data at a given position or item ID.
     * If the view to be reused is considered "dirty" the adapter will be asked to rebind it.
     * If not, the view can be quickly reused by the LayoutManager with no further work.
     * Clean views that have not {@link android.view.View#isLayoutRequested() requested layout}
     * may be repositioned by a LayoutManager without remeasurement.</p>
     */

Recycler的职责在于管理废弃(scrapped)的或者与RecyclerView分离(detached)的itemView, 以便重用itemView。

scrapped view是指依附在RecyclerView上， 但已标记为移除或重用的View。

Recycler典型的用法：为adapter数据集获取view，展示特定位置(position)或者特定itemID的数据。如果被重用的view是失效的（dirty），则需要adapter重新绑定view.如果是有效的(clean)， 则可以直接复用。 对于有效(clean)的view， 如果不主动调用isLayoutRequested()来请求layout， 则LayoutManager重新定位时就不需要重新测量大小。

**成员变量**

* ***ArrayList< ViewHolder> mAttachedScrap*** ： 依附RecyclerView的ViewHolder列表。
* ***ArrayList< ViewHolder>mChangedScrap***： 与RecyclerView分离的ViewHolder列表。
* ***ArrayList< ViewHolder> mCachedViews***： 缓存ViewHolder。
* ***private final List<ViewHolder> mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap)***： mAttachedScrap不可修改类型的列表
* ***RecycledViewPool mRecyclerPool***:  提供ViewHolder复用缓存池。
* ***ViewCacheExtension mViewCacheExtension***： 由开发者控制的ViewHolder缓存池。
* ***int mViewCacheMax*** ： mCacheView的最大值，  即ViewHolder缓存的最大值， 默认为2.

**成员方法**

* ***setViewCacheSize(int viewCount)*** : 设置缓存列表大小， 如果viewCount小于当前缓存列表的size()， 则将多余的ViewHolder添加到RecycledPool，并从缓存列表中移除。
* ***clear()*** 清空当前依附于RecyclerView的ViewHolder， 将缓存的ViewHolder移动到RecycledPool中。
* ***List<ViewHolder> getScrapList()*** : 返回当前依附RecyclerView的ViewHolder列表（mAttachedScrap）, 需要注意的是这里返回的是不可修改的的列表mUnmodifiableAttachedScrap。
* ***boolean validateViewHolderForOffsetPosition（ViewHolder holder）*** ： 该方法是getViewForPosition方法的辅助方法， 用于检验给出的ViewHolder对于某个位置是否可用。如果匹配则为true, 否则为false。
* ***bindViewToPosition(View view, int position)*** ： 该方法的作用是将一个View（ViewHolder）绑定到给定位置。该View可以是通过getViewForPosition(int)得到的， 也可以是通过 Adapter.onCreateViewHolder(ViewGroup, int)创建的。通常情况下， 一个LayoutManager应该通过getViewForPosition获取View，并交由 RecyclerView处理缓存。但对于要自己处理复用逻辑的LayoutManager,该方法是LayoutManager自己处理复用逻辑的辅助方法。另外需要注意的是： getViewForPosition获取到的View是已经绑定过position的。所以如果你不想要绑定另一个position的话，并不需要调用该方法。
* ***convertPreLayoutPositionToPostLayout(int position)***： RecyclerView在pre-layout状态下会提供人造的位置范围（item count），即当前可以绘制的item总数， 当调用getViewForPosition或者bindViewToPosition时会自动将该范围内的position映射对应adapter的位置position。通常情况下， LayoutManager不需要关心这些。然而，在某些情况下，比如你的LayoutManager调用自定义组件，需要传入一个真实的adapter中的位置position时，就可以调用该方法将pre-layout状态下的位置转化为adapter（post layout）状态下单位置。 **需要注意的是：**如果传入的位置所对应的ViewHolder已被删除，则返回－1。并且传入的position的位置应该在［0，itemCount］之间 。如果在post layout状态调用该方法， 则返回结果 ＝ 传入的position
*  ***View getViewForPosition(int position), getViewForPosition(int position, boolean dryRun)***: 获取某个位置的View。该方法应由LayoutManager调用，实现获取View展示adapter中的数据的目的。如果ViewHolder的viewType正确的话，Recycler会重用共享池中的scrap或者detached view，即ViewHolder。如果adapter没有说明给定位置的数据有变化， Recycler则会尝试返回一个之前初始化的scrap（ViewHolder） ，不需要重新绑定数据。

## RecyclerViewDataObserver

源码： 

``` java

private class RecyclerViewDataObserver extends AdapterDataObserver {
        @Override
        public void onChanged() {
            assertNotInLayoutOrScroll(null);
            if (mAdapter.hasStableIds()) {
                // TODO Determine what actually changed.
                // This is more important to implement now since this callback will disable all
                // animations because we cannot rely on positions.
                mState.mStructureChanged = true;
                setDataSetChangedAfterLayout();
            } else {
                mState.mStructureChanged = true;
                setDataSetChangedAfterLayout();
            }
            if (!mAdapterHelper.hasPendingUpdates()) {
                requestLayout();
            }
        }

        @Override
        public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
            assertNotInLayoutOrScroll(null);
            if (mAdapterHelper.onItemRangeChanged(positionStart, itemCount, payload)) {
                triggerUpdateProcessor();
            }
        }

        @Override
        public void onItemRangeInserted(int positionStart, int itemCount) {
            assertNotInLayoutOrScroll(null);
            if (mAdapterHelper.onItemRangeInserted(positionStart, itemCount)) {
                triggerUpdateProcessor();
            }
        }

        @Override
        public void onItemRangeRemoved(int positionStart, int itemCount) {
            assertNotInLayoutOrScroll(null);
            if (mAdapterHelper.onItemRangeRemoved(positionStart, itemCount)) {
                triggerUpdateProcessor();
            }
        }

        @Override
        public void onItemRangeMoved(int fromPosition, int toPosition, int itemCount) {
            assertNotInLayoutOrScroll(null);
            if (mAdapterHelper.onItemRangeMoved(fromPosition, toPosition, itemCount)) {
                triggerUpdateProcessor();
            }
        }

        void triggerUpdateProcessor() {
            if (mPostUpdatesOnAnimation && mHasFixedSize && mIsAttached) {
                ViewCompat.postOnAnimation(RecyclerView.this, mUpdateChildViewsRunnable);
            } else {
                mAdapterUpdateDuringMeasure = true;
                requestLayout();
            }
        }
    }

```


##SavedState

##AdapterHelper

##ChildHelper

##Adapter

##LayoutManager

##RecyclerListener

##EdgeEffectCompat

##ViewFlinger

##State

##RecyclerViewAccessibilityDelegate

##NestedScrollingChildHelper