---
layout: post
title:  "iOS GridView表格视图"
date:   2020-4-29 11:09:22 +0800
categories: iOS
---

> 微信公众号：Android部落格

## 一、背景

电商类应用会有一个九宫格展示分类下的详细信息，如下：
![](https://ftp.bmp.ovh/imgs/2020/04/cd5af6bad847d36a.jpg)

基于九宫格的需求，需要有一个九宫格类的View，当超过一页的时候可以左右滑动。

## 二、需求

对外提供数据接口，可以设置单个item左右上下间隔，可以设置一页中item的行数以及列数。多于一页可以分页展示。

## 三、实现

依赖UIScrollView和UIPageControl两个控件实现上述需求。

### 3.1 定义头文件
头文件对外暴露属性和方法，如下：

> UICustomGridView.h

```objectivec
#import <UIKit/UIKit.h>

@protocol UICustomGridDelegate <NSObject>

@optional
-(void)gridItemSelected:(NSInteger)index;

@end

@interface UICustomGridView : UIView

@property bool showPageIndicator;

@property CGFloat horizontalSpacing;

@property CGFloat verticalSpacing;

@property NSInteger maxItemsPerHorizontal;

@property NSInteger maxItemsPerVertical;

@property (nonatomic, weak) id<UICustomGridDelegate> delegate;

-(void)setChildItems:(NSArray<UIView*> *)items;

@property UIColor *pageIndicatorColor;//default page indicator
@property UIColor *currentPageIndicatorColor;//selected page indicator color

@end
```

- 首先定义了一个代理，用于发送item被点击的消息给订阅者。
- 在接口中定义了一些属性和方法，是否展示分页指示器，水平方向item之间的间隔，竖直方向item之间的间隔，item排列的行数，列数，分页指示器的颜色，还有代理。当然，如果这些参数设置不合理，最终添加item到滚动视图之前还会做一次校验，以便获得最合理的布局。
- setChildItems方法被调用的时候，会按照item尺寸，左右上下的间隔，以及滚动视图的宽高来决定最终item的位置。

### 3.2 实现方法

> UICustomGridView.m

```objectivec
@interface UICustomGridView () <UIScrollViewDelegate>

@property UIScrollView *contentScrollView;
@property UIPageControl *pageControlView;

@property NSInteger pages;

@end
```

先扩展一下UICustomGridView接口，实现UIScrollViewDelegate协议。

同时定义了内部属性，滚动视图以及分页指示器，还有总的页数。

### 3.3 初始化

```objectivec
@implementation UICustomGridView

const CGFloat FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT = 37;
const CGFloat FIX_HORIZONTAL_SPACING = 24;
const CGFloat FIX_VERTICAL_SPACING = 8;

-(instancetype)initWithCoder:(NSCoder *)aDecoder{
    self = [super initWithCoder:aDecoder];
    if(self){
        [self initView:self.frame];
    }
    return self;
}

-(instancetype)initWithFrame:(CGRect)frame{
    self = [super initWithFrame:frame];
    if(self){
        [self initView:frame];
    }
    return self;
}

-(void)initView:(CGRect)frame{
    _pageControlView = [[UIPageControl alloc] initWithFrame:CGRectMake(0, 0, 39, FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT)];
    _pageControlView.currentPage = 0;
    _pageControlView.hidesForSinglePage = true;
    
    _horizontalSpacing = FIX_HORIZONTAL_SPACING;
    _verticalSpacing = FIX_VERTICAL_SPACING;
}
@end
```

初始化兼顾了代码创建以及从xib创建两种方式，定义了默认水平和竖直方向间隔，以及设置了PageControlView的默认高度。

### 3.4 加载子视图

```objectivec
-(void)setChildItems:(NSArray<UIView*> *)items{
    if(!items || items.count == 0){
        return;
    }
    NSInteger itemsCount = items.count;
    
    CGFloat scrollViewHeight = _showPageIndicator ? CGRectGetHeight(self.frame) - FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT:CGRectGetHeight(self.frame);
    CGSize scrollViewSize = CGSizeMake(CGRectGetWidth(self.frame), scrollViewHeight);
    UIView *tempItemView = [items objectAtIndex:0];
    CGSize itemViewSize = tempItemView.frame.size;
    
    if(_maxItemsPerHorizontal == 0 || _maxItemsPerVertical == 0){
        _maxItemsPerHorizontal = floor((float)scrollViewSize.width / (1.5 * _horizontalSpacing + itemViewSize.width));
        _maxItemsPerVertical = floor((float)scrollViewSize.height / (_verticalSpacing + itemViewSize.height));
        NSLog(@"items per vertical and horizontal %d %d",_maxItemsPerVertical , _maxItemsPerHorizontal);
    }
    
    if(_horizontalSpacing == FIX_HORIZONTAL_SPACING || _verticalSpacing == FIX_VERTICAL_SPACING){
        _horizontalSpacing = (scrollViewSize.width - _maxItemsPerHorizontal * itemViewSize.width) / (_maxItemsPerHorizontal + 1);
        _verticalSpacing = (scrollViewSize.height - _maxItemsPerVertical * itemViewSize.height) / (_maxItemsPerVertical + 1);
    }

    NSInteger maxItemsPerPage = _maxItemsPerVertical * _maxItemsPerHorizontal;
    bool needSplitPage = maxItemsPerPage < itemsCount;
    _pages = ceil((float)itemsCount / (_maxItemsPerVertical * _maxItemsPerHorizontal));
    
    _contentScrollView = [[UIScrollView alloc] initWithFrame:CGRectMake(0, 0, scrollViewSize.width, scrollViewSize.height)];
    _contentScrollView.delegate = self;
    _contentScrollView.showsVerticalScrollIndicator = false;
    _contentScrollView.showsHorizontalScrollIndicator = false;
    [_contentScrollView setPagingEnabled:true];
    NSInteger horizontalIndex = 0,verticalIndex = 0;
    NSInteger curPage = 0;

    for(int index = 0;index < itemsCount;index++){
        if(index % _maxItemsPerHorizontal == 0){
            ++verticalIndex;
            horizontalIndex = 0;
        }
        horizontalIndex = index % _maxItemsPerHorizontal;
        if(index % maxItemsPerPage == 0){
            verticalIndex = 0;
            horizontalIndex = 0;
            curPage = index == 0 ? curPage : ++curPage;
        }
        UIView *itemView = [items objectAtIndex:index];
        itemView.tag = index;
        [itemView setUserInteractionEnabled:YES];
        UITapGestureRecognizer *singleFingerTap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleSingleTap:)];
        [itemView addGestureRecognizer:singleFingerTap];
        CGFloat pointX = scrollViewSize.width * curPage + (horizontalIndex + 1) * _horizontalSpacing + horizontalIndex * itemViewSize.width;
        CGFloat pointY = (verticalIndex + 1) * _verticalSpacing + verticalIndex * itemViewSize.height;
        NSLog(@"setChildItems horizontalIndex verticalIndex curPage %d %d %d",horizontalIndex , verticalIndex , curPage);
        NSLog(@"setChildItems point %f %f",pointX , pointY);
        [itemView setFrame:CGRectMake(pointX, pointY, itemViewSize.width, itemViewSize.height)];
        [_contentScrollView addSubview:itemView];
    }
    if(needSplitPage && _showPageIndicator){
        _pageControlView.numberOfPages = _pages;
        _pageControlView.pageIndicatorTintColor = _pageIndicatorColor?_pageIndicatorColor:[UIColor colorWithRed:190.0/255 green:190.0/255 blue:190.0/255 alpha:1];
        _pageControlView.currentPageIndicatorTintColor = _currentPageIndicatorColor?_currentPageIndicatorColor:[UIColor colorWithRed:0 green:0 blue:0 alpha:1];
        
        CGSize pageControlSize = _pageControlView.frame.size;
        [_pageControlView setFrame:CGRectMake(_contentScrollView.center.x - pageControlSize.width / 2, scrollViewSize.height, 39, FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT)];
        _pageControlView.autoresizingMask = UIViewAutoresizingNone;
        NSLog(@"_pageControlView size %f %f",pageControlSize.width , pageControlSize.height);
        NSLog(@"_pageControlView point %f %f",_pageControlView.frame.origin.x , _pageControlView.frame.origin.y);
        [self addSubview:_pageControlView];
    }
    _contentScrollView.contentSize = CGSizeMake(_pages * scrollViewSize.width, scrollViewSize.height);
    [self addSubview:_contentScrollView];
}
```
- 如果不设置单个页面的行数和列数，就通过滚动视图的宽度，高度，分别除以单个item的宽度，高度，同时要加上水平和竖直方向的间隔，得到水平和竖直方向item排列的个数。
- 如果水平和竖直方向间隔是默认的数值，就要根据滚动视图的宽高，子视图的宽度，行列最大item个数，来计算出水平竖直方向每个item之间的间隔。
- 最大页数等于行列排列的个数相乘，如果小于总的item数量，就要分页了，用总数量除以一页最多排布的数目，并向上取整，就是页数了。
- 初始化UIScrollView，设置代理，并实现代理方法，主要是为了拖拽的时候分页。同时隐藏水平和竖直方向的滚动条。
- 将item加入到UIScrollView中，根据水平竖直方向索引，计算出item的x,y坐标，同时为子item添加点击事件。
- 如果需要显示分页指示器的话，就设置分页指示器的总页数，同时设置其默认颜色以及页面选中时的颜色。设置其frame的时候，将中点与UIScrollView的中点对齐，减去自身宽度一半就居中了。
- 最后就是要根据分页的页数乘以UIScrollView的宽度作为内容的宽度，否则不会滑动。

### 3.5 处理滚动分页
UIScrollView的setPagingEnabled属性设置为true，就可以在左右滑动的时候，有分页效果了。重载scrollViewDidEndDecelerating方法，在滑动结束的时候处理分页指示器。如下：

```objectivec
#pragma scrollview delegate
-(void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView{
    float currentOffsetX = _contentScrollView.contentOffset.x;
    int index = (int)(currentOffsetX / _contentScrollView.frame.size.width);
    NSLog(@"scrollViewDidEndDecelerating currentOffsetX = %f,index:%d",currentOffsetX,index);
    _pageControlView.currentPage = index;
}
```
根据内容滑动偏移量，计算出当前页数，并设置给分页指示器。

### 3.6 处理item点击
item被点击之后，获取tag，之前tag被设置成序号，并将其通过协议方法回调给注册者。如下：

```objectivec
- (void)handleSingleTap:(UITapGestureRecognizer *)recognizer{
    NSInteger index = (NSInteger)recognizer.view.tag;
    NSLog(@"tapDetected %d",index);
    if ([self.delegate respondsToSelector:@selector(gridItemSelected:)]) {
        [self.delegate gridItemSelected:index];
    }
}

@end
```

## 四、使用
具体使用方式如下：

```objectivec
-(void)initCustomGridView{
    CGSize currentSize = self.view.frame.size;
    CGPoint currentPoint = _customGridView.frame.origin;
    _destSize = CGSizeMake(88, 120);
    _scale = 1;
    if ([[UIScreen mainScreen] scale] > 1.0) {
        _scale = 2;
        _destSize.width *= 2;
        _destSize.height *= 2;
    }
    
    NSBundle *bundle = [NSBundle bundleWithPath:_bundlePath];
    _myImages = [bundle pathsForResourcesOfType:@"jpeg" inDirectory:nil];
    _customGridView.delegate = self;
    _customGridView.backgroundColor = [UIColor whiteColor];
    [_customGridView setFrame:CGRectMake(currentPoint.x, currentPoint.y, currentSize.width, 293)];
    NSMutableArray *itemsArray = [NSMutableArray new];
    NSInteger imagesCount = _myImages.count;
    for (int index = 0; index < imagesCount; index++) {
        UICustomGridViewCell *itemCell = [[UICustomGridViewCell alloc] initWithFrame:CGRectMake(0, 0, 88, 120)];
        NSString *imgPath= [_myImages objectAtIndex:index];
        NSLog(@"initCell imgPath is %@",imgPath);
        NSData *data = [[NSFileManager defaultManager] contentsAtPath:imgPath];
        UIImage *image = [self downsample:data pointSize:_destSize scale:_scale];
        itemCell.imageView.image = image;
        itemCell.titleLabel.text = @(index).stringValue;
        itemCell.titleLabel.font = [UIFont systemFontOfSize:13];
        itemCell.titleLabel.textColor = [UIColor colorWithRed:0 green:0 blue:0 alpha:1];
        itemCell.titleLabel.textAlignment = NSTextAlignmentCenter;
        [itemsArray addObject:itemCell];
    }
    [_customGridView setChildItems:itemsArray];
}

-(void)gridItemSelected:(NSInteger)index{
    NSLog(@"gridItemSelected index is %d",index);
}
```
- _destSize就是单个item的尺寸，可以根据具体项目需要调整。这里主要将图片资源放到Bundle里面，然后遍历创建子视图。如果图片过大要做缩放处理。具体缩放的方法在之前文章里面有。

- 贴一下UICustomGridViewCell的代码：

> UICustomGridViewCell.h

```
#import <UIKit/UIKit.h>

@interface UICustomGridViewCell : UIView

@property UIImageView *imageView;
@property UILabel *titleLabel;

-(instancetype)initWithCoder:(NSCoder *)aDecoder;
-(instancetype)initWithFrame:(CGRect)frame;

@end
```

> UICustomGridViewCell.m

```objectivec
#import "UICustomGridViewCell.h"

@implementation UICustomGridViewCell

-(instancetype)initWithCoder:(NSCoder *)aDecoder{
    self = [super initWithCoder:aDecoder];
    if(self){
        [self initView:self.frame];
    }
    return self;
}

-(instancetype)initWithFrame:(CGRect)frame{
    self = [super initWithFrame:frame];
    if(self){
        [self initView:frame];
    }
    return self;
}

-(void)initView:(CGRect)frame{
    _imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, frame.size.width, frame.size.width)];
    _titleLabel = [[UILabel alloc] initWithFrame:CGRectMake(0, frame.size.width, frame.size.width, 32)];
    [self addSubview:_imageView];
    [self addSubview:_titleLabel];
}

@end

```

## 五、最后一公里

> UICustomGridView.h

```objectivec
#import <UIKit/UIKit.h>

@protocol UICustomGridDelegate <NSObject>

@optional
-(void)gridItemSelected:(NSInteger)index;

@end

@interface UICustomGridView : UIView

@property bool showPageIndicator;

@property CGFloat horizontalSpacing;

@property CGFloat verticalSpacing;

@property NSInteger maxItemsPerHorizontal;

@property NSInteger maxItemsPerVertical;

@property (nonatomic, weak) id<UICustomGridDelegate> delegate;

-(void)setChildItems:(NSArray<UIView*> *)items;

@property UIColor *pageIndicatorColor;//default page indicator
@property UIColor *currentPageIndicatorColor;//selected page indicator color

@end
```

> UICustomGridView.m

```objectivec
#import "UICustomGridView.h"

@interface UICustomGridView () <UIScrollViewDelegate>

@property UIScrollView *contentScrollView;
@property UIPageControl *pageControlView;

@property NSInteger pages;

@end

@implementation UICustomGridView

const CGFloat FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT = 37;
const CGFloat FIX_HORIZONTAL_SPACING = 24;
const CGFloat FIX_VERTICAL_SPACING = 8;

-(instancetype)initWithCoder:(NSCoder *)aDecoder{
    self = [super initWithCoder:aDecoder];
    if(self){
        [self initView:self.frame];
    }
    return self;
}

-(instancetype)initWithFrame:(CGRect)frame{
    self = [super initWithFrame:frame];
    if(self){
        [self initView:frame];
    }
    return self;
}

-(void)initView:(CGRect)frame{
    _pageControlView = [[UIPageControl alloc] initWithFrame:CGRectMake(0, 0, 39, FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT)];
    _pageControlView.currentPage = 0;
    _pageControlView.hidesForSinglePage = true;
    
    _showPageIndicator = true;
    
    _horizontalSpacing = FIX_HORIZONTAL_SPACING;
    _verticalSpacing = FIX_VERTICAL_SPACING;
}

-(void)setChildItems:(NSArray<UIView*> *)items{
    if(!items || items.count == 0){
        return;
    }
    NSInteger itemsCount = items.count;
    
    CGFloat scrollViewHeight = _showPageIndicator ? CGRectGetHeight(self.frame) - FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT:CGRectGetHeight(self.frame);
    CGSize scrollViewSize = CGSizeMake(CGRectGetWidth(self.frame), scrollViewHeight);
    UIView *tempItemView = [items objectAtIndex:0];
    CGSize itemViewSize = tempItemView.frame.size;
    
    if(_maxItemsPerHorizontal == 0 || _maxItemsPerVertical == 0){
        _maxItemsPerHorizontal = floor((float)scrollViewSize.width / (1.5 * _horizontalSpacing + itemViewSize.width));
        _maxItemsPerVertical = floor((float)scrollViewSize.height / (_verticalSpacing + itemViewSize.height));
        NSLog(@"items per vertical and horizontal %d %d",_maxItemsPerVertical , _maxItemsPerHorizontal);
    }
    
    if(_horizontalSpacing == FIX_HORIZONTAL_SPACING || _verticalSpacing == FIX_VERTICAL_SPACING){
        _horizontalSpacing = (scrollViewSize.width - _maxItemsPerHorizontal * itemViewSize.width) / (_maxItemsPerHorizontal + 1);
        _verticalSpacing = (scrollViewSize.height - _maxItemsPerVertical * itemViewSize.height) / (_maxItemsPerVertical + 1);
    }

    NSInteger maxItemsPerPage = _maxItemsPerVertical * _maxItemsPerHorizontal;
    bool needSplitPage = maxItemsPerPage < itemsCount;
    _pages = ceil((float)itemsCount / (_maxItemsPerVertical * _maxItemsPerHorizontal));
    
    _contentScrollView = [[UIScrollView alloc] initWithFrame:CGRectMake(0, 0, scrollViewSize.width, scrollViewSize.height)];
    _contentScrollView.delegate = self;
    _contentScrollView.showsVerticalScrollIndicator = false;
    _contentScrollView.showsHorizontalScrollIndicator = false;
    [_contentScrollView setPagingEnabled:true];
    NSInteger horizontalIndex = 0,verticalIndex = 0;
    NSInteger curPage = 0;

    for(int index = 0;index < itemsCount;index++){
        if(index % _maxItemsPerHorizontal == 0){
            ++verticalIndex;
            horizontalIndex = 0;
        }
        horizontalIndex = index % _maxItemsPerHorizontal;
        if(index % maxItemsPerPage == 0){
            verticalIndex = 0;
            horizontalIndex = 0;
            curPage = index == 0 ? curPage : ++curPage;
        }
        UIView *itemView = [items objectAtIndex:index];
        itemView.tag = index;
        [itemView setUserInteractionEnabled:YES];
        UITapGestureRecognizer *singleFingerTap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleSingleTap:)];
        [itemView addGestureRecognizer:singleFingerTap];
        CGFloat pointX = scrollViewSize.width * curPage + (horizontalIndex + 1) * _horizontalSpacing + horizontalIndex * itemViewSize.width;
        CGFloat pointY = (verticalIndex + 1) * _verticalSpacing + verticalIndex * itemViewSize.height;
        NSLog(@"setChildItems horizontalIndex verticalIndex curPage %d %d %d",horizontalIndex , verticalIndex , curPage);
        NSLog(@"setChildItems point %f %f",pointX , pointY);
        [itemView setFrame:CGRectMake(pointX, pointY, itemViewSize.width, itemViewSize.height)];
        [_contentScrollView addSubview:itemView];
    }
    if(needSplitPage && _showPageIndicator){
        _pageControlView.numberOfPages = _pages;
        _pageControlView.pageIndicatorTintColor = _pageIndicatorColor?_pageIndicatorColor:[UIColor colorWithRed:190.0/255 green:190.0/255 blue:190.0/255 alpha:1];
        _pageControlView.currentPageIndicatorTintColor = _currentPageIndicatorColor?_currentPageIndicatorColor:[UIColor colorWithRed:0 green:0 blue:0 alpha:1];
        
        CGSize pageControlSize = _pageControlView.frame.size;
        [_pageControlView setFrame:CGRectMake(_contentScrollView.center.x - pageControlSize.width / 2, scrollViewSize.height, 39, FIX_GRID_PAGE_CONTROL_VIEW_HEIGHT)];
        _pageControlView.autoresizingMask = UIViewAutoresizingNone;
        NSLog(@"_pageControlView size %f %f",pageControlSize.width , pageControlSize.height);
        NSLog(@"_pageControlView point %f %f",_pageControlView.frame.origin.x , _pageControlView.frame.origin.y);
        [self addSubview:_pageControlView];
    }
    _contentScrollView.contentSize = CGSizeMake(_pages * scrollViewSize.width, scrollViewSize.height);
    [self addSubview:_contentScrollView];
}

#pragma scrollview delegate
-(void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView{
    float currentOffsetX = _contentScrollView.contentOffset.x;
    int index = (int)(currentOffsetX / _contentScrollView.frame.size.width);
    NSLog(@"scrollViewDidEndDecelerating currentOffsetX = %f,index:%d",currentOffsetX,index);
    _pageControlView.currentPage = index;
}


- (void)handleSingleTap:(UITapGestureRecognizer *)recognizer{
    NSInteger index = (NSInteger)recognizer.view.tag;
    NSLog(@"tapDetected %d",index);
    if ([self.delegate respondsToSelector:@selector(gridItemSelected:)]) {
        [self.delegate gridItemSelected:index];
    }
}

@end
```

微信公众号：![](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)
