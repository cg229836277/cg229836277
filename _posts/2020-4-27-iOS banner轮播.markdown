---
layout: post
title:  "iOS banner轮播"
date:   2020-4-27 11:09:22 +0800
categories: iOS
---

> 微信公众号：Android部落格

# 一、背景
电商App一般都会存在图片轮播的场景，而iOS没有轮播UI组件，因此需要自定义一个UI组件以适应项目需要。

# 二、框架
整体框架是UIScrollView作为父视图，在视图中添加多个子视图，同时设置好子视图的frame，再设置滚动视图的内容宽度，让UISCrollView能够左右滑动。接下来添加一个定时器，按照设定的播放时间间隔重复触发，循环播放子UIView即可。

# 三、定义属性
将对外暴露的属性定义到一个CustomBanner.h文件中，如下：

```objectivec
#import <UIKit/UIKit.h>

@protocol UICustomBannerDelegate <NSObject>

@optional
-(void)pageSelected:(int)index;

@end

@interface CustomBanner : UIView

@property bool showIndicator;//whether show bottom page control view
@property UIColor *indicatorColor;//default page indicator
@property UIColor *currentIndicatorColor;//selected page indicator color
@property float playIntervalTime;//images or some other uiviews interval play time
@property bool showScrollIndicator;//whether show scroll view indicator,including horizontal and vertical

@property (nonatomic, weak) id<UICustomBannerDelegate> delegate;

-(void)start:(NSArray *)views;
-(void)stop;

@end
```

- 首先定义了一个代理，用于对外发送当前播放view的序号，回调方法可选。
- 有一些项目的banner需要在播放的图片下边展示指示器，用于指示当前view被选中了，这里先提供是否展示的属性，如果可以展示的话，再提供默认和选中时指示器的颜色。这里的指示器使用cocoa touch默认提供的组件UIPageControl。
- 设置播放间隔，也就是设置定时器的触发间隔。
- 是否展示滚动视图的滚动条，一般是不显示。
- 对外提供代理设置对象。
- start方法对外暴露，用于提供一组播放UIView，调用这个方法的同时，会将子视图添加到UIScrollView中。
- stop方法用于在一些情景下停止播放。

# 四、实现

## 4.1 扩展
实现放在CustomBanner.m文件中，先扩展一下接口CustomBanner：

```objectivec
@interface CustomBanner () <UIScrollViewDelegate>

@property NSTimer *timer;
@property CGSize currentViewSize;

@property UIScrollView *contentScrollView;
@property UIPageControl *pageControlView;
@property CGSize scrollViewSize;

@end
```
这里定义了一个NSTimer对象，用于实现定时器，currentViewSize用于获取当前父视图的大小，用来调整UIScrollView的大小，contentScrollView是可滚动视图，pageControlView用来做指示器，scrollViewSize获取的是contentScrollView的大小，用来约束pageControlView的位置。

## 4.2 初始化
初始化如下：

```objectivec
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
    _contentScrollView = [[UIScrollView alloc] init];
    _contentScrollView.delegate = self;
    [_contentScrollView setPagingEnabled:true];
    
    _pageControlView = [[UIPageControl alloc] init];
    _pageControlView.currentPage = 0;
    _pageControlView.hidesForSinglePage = true;
}
```
当从xib或storyboard里面加载的时候，会调用initWithCoder；如果直接用代码创建的话，就走initWithFrame，最终会走到initView方法，在这个方法里面，初始化了_contentScrollView和_pageControlView，这里暂时先不设置frame，当添加子视图的时候开始设置frame。另外要把_contentScrollView的setPagingEnabled属性设置为true，否则拖动的时候就没有分页效果了。

## 4.3 设置定时器
定时器用于定时触发，触发的时候修改当前展示的view，并将当前view的序号回调给消息注册者：

```objectivec
-(void)startTimer{
    _timer = [NSTimer timerWithTimeInterval:_playIntervalTime == 0?2:_playIntervalTime
                                     target:self
                                   selector:@selector(updateTimer:)
                                   userInfo:nil
                                    repeats:YES];
    [[NSRunLoop currentRunLoop] addTimer:_timer forMode:NSDefaultRunLoopMode];
}
```
定时间隔不设置的话，默认2s。

看看触发的方法：

```objectivec
- (void)updateTimer:(NSTimer *)timer {
    NSLog(@"updateTimer");
    ++default_init_index;
    if(default_init_index == CHILD_ITEM_COUNT){
        default_init_index = 0;
    }
    [self startScroll];
}
```
这里修改初始化子view的序号，不断累加，并重置。

## 4.4 添加子视图
添加子视图之后需要约束子视图的位置，以及设置UIScrollView和PageControl的一些属性，如下：

```objectivec
-(void)start:(NSArray *)views{
    if(!views || views.count == 0){
        return;
    }
    [self customInit];
    NSInteger count = views.count;
    CHILD_ITEM_COUNT = count;
    _pageControlView.numberOfPages = count;
    CGRect pageScrollRect = _pageControlView.frame;
    [_pageControlView setFrame:CGRectMake(pageScrollRect.origin.x - pageScrollRect.size.width / 2, _scrollViewSize.height, 39, FIX_PAGE_CONTROL_VIEW_HEIGHT)];
    
    for(int index = 0; index < count;index++){
        UIView *childView = [views objectAtIndex:index];
        float childViewPointX = index * _scrollViewSize.width;
        [childView setFrame:CGRectMake(childViewPointX, 0, _scrollViewSize.width, _scrollViewSize.height)];
        [_contentScrollView addSubview:childView];
    }
    [_contentScrollView setContentSize:CGSizeMake(_scrollViewSize.width * count, _scrollViewSize.height)];
    [self startTimer];
}
```
- 先获取子视图的个数，个数就是_pageControlView要控制的页面数
- 调整_pageControlView的位置。整个_pageControlView应该是居中的，因此这里的逻辑是将他的center和_contentScrollView的center对齐，然后减去_pageControlView宽度的一半就可以整体居中了。y坐标就是_contentScrollView的高度就行了。
- 将子视图添加到_contentScrollView，并随后设置他的ContentSize属性，如果不设置的话，就不能左右滑动了。宽度是父视图宽度乘以子view的个数，高度就是父视图高度。

## 4.5 调整视图
在customInit方法中，将_pageControlView和_contentScrollView的宽高做了一些调整：

```objectivec
-(void)customInit{
    _currentViewSize = self.frame.size;
    NSLog(@"size width height %f %f ",_currentViewSize.width,_currentViewSize.height);
    CGRect scrollViewRect = CGRectMake(0, 0, _currentViewSize.width, _showIndicator?_currentViewSize.height - FIX_PAGE_CONTROL_VIEW_HEIGHT:_currentViewSize.height);
    [_contentScrollView setFrame:scrollViewRect];
    _scrollViewSize = scrollViewRect.size;
    
    if(!_showScrollIndicator){
        _contentScrollView.showsHorizontalScrollIndicator = false;
        _contentScrollView.showsVerticalScrollIndicator = false;
    }
    
    if(_showIndicator){
        _pageControlView.pageIndicatorTintColor = _indicatorColor?_indicatorColor:[UIColor colorWithRed:190.0/255 green:190.0/255 blue:190.0/255 alpha:1];
        _pageControlView.currentPageIndicatorTintColor = _currentIndicatorColor?_indicatorColor:[UIColor colorWithRed:0 green:0 blue:0 alpha:1];
        _pageControlView.currentPage = 0;
        _pageControlView.hidesForSinglePage = true;
        _pageControlView.center = _contentScrollView.center;
        [_pageControlView setFrame:CGRectMake(_contentScrollView.center.x, _scrollViewSize.height, 39, FIX_PAGE_CONTROL_VIEW_HEIGHT)];
    }
    [self addSubview:_contentScrollView];
    if(_showIndicator){
        [self addSubview:_pageControlView];
    }
}
```
- 重新设置_contentScrollView的frame属性是因为，在调用的时候，将这个banner放到storyboard里面添加与父视图等宽的约束之后，发现banner的宽度总是小于模拟器设备的宽度。

    通过搜索发现，如果在UIScrollView里面添加一个UIView作为contentView，并对这个view设置等宽，左右间距为0，centerX,centerY，然后在UIScrollView的size inspector视图中将intrinsic size设置为placeholder之后，UIScrollView的宽度就与父视图等宽了。这里不会使用xib去初始化UIScrollView，所以在start之前还是要设置一下banner的frame，后面讲怎么用的时候会讲到这个问题。

- 接下来分别处理了是否展示滚动条和展示分页指示器，如果不展示分页指示器的话，就不将其放到父视图里面了。

## 4.6 开始播放
通过定时器默认启动播放，如下：

```objectivec
-(void)startScroll{
    if ([self.delegate respondsToSelector:@selector(pageSelected:)]) {
         [self.delegate pageSelected:default_init_index];
     }
    _pageControlView.currentPage = default_init_index;
    int offsetIndex = default_init_index % CHILD_ITEM_COUNT;
    float currentPointX = offsetIndex * _currentViewSize.width;
    [_contentScrollView setContentOffset:CGPointMake(currentPointX, 0) animated:true];
}
```
每次滚动的时候，重新算一个当前的序号，并回调给消息注册者。其实滚动的实现依靠设置UIScrollView的内容偏移量。

## 4.7 处理手势拖拽
在上面扩展章节，可以看到我们继承了UIScrollViewDelegate，因此在.m文件中，可以定义相关代理方法处理回调，如下：

```objectivec
#pragma scrollview delegate
-(void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView{
    float currentOffsetX = _contentScrollView.contentOffset.x;
    default_init_index = (int)(currentOffsetX / _currentViewSize.width);
    NSLog(@"scrollViewDidEndDecelerating currentOffsetX = %f,default_init_index:%d",currentOffsetX,default_init_index);
    _pageControlView.currentPage = default_init_index;
    if ([self.delegate respondsToSelector:@selector(pageSelected:)]) {
        [self.delegate pageSelected:default_init_index];
    }
    [self startTimer];
}

-(void)scrollViewWillBeginDragging:(UIScrollView *)scrollView{
    NSLog(@"scrollViewWillBeginDragging");
    [self stop];
}
#pragma scrollview delegate
```
- scrollViewDidEndDecelerating 意思就是滚动已经结束了，在这里我们重新计算了当前子view的序号，并把最新的序号设置给_pageControlView，并发送页面变动消息给注册者。处理完毕重新开启定时器。
- scrollViewWillBeginDragging 用户开始拖拽了，这时候需要将定时器取消，以用户的操作为主

自此所有实现已经完成。

# 五、使用

在ViewController中既可以通过代码初始化也可以通过storyboard添加一个UIView并将其属性设置为CustomBanner。以后者为例，看看在ViewController中怎么调用的。
## 5.1 引入代理
在ViewController中添加代理

```objectivec
@interface ViewController ()<UIScrollViewDelegate , UICustomBannerDelegate>

@property (strong, nonatomic) IBOutlet CustomBanner *customBanner;

@end
```

## 5.2 添加子视图
在viewWillAppear回调方法中开始添加子视图，这里以添加UIImageView举例说明：

```objectivec
-(void)viewWillAppear:(BOOL)animated{
    CGSize currentSize = self.view.frame.size;
    NSLog(@"parent size %f %f",currentSize.width,currentSize.height);
    _customBanner.showIndicator = true;
    _customBanner.delegate = self;
    NSArray *imagesUrl = [[NSArray alloc] initWithObjects:@"a1.jpg",
                       @"a2.jpg",
                       @"a3.jpeg",
                       @"a4.jpeg",nil];
    double bannerViewHeight = _customBanner.frame.size.height;
    NSMutableArray *imageViewArray = [NSMutableArray new];
    
    CGSize destSize = CGSizeMake(currentSize.width, 200);
    CGFloat scale = 1;
    if ([[UIScreen mainScreen] scale] > 1.0) {
        scale = 2;
        destSize.width *= 2;
        destSize.height *= 2;
    }
    
    NSString *bundlePath = [[NSBundle mainBundle] pathForResource:@"banner" ofType:@"bundle"];
    for(int index = 0;index < imagesUrl.count;index++){
        NSString *imgPath= [bundlePath stringByAppendingPathComponent:[imagesUrl objectAtIndex:index]];
        NSLog(@"imgPath is %@",imgPath);
        NSData *data = [[NSFileManager defaultManager] contentsAtPath:imgPath];
        UIImage *image = [self downsample:data pointSize:destSize scale:scale];
        UIImageView *imageView = [[UIImageView alloc] initWithFrame:CGRectMake(0, 0, currentSize.width, bannerViewHeight)];
        imageView.image = image;
        [imageViewArray addObject:imageView];
    }
    
    CGPoint bannerPoint = CGPointMake(_customBanner.frame.origin.x, _customBanner.frame.origin.y);
    [_customBanner setFrame:CGRectMake(bannerPoint.x, bannerPoint.y, currentSize.width, _customBanner.frame.size.height)];
    [_customBanner start:imageViewArray];
}
```
- 在customBanner调用start方法之前，先设置一下frame，目的是设置自身的宽度，避免其宽度与父视图的宽度不一致。

- 当banner轮播图片的分辨率远远小于banner本身的尺寸时，要对图片做缩放处理，以当前测试的四张高分辨率的图片来说，就消耗100M以上的内存。因为图片在被加载之前是压缩格式，正式加载之前还要解压缩，再渲染到控件上，对CPU和内存都是不小的考验。这里使用了苹果推荐的downsampling方法，如下：

```objectivec
-(UIImage*)downsample:(NSObject *)imageSource pointSize:(CGSize)pointSize scale:(CGFloat)scale{
    CGFloat maxw = pointSize.width;
    CGFloat maxh = pointSize.height;
    
    CGImageSourceRef src = NULL;
    if ([imageSource isKindOfClass:[NSURL class]])
        src = CGImageSourceCreateWithURL((__bridge CFURLRef)imageSource, nil);
    else if ([imageSource isKindOfClass:[NSData class]])
        src = CGImageSourceCreateWithData((__bridge CFDataRef)imageSource, nil);
    
    // load the image at the desired size
    NSDictionary* d = @{
                        (id)kCGImageSourceShouldAllowFloat: (id)kCFBooleanTrue,
                        (id)kCGImageSourceCreateThumbnailWithTransform: (id)kCFBooleanTrue,
                        (id)kCGImageSourceCreateThumbnailFromImageAlways: (id)kCFBooleanTrue,
                        (id)kCGImageSourceThumbnailMaxPixelSize: @((int)(maxw > maxh ? maxw : maxh))
                        };
    CGImageRef imref = CGImageSourceCreateThumbnailAtIndex(src, 0, (__bridge CFDictionaryRef)d);
    if (NULL != src)
        CFRelease(src);
    UIImage* im = [UIImage imageWithCGImage:imref scale:scale orientation:UIImageOrientationUp];
    if (NULL != imref)
        CFRelease(imref);
    return im;
}
```
这样处理图片之后，内存消耗明显降低。

上面的四张图片是放到自定义的bundle里面引用的。因为是自定义的bundle，这里需要注意的是，自定义bundle build之后，需要在主工程中将这个bundle添加到主bundle里去，否则调用`[NSBundle mainBundle]`方法的时候返回nil。添加路径是：选中主工程/build phases/copy bundle resources，点击+号添加。

# 六、最后一公里

> CustomBanner.h

```objectivec
#import <UIKit/UIKit.h>

@protocol UICustomBannerDelegate <NSObject>

@optional
-(void)pageSelected:(int)index;

@end

@interface CustomBanner : UIView

@property bool showIndicator;//whether show bottom page control view
@property UIColor *indicatorColor;//default page indicator
@property UIColor *currentIndicatorColor;//selected page indicator color
@property float playIntervalTime;//images or some other uiviews interval play time
@property bool showScrollIndicator;//whether show scroll view indicator,including horizontal and vertical

@property (nonatomic, weak) id<UICustomBannerDelegate> delegate;

-(void)start:(NSArray *)views;
-(void)stop;

@end
```
> CustomBanner.m

```objectivec
#import "CustomBanner.h"

@interface CustomBanner () <UIScrollViewDelegate>

//@property NSTimer *timer;
@property dispatch_source_t timer;

@property CGSize currentViewSize;

@property UIScrollView *contentScrollView;
@property UIPageControl *pageControlView;
@property CGSize scrollViewSize;

@end


@implementation CustomBanner

NSInteger CHILD_ITEM_COUNT = 0;
const CGFloat FIX_PAGE_CONTROL_VIEW_HEIGHT = 37;
int default_init_index = 0;

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
    _contentScrollView = [[UIScrollView alloc] init];
    _contentScrollView.delegate = self;
    [_contentScrollView setPagingEnabled:true];
    
    _pageControlView = [[UIPageControl alloc] init];
    _pageControlView.currentPage = 0;
    _pageControlView.hidesForSinglePage = true;
    
    dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
}

-(void)customInit{
    _currentViewSize = self.frame.size;
    NSLog(@"size width height %f %f ",_currentViewSize.width,_currentViewSize.height);
    CGRect scrollViewRect = CGRectMake(0, 0, _currentViewSize.width, _showIndicator?_currentViewSize.height - FIX_PAGE_CONTROL_VIEW_HEIGHT:_currentViewSize.height);
    [_contentScrollView setFrame:scrollViewRect];
    _scrollViewSize = scrollViewRect.size;
    
    if(!_showScrollIndicator){
        _contentScrollView.showsHorizontalScrollIndicator = false;
        _contentScrollView.showsVerticalScrollIndicator = false;
    }
    
    if(_showIndicator){
        _pageControlView.pageIndicatorTintColor = _indicatorColor?_indicatorColor:[UIColor colorWithRed:190.0/255 green:190.0/255 blue:190.0/255 alpha:1];
        _pageControlView.currentPageIndicatorTintColor = _currentIndicatorColor?_indicatorColor:[UIColor colorWithRed:0 green:0 blue:0 alpha:1];
        _pageControlView.currentPage = 0;
        _pageControlView.hidesForSinglePage = true;
        _pageControlView.center = _contentScrollView.center;
        [_pageControlView setFrame:CGRectMake(_contentScrollView.center.x, _scrollViewSize.height, 39, FIX_PAGE_CONTROL_VIEW_HEIGHT)];
    }
    [self addSubview:_contentScrollView];
    if(_showIndicator){
        [self addSubview:_pageControlView];
    }
}

-(void)start:(NSArray *)views{
    if(!views || views.count == 0){
        return;
    }
    [self customInit];
    NSInteger count = views.count;
    CHILD_ITEM_COUNT = count;
    _pageControlView.numberOfPages = count;
    CGRect pageScrollRect = _pageControlView.frame;
    [_pageControlView setFrame:CGRectMake(pageScrollRect.origin.x - pageScrollRect.size.width / 2, _scrollViewSize.height, 39, FIX_PAGE_CONTROL_VIEW_HEIGHT)];
    
    for(int index = 0; index < count;index++){
        UIView *childView = [views objectAtIndex:index];
        float childViewPointX = index * _scrollViewSize.width;
        [childView setFrame:CGRectMake(childViewPointX, 0, _scrollViewSize.width, _scrollViewSize.height)];
        [_contentScrollView addSubview:childView];
    }
    [_contentScrollView setContentSize:CGSizeMake(_scrollViewSize.width * count, _scrollViewSize.height)];
    dispatch_source_set_timer(_timer, dispatch_walltime(NULL, 0), (_playIntervalTime == 0?2:_playIntervalTime) * NSEC_PER_SEC, 0);
    dispatch_source_set_event_handler(_timer, ^{
        dispatch_async(dispatch_get_main_queue(), ^{
            // 在主线程中实现需要的功能
            [self updateTimer];
        });
     });
    [self startTimer];
}

-(void)startTimer{
    dispatch_resume(_timer);
//    _timer = [NSTimer timerWithTimeInterval:_playIntervalTime == 0?2:_playIntervalTime
//                                     target:self
//                                   selector:@selector(updateTimer:)
//                                   userInfo:nil
//                                    repeats:YES];
//    [[NSRunLoop currentRunLoop] addTimer:_timer forMode:NSDefaultRunLoopMode];
}

- (void)updateTimer {
//    NSLog(@"updateTimer");
    ++default_init_index;
    if(default_init_index == CHILD_ITEM_COUNT){
        default_init_index = 0;
    }
    [self startScroll];
}

-(void)startScroll{
    if ([self.delegate respondsToSelector:@selector(pageSelected:)]) {
         [self.delegate pageSelected:default_init_index];
     }
    _pageControlView.currentPage = default_init_index;
    int offsetIndex = default_init_index % CHILD_ITEM_COUNT;
    float currentPointX = offsetIndex * _currentViewSize.width;
    [_contentScrollView setContentOffset:CGPointMake(currentPointX, 0) animated:true];
}

-(void)stop{
    if(_timer){
        NSLog(@"stop Timer");
       dispatch_source_cancel(_timer);
        _timer = nil;
    }
}

#pragma scrollview delegate
-(void)scrollViewDidEndDecelerating:(UIScrollView *)scrollView{
    float currentOffsetX = _contentScrollView.contentOffset.x;
    default_init_index = (int)(currentOffsetX / _currentViewSize.width);
    NSLog(@"scrollViewDidEndDecelerating currentOffsetX = %f,default_init_index:%d",currentOffsetX,default_init_index);
    _pageControlView.currentPage = default_init_index;
    if ([self.delegate respondsToSelector:@selector(pageSelected:)]) {
        [self.delegate pageSelected:default_init_index];
    }
    [self startTimer];
}

-(void)scrollViewWillBeginDragging:(UIScrollView *)scrollView{
    NSLog(@"scrollViewWillBeginDragging");
    dispatch_suspend(_timer);
}
#pragma scrollview delegate

@end
```

微信公众号：![](https://ftp.bmp.ovh/imgs/2020/04/f472fa7e9c62a8c7.jpg)
