---
layout: post
title:  "Android RecyclerView源码学习"
date:   2020-6-4 9:53:25 +0800
categories: Android
---


> 文章篇幅较长，文末有总结和流程图。
> 个人主页：https://chengang.plus/
> 文章将会同步到个人微信公众号：Android部落格，文末有二维码

## 1、用法
一个比较简单的用法如下：

```kotlin
class AndroidDeepLearnActivity : Activity() {

    lateinit var dlRecyclerView: RecyclerView
    lateinit var context: Context

    private val images = arrayOf(
        "http://api.learn2crack.com/android/images/donut.png",
        "http://api.learn2crack.com/android/images/eclair.png",
        "http://api.learn2crack.com/android/images/froyo.png",
        "http://api.learn2crack.com/android/images/ginger.png",
        "http://api.learn2crack.com/android/images/honey.png",
        "http://api.learn2crack.com/android/images/icecream.png",
        "http://api.learn2crack.com/android/images/jellybean.png",
        "http://api.learn2crack.com/android/images/kitkat.png",
        "http://api.learn2crack.com/android/images/lollipop.png",
        "http://api.learn2crack.com/android/images/marshmallow.png"
    )

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_android_deep_learn)
        context = this
        dlRecyclerView = findViewById(R.id.dp_recycler_view)

        val itemDecorator = DividerItemDecoration(context, DividerItemDecoration.VERTICAL)
        itemDecorator.setDrawable(context.getDrawable(R.drawable.divider))
        dlRecyclerView.addItemDecoration(itemDecorator)

        val layoutManager = LinearLayoutManager(this, LinearLayoutManager.VERTICAL, false)
        dlRecyclerView.layoutManager = layoutManager
        dlRecyclerView.adapter = DLRecyclerViewAdapter(context, images)
    }

    class DLRecyclerViewAdapter(context: Context, images: Array<String>) :
        RecyclerView.Adapter<DLViewHolder>() {
        private val innerImages = images
        private val innerCount = images?.size
        private val mContext = context

        private var layoutInflater: LayoutInflater

        init {
            layoutInflater = LayoutInflater.from(mContext)
        }

        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): DLViewHolder {
            val view = layoutInflater.inflate(R.layout.dl_recyclerview_item_view, parent, false)
            return DLViewHolder(view)
        }

        override fun getItemCount(): Int {
            return innerCount
        }

        override fun onBindViewHolder(holder: DLViewHolder, position: Int) {
            holder.itemImage = holder.itemView.findViewById(R.id.dl_image_item)
            holder.itemImage.setImageURI(innerImages[position])
        }
    }

    class DLViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        lateinit var itemImage: SimpleDraweeView
    }
}
```

## 2、第一次构建
### 2.1 RecyclerView构造函数
RecyclerView在xml文件中定义，在View初始化的时候，会调用到他的构造函数：

```java
public RecyclerView(@NonNull Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    initAdapterManager();
    initChildrenHelper();
    createLayoutManager(context, layoutManagerName, attrs, defStyleAttr, 0);
    setNestedScrollingEnabled(nestedScrollingEnabled);
}
```

#### 2.1.1 AdapterHelper
initAdapterManager方法初始化了一个AdapterHelper对象，如下：

```java
mAdapterHelper = new AdapterHelper(new AdapterHelper.Callback() {
@Override
    public ViewHolder findViewHolder(int position) {}
    
    @Override
    public void offsetPositionsForRemovingInvisible(int start, int count) {}
    
    @Override
    public void offsetPositionsForRemovingLaidOutOrNewView(int positionStart, int itemCount) {}
    
    @Override
    public void markViewHoldersUpdated(int positionStart, int itemCount, Object payload) {}
    
    @Override
    public void onDispatchFirstPass(AdapterHelper.UpdateOp op) {
        dispatchUpdate(op);
    }
    
    void dispatchUpdate(AdapterHelper.UpdateOp op) {}
    
    @Override
    public void onDispatchSecondPass(AdapterHelper.UpdateOp op) {
        dispatchUpdate(op);
    }
    
    @Override
    public void offsetPositionsForAdd(int positionStart, int itemCount) {}
    
    @Override
    public void offsetPositionsForMove(int from, int to) {}
}
```

#### 2.1.2 ChildHelper
initChildrenHelper方法初始化了一个ChildHelper对象，如下：

```java
mChildHelper = new ChildHelper(new ChildHelper.Callback() {
    @Override
    public int getChildCount() {
        return RecyclerView.this.getChildCount();
    }
    
    @Override
    public void addView(View child, int index) {
        RecyclerView.this.addView(child, index);
    }
    
    @Override
    public int indexOfChild(View view) {
        return RecyclerView.this.indexOfChild(view);
    }
    
    @Override
    public void removeViewAt(int index) {
        final View child = RecyclerView.this.getChildAt(index);
        RecyclerView.this.removeViewAt(index);
    }
    
    @Override
    public View getChildAt(int offset) {
        return RecyclerView.this.getChildAt(offset);
    }
    
    @Override
    public void removeAllViews() {}
    
    @Override
    public ViewHolder getChildViewHolder(View view) {
        return getChildViewHolderInt(view);
    }
    
    @Override
    public void attachViewToParent(View child, int index, ViewGroup.LayoutParams layoutParams) {}
            
    @Override
    public void detachViewFromParent(int offset) {}
    
    @Override
    public void onEnteredHiddenState(View child) {}
    
    @Override
    public void onLeftHiddenState(View child) {}
}
```

#### 2.1.3 xml中定义layoutManager

如果在xml中定义了layoutManager类型，此处会调用createLayoutManager方法创建LayoutManager。

#### 2.1.4 setNestedScrollingEnabled
对应nestedScrollingEnabled属性，既可以在xml中定义RecyclerView控件时定义，也可以在代码中设置。对应的是由父控件分发滚动消息还是自身分发，默认是自身分发。

该属性在Android SDK_INT 大于等于21的时候才有效。否则一律默认为true。


### 2.2 LayoutManager
以LinearLayoutManager为例，他继承自RecyclerView.LayoutManager。

#### 2.2.1 构造函数
> LinearLayoutManager

```java
public LinearLayoutManager(Context context, @RecyclerView.Orientation int orientation, boolean reverseLayout) {
    setOrientation(orientation);
    setReverseLayout(reverseLayout);
}
```
这里有三个参数，第一个是上下文；第二个是滚动方向，分为竖直和水平方向；第三个参数是视图排布方向，true的话，视图从尾到头排布，否则从头到尾排布。

#### 2.2.2 设置方向
通过setOrientation方法设置滚动，如下：
> LinearLayoutManager

```java
public void setOrientation(@RecyclerView.Orientation int orientation) {
    if (orientation != mOrientation || mOrientationHelper == null) {
        mOrientationHelper = OrientationHelper.createOrientationHelper(this, orientation);
        mAnchorInfo.mOrientationHelper = mOrientationHelper;
        mOrientation = orientation;
        requestLayout();
    }
}
```
在这里可以看到如果不设置方向的话，默认是竖直方向。第一次进来的时候，mOrientationHelper没有初始化，为null，这里先执行createOrientationHelper方法：
> OrientationHelper

```java
public static OrientationHelper createOrientationHelper(
        RecyclerView.LayoutManager layoutManager, @RecyclerView.Orientation int orientation) {
    switch (orientation) {
        case HORIZONTAL:
            return createHorizontalHelper(layoutManager);
        case VERTICAL:
            return createVerticalHelper(layoutManager);
    }
    throw new IllegalArgumentException("invalid orientation");
}
```
根据不同的方向分别创建不同的Helper，以VERTICAL为例，看看createVerticalHelper方法：
> OrientationHelper

```java
public static OrientationHelper createVerticalHelper(RecyclerView.LayoutManager layoutManager) {
    return new OrientationHelper(layoutManager) {};
}
```
这里新建一个持有RecyclerView.LayoutManager对象的OrientationHelper。

- AnchorInfo

到这里会把创建的OrientationHelper对象给到一个AnchorInfo；把用户设置的orientation给到全局变量mOrientation。

#### 2.2.3 设置反转视图
如果初始化LinearLayoutManager的时候，将第三个参数设置为true，那么就会反转视图。将LinearLayoutManager中的全局变量mReverseLayout设置为用户传递的参数，同时执行requestLayout方法。

### 2.3 设置LayoutManager
贴一下setLayoutManager方法看看：
> RecyclerView

```java
public void setLayoutManager(@Nullable LayoutManager layout) {
    if (layout == mLayout) {
        return;
    }
    stopScroll();
    // TODO We should do this switch a dispatchLayout pass and animate children. There is a good
    // chance that LayoutManagers will re-use views.
    if (mLayout != null) {
        // end all running animations
        if (mItemAnimator != null) {
            mItemAnimator.endAnimations();
        }
        mLayout.removeAndRecycleAllViews(mRecycler);
        mLayout.removeAndRecycleScrapInt(mRecycler);
        mRecycler.clear();

        if (mIsAttached) {
            mLayout.dispatchDetachedFromWindow(this, mRecycler);
        }
        mLayout.setRecyclerView(null);
        mLayout = null;
    } else {
        mRecycler.clear();
    }
    // this is just a defensive measure for faulty item animators.
    mChildHelper.removeAllViewsUnfiltered();
    mLayout = layout;
    if (layout != null) {
        if (layout.mRecyclerView != null) {
            throw new IllegalArgumentException("LayoutManager " + layout
                    + " is already attached to a RecyclerView:"
                    + layout.mRecyclerView.exceptionLabel());
        }
        mLayout.setRecyclerView(this);
        if (mIsAttached) {
            mLayout.dispatchAttachedToWindow(this);
        }
    }
    mRecycler.updateViewCacheSize();
    requestLayout();
}
```
- mLayout是LayoutManager类型，如果当前的形参layout与mLayout是同一个对象，就直接返回。
- 停止滚动。
- mLayout不为空，意味着之前已经设置过了，此时就要停止动画，将mLayout里面所有的视图全部清掉，并置RecyclerView和mLayout为空，等于是重新开始了。
- Recycler，回收者，是一个全局new出来的对象，在RecyclerView中。看注释他的作用是管理着那些报废或脱离父视图的组件重用。报废的意思是视图仍然依附着父视图，只是被标记为被移除或重用。Recycler持有几个ArrayList变量，都是与item缓存回收相关。这里执行clear方法，意味着将缓存的item全部清除。
- 初始化mChildHelper，同时将mLayout重新复制，并同时设置LayoutManager中的RecyclerView变量。
- 更新缓存大小，不设置的话，默认两个。同时请求更新视图。

#### 2.3.1 更新视图
其实在new LinearLayoutManager的时候，会调用`setOrientation`和`setReverseLayout`方法，这两个方法里面都用调用requestLayout，在LayoutManager中定义：

```java
public void requestLayout() {
    if (mRecyclerView != null) {
        mRecyclerView.requestLayout();
    }
}
```
但是在调用这两个方法的时候mRecyclerView没有被初始化，一直到setLayoutManager方法的时候才被初始化。

在setLayoutManager执行的最后，调用requestLayout方法，在RecyclerView中定义：

```java
@Override
public void requestLayout() {
    if (mInterceptRequestLayoutDepth == 0 && !mLayoutSuppressed) {
        super.requestLayout();
    } else {
        mLayoutWasDefered = true;
    }
}
```
这里会执行到View的requestLayout方法，触发视图更新。

我们知道View更新的流程是onMeasure，onLayout，onDraw.

通过查看这三个方法，我们发现不设置adapter，根本不会触发视图的更新操作。

### 2.4 设置adapter
最开始的demo中的DLRecyclerViewAdapter类继承自RecyclerView.Adapter，可以限定一下Adapter的ViewHolder类型，但是对于在RecyclerView中加载复杂类型的item来说，需要考虑到ViewHolder的通用性。

> RecyclerView

```java
public void setAdapter(@Nullable Adapter adapter) {
    // bail out if layout is frozen
    setLayoutFrozen(false);
    setAdapterInternal(adapter, false, true);
    processDataSetCompletelyChanged(false);
    requestLayout();
}
```
#### 2.4.1 setLayoutFrozen
`setLayoutFrozen(false)`意味着items被enable和滚动
#### 2.4.2 setAdapterInternal
`setAdapterInternal`看看这个方法：

```java
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious,
            boolean removeAndRecycleViews) {
    if (mAdapter != null) {
        mAdapter.unregisterAdapterDataObserver(mObserver);
        mAdapter.onDetachedFromRecyclerView(this);
    }
    if (!compatibleWithPrevious || removeAndRecycleViews) {
        removeAndRecycleViews();
    }
    mAdapterHelper.reset();
    final Adapter oldAdapter = mAdapter;
    mAdapter = adapter;
    if (adapter != null) {
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    if (mLayout != null) {
        mLayout.onAdapterChanged(oldAdapter, mAdapter);
    }
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
    mState.mStructureChanged = true;
}
```
- 先判断mAdapter是否为空，不为空的话，解注册mObserver，是一个全局新建的对象，RecyclerViewDataObserver类型，这个类继承自AdapterDataObserver，实现了dataset changed相关的函数，比如`onItemRangeChanged`，`onItemRangeInserted`等。

解注册最终实现的地方在Observable类中：
> Observable

```java
public void unregisterObserver(T observer) {
    if (observer == null) {
        throw new IllegalArgumentException("The observer is null.");
    }
    synchronized(mObservers) {
        int index = mObservers.indexOf(observer);
        if (index == -1) {
            throw new IllegalStateException("Observer " + observer + " was not registered.");
        }
        mObservers.remove(index);
    }
}
```
mObservers是一个AdapterDataObserver类型的ArrayList。
- removeAndRecycleViews做一些清理相关的工作以及终止item动画等
- AdapterHelper是在RecyclerView的构造函数中创建的，在这里复位
- 将mAdapter赋值为新的adapter
- 重新注册mObserver
- 调用onAttachedToRecyclerView，在Adapter中可以复写这个方法
- 执行LayoutManager类的onAdapterChanged方法
- 调用Recycler的onAdapterChanged方法：

> Recycler

```java
void onAdapterChanged(Adapter oldAdapter, Adapter newAdapter, boolean compatibleWithPrevious) {
    clear();
    getRecycledViewPool().onAdapterChanged(oldAdapter, newAdapter, compatibleWithPrevious);
}
```
清理废弃和缓存的item。

#### 2.4.3 RecycledViewPool
获取RecycledViewPool，看看这个方法：
```java
RecycledViewPool getRecycledViewPool() {
    if (mRecyclerPool == null) {
        mRecyclerPool = new RecycledViewPool();
    }
    return mRecyclerPool;
}
```
首次设置Adapter，mRecyclerPool参数为空，需要new一个。这个类的作用是在多个RecycledView之间共享item，可以通过setRecycledViewPool方法设置一个RecycledViewPool，如果不设置的话，会默认创建一个。

接下来执行RecycledViewPool的onAdapterChanged方法，主要逻辑是将旧的Adapter detach，将新的Adapter attach，对应的函数操作就是将变量mAttachCount自减或自增。

回到setAdapter方法，接下来执行processDataSetCompletelyChanged方法，主要目的是将之前Holder持有的view设置为valid。

下一步执行requestLayout，更新视图。

### 2.5 更新视图
#### 2.5.1 onMeasure
因为有各种变量判断，这里简单抽取一下，就只剩下几行代码：

```java
@Override
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mAdapter != null) {
        mState.mItemCount = mAdapter.getItemCount();
    } else {
        mState.mItemCount = 0;
    }
    startInterceptRequestLayout();
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    stopInterceptRequestLayout(false);
    mState.mInPreLayout = false; // clear
}
```

- 触发获取item数量
- 触发RecyclerView的onMeasure，设置滚动视图的尺寸
- startInterceptRequestLayout和stopInterceptRequestLayout成对出现，目的是为了避免多个requestLayout方法同时调用，待当次执行完毕后，再执行

#### 2.5.2 onLayout
看看这个定义在RecyclerView中的方法：

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```
调用dispatchLayout：

```java
void dispatchLayout() {
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        // leave the state in START
        return;
    }
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        // leave the state in START
        return;
    }
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
            || mLayout.getHeight() != getHeight()) {
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```
mState是一个全局的State对象，持有RecyclerView的滚动位置和焦点等状态信息。

#### 2.5.3 dispatchLayoutStep1
第一次到这里只会执行简单的逻辑：
> RecyclerView

```java
private void dispatchLayoutStep1() {
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

#### 2.5.4 dispatchLayoutStep2
接下来执行dispatchLayoutStep2函数：
> RecyclerView

```java
private void dispatchLayoutStep2() {
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);
    mState.mStructureChanged = false;
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
}
```
首先获取RecyclerView里面的item数量，这里会直接调用到我们自己继承Adapter的类中，覆写的getItemCount函数。

给State类的一些变量赋值。

调用LinearLayoutManager的onLayoutChildren方法，携带Recycler和State参数。

#### 2.5.5 onLayoutChildren
为子item布局，看看这个方法：
> LinearLayoutManager

```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    final int firstLayoutDirection;
    if (mAnchorInfo.mLayoutFromEnd) {
        firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL
                : LayoutState.ITEM_DIRECTION_HEAD;
    } else {
        firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                : LayoutState.ITEM_DIRECTION_TAIL;
    }
    if (mAnchorInfo.mLayoutFromEnd) {
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForStart;
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        final int firstElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForEnd += mLayoutState.mAvailable;
        }
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
    
        if (mLayoutState.mAvailable > 0) {
            // end could not consume all. add more items towards start
            extraForStart = mLayoutState.mAvailable;
            updateLayoutStateToFillStart(firstElement, startOffset);
            mLayoutState.mExtraFillSpace = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
        }
    } else {
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForEnd;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;
        final int lastElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForStart += mLayoutState.mAvailable;
        }
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtraFillSpace = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
    
        if (mLayoutState.mAvailable > 0) {
            extraForEnd = mLayoutState.mAvailable;
            // start could not consume all it should. add more items towards end
            updateLayoutStateToFillEnd(lastElement, endOffset);
            mLayoutState.mExtraFillSpace = extraForEnd;
            fill(recycler, mLayoutState, state, false);
            endOffset = mLayoutState.mOffset;
        }
    }
}
```

主要填充方式分两种，锚点在起始位置或底部。如果把设备当做一个Stack，那么屏幕顶部为start，底部为end。默认mLayoutFromEnd变量为false。

这里会调用两次fill方法，一个填充到底部，一个填充到顶部。

#### 2.5.6 fill填充
> LinearLayoutManager

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
    final int start = layoutState.mAvailable;
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        // TODO ugly bug fix. should not happen
        if (layoutState.mAvailable < 0) {
            layoutState.mScrollingOffset += layoutState.mAvailable;
        }
        recycleByLayoutState(recycler, layoutState);
    }
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
    }
    return start - layoutState.mAvailable;
}
```
这个方法主要看layoutChunk方法，绘制每个view块。

#### 2.5.7 layoutChunk 绘制view块
> LinearLayoutManager

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state, LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection == LayoutState.LAYOUT_START)) {
            addDisappearingView(view);
        } else {
            addDisappearingView(view, 0);
        }
    }
    measureChildWithMargins(view, 0, 0);
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    int left, top, right, bottom;
    if (mOrientation == VERTICAL) {
        if (isLayoutRTL()) {
            right = getWidth() - getPaddingRight();
            left = right - mOrientationHelper.getDecoratedMeasurementInOther(view);
        } else {
            left = getPaddingLeft();
            right = left + mOrientationHelper.getDecoratedMeasurementInOther(view);
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            bottom = layoutState.mOffset;
            top = layoutState.mOffset - result.mConsumed;
        } else {
            top = layoutState.mOffset;
            bottom = layoutState.mOffset + result.mConsumed;
        }
    } else {
        top = getPaddingTop();
        bottom = top + mOrientationHelper.getDecoratedMeasurementInOther(view);
        
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            right = layoutState.mOffset;
            left = layoutState.mOffset - result.mConsumed;
        } else {
            left = layoutState.mOffset;
            right = layoutState.mOffset + result.mConsumed;
        }
    }
    // We calculate everything with View's bounding box (which includes decor and margins)
    // To calculate correct layout position, we subtract margins.
    layoutDecoratedWithMargins(view, left, top, right, bottom);
}
```
先调用next方，传递Recycler参数，然后addView。

#### 2.5.8 next 遍历获取子View
> LinearLayoutManager

```java
View next(RecyclerView.Recycler recycler) {
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}
```

#### 2.5.9 getViewForPosition
> RecyclerView.Recycler

```java
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
```

#### 2.5.10 tryGetViewHolderForPositionByDeadline
> RecyclerView.Recycler

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    if(holder == null){
        final int type = mAdapter.getItemViewType(offsetPosition);
    }
    if (holder == null) {
        long start = getNanoTime();
        if (deadlineNs != FOREVER_NS
                && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
            // abort - we have a deadline we can't meet
            return null;
        }
        holder = mAdapter.createViewHolder(RecyclerView.this, type);
        if (ALLOW_THREAD_GAP_WORK) {
            // only bother finding nested RV if prefetching
            RecyclerView innerView = findNestedRecyclerView(holder.itemView);
            if (innerView != null) {
                holder.mNestedRecyclerView = new WeakReference<>(innerView);
            }
        }
    
        long end = getNanoTime();
        mRecyclerPool.factorInCreateTime(type, end - start);
        if (DEBUG) {
            Log.d(TAG, "tryGetViewHolderForPositionByDeadline created new ViewHolder");
        }
    }

    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        // do not update unless we absolutely have to.
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        if (DEBUG && holder.isRemoved()) {
            throw new IllegalStateException("Removed holder should be bound and it should"
                    + " come here only in pre-layout. Holder: " + holder
                    + exceptionLabel());
        }
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }
}
```
在tryGetViewHolderForPositionByDeadline方法中，会有各种boolean参数判断，这里主要看holder，第一次创建的时候肯定是null，会直接调用createViewHolder创建Holder。

#### 2.5.11 getItemViewType
获取view的类型，一般的，如果RecyclerView里面如果item的类型比较复杂，类似电商App首页这种，就要为banner，限时热卖，商品列表等设置不同的视图类型。

如果在定义的Adapter中不覆写这个方法，就默认返回0。

#### 2.5.11 创建Holder
`createViewHolder`方法用来创建ViewHolder。

> RecyclerView.Adapter

```java
public final VH createViewHolder(@NonNull ViewGroup parent, int viewType) {
    final VH holder = onCreateViewHolder(parent, viewType);
    holder.mItemViewType = viewType;
    return holder;
}
```
这里`onCreateViewHolder`就直接回调到我们代码里面覆写RecyclerView.Adapter的onCreateViewHolder方法中，在这个方法中，创建自定义的ViewHolder（继承自RecyclerView.ViewHolder），然后返回这个ViewHolder的实例。

记下来继续tryGetViewHolderForPositionByDeadline方法中的流程。

#### 2.5.12 tryBindViewHolderByDeadline
> RecyclerView.Recycler

```java
private boolean tryBindViewHolderByDeadline(@NonNull ViewHolder holder, int offsetPosition, int position, long deadlineNs) {
    holder.mOwnerRecyclerView = RecyclerView.this;
    final int viewType = holder.getItemViewType();
    long startBindNs = getNanoTime();
    if (deadlineNs != FOREVER_NS
            && !mRecyclerPool.willBindInTime(viewType, startBindNs, deadlineNs)) {
        // abort - we have a deadline we can't meet
        return false;
    }
    mAdapter.bindViewHolder(holder, offsetPosition);
    long endBindNs = getNanoTime();
    mRecyclerPool.factorInBindTime(holder.getItemViewType(), endBindNs - startBindNs);
    attachAccessibilityDelegateOnBind(holder);
    if (mState.isPreLayout()) {
        holder.mPreLayoutPosition = position;
    }
    return true;
}
```
调用tryBindViewHolderByDeadline方法，核心是bindViewHolder。

#### 2.5.13 bindViewHolder
> RecyclerView.Recycler

```java
public final void bindViewHolder(@NonNull VH holder, int position) {
    holder.mPosition = position;
    holder.clearPayload();
    final ViewGroup.LayoutParams layoutParams = holder.itemView.getLayoutParams();
    if (layoutParams instanceof RecyclerView.LayoutParams) {
        ((LayoutParams) layoutParams).mInsetsDirty = true;
    }
}

public void onBindViewHolder(@NonNull VH holder, int position, @NonNull List<Object> payloads) {
    onBindViewHolder(holder, position);
}
```
跟`onCreateViewHolder`方法类似，到这里，就直接回调到我们代码里面覆写RecyclerView.Adapter的onBindViewHolder方法中，在这个方法中，可以为itemView中的child View设置数据。

通过以上步骤之后，会调用addView将每一个获取到的child view添加到RecyclerView中。

#### 2.5.14 onDraw 绘制
绘制代码如下：
> RecyclerView

```java
@Override
public void onDraw(Canvas c) {
    super.onDraw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);
    }
}
```
这里的mItemDecorations是ArrayList<ItemDecoration>类型，当在我们自己的代码里面初始化RecyclerView，设置item之间的间隔时，就会将DividerItemDecoration对象添加到这个List中。

以DividerItemDecoration为例，这里会调用它的onDraw方法。

#### 2.5.15 绘制item间隔
`DividerItemDecoration`继承自`RecyclerView.ItemDecoration`，在上一步中会直接调用到DividerItemDecoration的onDraw方法，并根据不同的方向绘制间隔，前提是DividerItemDecoration要调用setDrawable设置绘制资源。

```java
@Override
public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
    if (parent.getLayoutManager() == null || mDivider == null) {
        return;
    }
    if (mOrientation == VERTICAL) {
        drawVertical(c, parent);
    } else {
        drawHorizontal(c, parent);
    }
}
```

### 2.6 总结
到这里，第一次实例化RecyclerView，并创建各种对象，回调Adapter的各个函数，并填充绘制item view，最后绘制item之间的间隔。
#### 2.6.1 RecyclerView持有的对象
- RecyclerViewDataObserver mObserver，处理数据变动，增加或删除数据。
- Recycler mRecycler，负责item view回收复用。
- AdapterHelper mAdapterHelper，处理Adapter更新。
- ChildHelper mChildHelper，初始化时，会有一个CallBack接口，LayoutManager通过它的对象调用RecyclerView中对应的方法。
- Adapter mAdapter，完成data和view的绑定，并提供创建ViewHolder的回调方法。
- LayoutManager mLayout，管理item view的布局，回收，重用等。

## 3、缓存与回收
在滑动列表的过程中，会存在缓存，回收复用item view的过程。

### 3.1 滑动事件处理
滑动的时候，在RecyclerView里面会通过onTouchEvent方法获取滑动事件：
> RecyclerView

```java
@Override
public boolean onTouchEvent(MotionEvent e) {
    switch (action) {
        case MotionEvent.ACTION_DOWN: {
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            if (mScrollState != SCROLL_STATE_DRAGGING) {
                if (startScroll) {
                    setScrollState(SCROLL_STATE_DRAGGING);
                }
            }
            if (mScrollState == SCROLL_STATE_DRAGGING) {
                if (scrollByInternal(canScrollHorizontally ? dx : 0, canScrollVertically ? dy : 0, e, TYPE_TOUCH)) {
                    getParent().requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }
    }
    return true;
}
```
放拖拽滑动的时候，会走到ACTION_MOVE里面，并调用scrollByInternal方法，这里的x是水平方向滑动的距离，y是竖直方向滑动的距离：
> RecyclerView

```java
boolean scrollByInternal(int x, int y, MotionEvent ev, int type) {
    if (mAdapter != null) {
        mReusableIntPair[0] = 0;
        mReusableIntPair[1] = 0;
        scrollStep(x, y, mReusableIntPair);
        consumedX = mReusableIntPair[0];
        consumedY = mReusableIntPair[1];
        unconsumedX = x - consumedX;
        unconsumedY = y - consumedY;
    }
}
```
接着执行scrollStep方法，传递x，y参数，以及一个int数组。
> RecyclerView

```java
void scrollStep(int dx, int dy, @Nullable int[] consumed) {
    int consumedX = 0;
    int consumedY = 0;
    if (dx != 0) {
        consumedX = mLayout.scrollHorizontallyBy(dx, mRecycler, mState);
    }
    if (dy != 0) {
        consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
    }
    if (consumed != null) {
        consumed[0] = consumedX;
        consumed[1] = consumedY;
    }
}
```
如果设置RecyclerView为竖直方向滚动，那么dy不等于0，调用scrollVerticallyBy方法，在LinearLayoutManager类中：
> LinearLayoutManager

```java
@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    if (mOrientation == HORIZONTAL) {
        return 0;
    }
    return scrollBy(dy, recycler, state);
}

int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
    mLayoutState.mRecycle = true;
    updateLayoutState(layoutDirection, absDelta, true, state);
    final int consumed = mLayoutState.mScrollingOffset
            + fill(recycler, mLayoutState, state, false);
    final int scrolled = absDelta > consumed ? layoutDirection * consumed : delta;
    mLayoutState.mLastScrollDelta = scrolled;
    return scrolled;
}
```
可以看到上边两个方法的核心目的是为了计算dy值，也就是在竖直方向上滑动了多少距离。

#### 3.1.1 更新视图状态
updateLayoutState方法里面计算了偏移的delta值：
> LinearLayoutManager

```java
private void updateLayoutState(int layoutDirection, int requiredSpace, boolean canUseExistingSpace, RecyclerView.State state) {
    mReusableIntPair[0] = 0;
    mReusableIntPair[1] = 0;
    calculateExtraLayoutSpace(state, mReusableIntPair);
    mLayoutState.mExtraFillSpace = layoutToEnd ? extraForEnd : extraForStart;
    mLayoutState.mNoRecycleSpace = layoutToEnd ? extraForStart : extraForEnd;
    int scrollingOffset;
    if (layoutToEnd) {
        mLayoutState.mExtraFillSpace += mOrientationHelper.getEndPadding();
        // get the first child in the direction we are going
        final View child = getChildClosestToEnd();
        // the direction in which we are traversing children
        mLayoutState.mItemDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD
                : LayoutState.ITEM_DIRECTION_TAIL;
        mLayoutState.mCurrentPosition = getPosition(child) + mLayoutState.mItemDirection;
        mLayoutState.mOffset = mOrientationHelper.getDecoratedEnd(child);
        // calculate how much we can scroll without adding new children (independent of layout)
        scrollingOffset = mOrientationHelper.getDecoratedEnd(child) - mOrientationHelper.getEndAfterPadding();
    } else {
        ...
    }
    mLayoutState.mAvailable = requiredSpace;
    if (canUseExistingSpace) {
        mLayoutState.mAvailable -= scrollingOffset;
    }
    mLayoutState.mScrollingOffset = scrollingOffset;
}
```
- 这里可以看到有一个calculateExtraLayoutSpace方法，用来计算额外布局的空间，LinearLayoutManager为了流畅滑动考虑，提前布局额外一页的item。在这个方法中，如果是竖直方向滑动，mReusableIntPair第一个值为0，第二个值为RecyclerView高度减去top和bottom的padding。

- layoutToEnd，如果向下滑动，那么这个boolean值为true。
- getChildClosestToEnd，用于获取当前页最后一个child item view，具体是通过调用ChildHelper.getChildAt(index)方法，这里的index等于总的item个数减去隐藏（hidden）的个数。
- getDecoratedEnd，获取的是这个view bottom值加上他所属间隔view的bottom，再加上他的margin_bottom值，至此可以得到最后一个item view的y值。
- scrollingOffset最终就等于getDecoratedEnd获取的值减去RecyclerView的padding_top值。
- LayoutState mLayoutState里面记录了child view的位置坐标相关信息。

### 3.2 fill 填充视图
填充之前会做回收工作：
> LinearLayoutManager

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
    recycleByLayoutState(recycler, layoutState);
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
    }
}
```

#### 3.2.1 开始回收
recycleByLayoutState方法里面会做回收逻辑：
> LinearLayoutManager

```java
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
    if (!layoutState.mRecycle || layoutState.mInfinite) {
        return;
    }
    int scrollingOffset = layoutState.mScrollingOffset;
    int noRecycleSpace = layoutState.mNoRecycleSpace;
    if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
        recycleViewsFromEnd(recycler, scrollingOffset, noRecycleSpace);
    } else {
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}

private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset, int noRecycleSpace) {
    final int limit = scrollingOffset - noRecycleSpace;
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
        View child = getChildAt(i);
        if (mOrientationHelper.getDecoratedEnd(child) > limit || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
            // stop here
            recycleChildren(recycler, 0, i);
            return;
        }
    }
}
```
- 如果向上滑动，那么调用recycleViewsFromStart回收顶部滑出界面的item view。
- 如果向下滑动，调用recycleViewsFromEnd回收。
- 以recycleViewsFromStart为例，如果向上滑动，就会回收滑出界面的item view。

#### 3.2.2 remove子view
通过执行recycleChildren方法remove子view。

> LinearLayoutManager

```java
private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
    if (endIndex > startIndex) {
        for (int i = endIndex - 1; i >= startIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    } else {
        for (int i = startIndex; i > endIndex; i--) {
            removeAndRecycleViewAt(i, recycler);
        }
    }
}
```

> RecyclerView.LayoutManager

```java
public void removeAndRecycleViewAt(int index, @NonNull Recycler recycler) {
    final View view = getChildAt(index);
    removeViewAt(index);
    recycler.recycleView(view);
}
```

向上滑动时，从头部开始回收，回收的策略是从RecyclerView中删除child view，在RecyclerView的initChildrenHelper方法中，有覆写ChildHelper.Callback接口的removeViewAt函数。

删除之后通过recycleView方法缓存。

#### 3.2.3 缓存子view
缓存子view通过recycleView方法实现，在RecyclerView类中：
> RecyclerView.Recycler

```java
public void recycleView(@NonNull View view) {
    ViewHolder holder = getChildViewHolderInt(view);
    recycleViewHolderInternal(holder);
}

void recycleViewHolderInternal(ViewHolder holder) {
    boolean cached = false;
    boolean recycled = false;
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            // Retire oldest cached view
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                recycleCachedViewAt(0);
                cachedViewSize--;
            }

            int targetCacheIndex = cachedViewSize;
            if (ALLOW_THREAD_GAP_WORK
                    && cachedViewSize > 0
                    && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                // when adding the view, skip past most recently prefetched views
                int cacheIndex = cachedViewSize - 1;
                while (cacheIndex >= 0) {
                    int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                    if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                        break;
                    }
                    cacheIndex--;
                }
                targetCacheIndex = cacheIndex + 1;
            }
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    }
}
```
- ArrayList<ViewHolder> mCachedViews，这个List中只能存储最多2个ViewHolder，如果大于等于2个，先删除第一个，并不是不要了，而是交给RecycledViewPool托底：

> RecyclerView.Recycler

```java
void recycleCachedViewAt(int cachedViewIndex) {
    ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);
    addViewHolderToRecycledViewPool(viewHolder, true);
    mCachedViews.remove(cachedViewIndex);
}

void addViewHolderToRecycledViewPool(@NonNull ViewHolder holder, boolean dispatchRecycled) {
    if (dispatchRecycled) {
        dispatchViewRecycled(holder);
    }
    getRecycledViewPool().putRecycledView(holder);
}
```

> RecyclerView.RecycledViewPool

```java
public void putRecycledView(ViewHolder scrap) {
    final int viewType = scrap.getItemViewType();
    final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
    if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
        return;
    }
    scrap.resetInternal();
    scrapHeap.add(scrap);
}
SparseArray<ScrapData> mScrap = new SparseArray<>();

static class ScrapData {
    final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
    int mMaxScrap = DEFAULT_MAX_SCRAP;
    long mCreateRunningAverageNs = 0;
    long mBindRunningAverageNs = 0;
}
```
- 加入RecycledViewPool缓存之前，会调用mAdapter.onViewRecycled(holder)通知客户端这个ViewHolder被回收缓存了，可以在自己的Adapter中覆写这个方法做对应的处理。
- ScrapData内部静态类创建了一个mScrapHeap，List类型，存储的是ViewHolder对象，而外部类定义了一个SparseArray，用于存储ScrapData，key是ViewHolder的viewType，也就是说将ViewHolder通过viewType分开存储。
- 存ViewHolder的时候，先获取viewType，并创建一个对应的ArrayList<ViewHolder>，再将要缓存的ViewHolder加进去。
- RecycledViewPool完成缓存之后，mCachedViews删除最后一个ViewHolder。
- 接下来还是会尝试将ViewHolder缓存到mCachedViews中，如果缓存失败，还是交给RecycledViewPool。

#### 3.2.4 创建新的子view
回到fill方法中，在缓存之后，要重新调用layoutChunk方法，去填充向下滑动产生的空间。

> LinearLayoutManager

```java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state, LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    addView(view);
}
```
从next方法出发，跟第一次构建时的调用栈类似，不过这次填充会从缓存里面去取。
> RecyclerView.Recycler

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    if(holder == null){
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    if(holder == null){
        holder = getRecycledViewPool().getRecycledView(type);
    }
    mAdapter.bindViewHolder(holder, offsetPosition);
}
```
getScrapOrHiddenOrCachedHolderForPosition的策略是：
- 先从ArrayList<ViewHolder> mAttachedScrap里面取，mAttachedScrap存储的是那些被hidden，但是仍然attach parent view的child item。
- 从ChildHelper的List<View> mHiddenViews中取那些被hidden但是没有被移除的view，然后通过view获取ViewHolder，顺带把ViewHolder添加到mAttachedScrap中。
- 从ArrayList<ViewHolder> mCachedViews中取。

以上三个缓存中都取不到的话，直接从RecycledViewPool里面取，如果不是第一次创建的话，这里基本上都可以获取到缓存的ViewHolder。

后边通过bindViewHolder通知我们自己Adapter里面的bindViewHolder做数据和view的绑定。

## 4、总结
### 4.1 RecyclerView
- RecyclerView直接控制View的add和remove。
- RecyclerView中创建了Recycler，负责回收，在后续获取ViewHolder的时候，返回缓存的数据。
- RecyclerView中创建了RecyclerViewDataObserver（AdapterDataObserver），用于接收数据变动的通知并处理，处理方式是请求更新（requestLayout）。
- RecyclerView中创建了RecycledViewPool,用于缓存兜底。
- RecyclerView中定义了静态虚类Adapter，用于客户端编写代码继承该类并覆写一些函数，获取数据集大小，创建ViewHolder，将ViewHolder与View数据绑定，回收回调等。
- RecyclerView中定义了静态虚类ViewHolder，它持有一个View对象，并且可以通过这个View反向获取ViewHolder(getChildViewHolderInt)，中间做过渡的是类是LayoutParams，也定义在RecyclerView中，并继承自ViewGroup.MarginLayoutParams。在View创建初期，会将这个LayoutParams设置到View上。ViewHolder还包含了View的位置和ViewType参数。

- RecyclerView中定义了静态虚类LayoutManager，LinearLayoutManager，StaggeredGridLayoutManager都是继承自LayoutManager，它管理着View的状态，中间人是ChildHelper。

### 4.2 流程图
![流程图](https://ftp.bmp.ovh/imgs/2020/06/aaadae68311d14a9.jpg)


微信公众号：![微信公众号](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)



