---
layout: post
title: CoreAnimation动画（一）：遮罩动画/注水动画
date: 2016-04-25 23:54:40 +0800
tags: CoreAnimation
---
## 遮罩动画/注水动画

一般用CoreAnimation+mask 来实现其动画效果。
mask指lyaer.mask属性，等同于UIView的clipsToBounds属性，将超出自身范围外的内容剪裁掉，不显示。

来看动画效果：

![easymaskanimating8.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating8.gif)

注水动画的原理（很简单的）：
有两个layer层，一个图形层，一个遮罩层。用遮罩层去逐渐覆盖背景图形层，达到动画的效果。

本着由易到难的原则，先看看最简单的动画如何去做。

![easymaskanimating.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating.gif)

这是一个类似进度条的动画（丑了点），绿色是遮盖层，橙色是背景图形层。可以看到绿色的遮罩层逐渐的覆盖了橙色的背景层。类似进度由0到1的过程。
那么，这个动画如何实现呢？
根据条件的不同，动画实现的方式也不同：

 - 不用控制进度，只要完成动画就好
 - 需要控制进度，还要控制动画的快慢

首先，不需要控制进度的实现。在一定时间内，直接用遮罩层去覆盖背景图层好了。这里插一句：

>Core Animation 维护了两个平行 layer 层次结构： model layer tree（模型层树） 和 presentation layer tree（表示层树）。前者中的 layers 反映了我们能直接看到的 layers 的状态，而后者的 layers 则是动画正在表现的值的近似。


当动画运行结束后，model layer的值并不会被改变，所以添加动画的图层仍然会回到初始位置。但当我们想要动画结束后，动画停留在结束的位置，该怎么办呢？两种方法：

 - 设定动画removedOnCompletion属性为NO，在图层添加动画之后（即[layer addAnimation:XXX forKey:ni]之后），手动修改model layer的结束位置，下面的示例就是用的这种方法.
 - 图层添加动画之前，设定removedOnCompletion属性为YES，同时设定animation.fillMode = kCAFillModeForward;

参考[这里](http://objccn.io/issue-12-1/)

动画：

![easymaskanimating.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating.gif)

keyPath=@"position.x"的实现方法：

```objc
    //position.x animation
    CALayer *pContainerLayer = [CALayer layer];
    pContainerLayer.frame = CGRectMake(200, 50, 100, 50);
    pContainerLayer.backgroundColor = [[UIColor orangeColor] CGColor];
    [self.view.layer addSublayer:pContainerLayer];
    
    CALayer *pCoverLayer = [CALayer layer];
    pCoverLayer.frame = CGRectMake(0 - 100 , 0, 100, 50);
    pCoverLayer.backgroundColor = [[UIColor greenColor] CGColor];
    [pContainerLayer addSublayer:pCoverLayer];

    CGFloat pToX = 100 / 2;
    CABasicAnimation *pAnimation = [CABasicAnimation animationWithKeyPath:@"position.x"];;
    pAnimation.fromValue = @(pCoverLayer.position.x);
    pAnimation.toValue = @(pToX);
    pAnimation.duration = 5.0f;
    pAnimation.repeatCount = 1;
    pAnimation.removedOnCompletion = YES;

    [pCoverLayer addAnimation:pAnimation forKey:nil];

```

动画：

![easymaskanimating1.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating1.gif)

keyPath=@"bounds.size.width"的实现方法：宽度由 0->100

```objc
    //bounds.size.width animation
    CALayer *bContainerLayer = [CALayer layer];
    bContainerLayer.frame = CGRectMake(200, 110, 100, 50);
    bContainerLayer.backgroundColor = [[UIColor orangeColor] CGColor];
    [self.view.layer addSublayer:bContainerLayer];
    
    CALayer *bCoverLayer = [CALayer layer];
    bCoverLayer.frame = CGRectMake(0, 0, 0, 50);
    bCoverLayer.backgroundColor = [[UIColor greenColor] CGColor];
    bCoverLayer.anchorPoint = CGPointMake(0, 0.5);
    [bContainerLayer addSublayer:bCoverLayer];
   
    CGFloat bToWidth = 100;
    CABasicAnimation *bAnimation = [CABasicAnimation animationWithKeyPath:@"bounds.size.width"];
    bAnimation.fromValue = @(0);
    bAnimation.toValue = @(bToWidth);
    bAnimation.duration = 5.0f;
    bAnimation.repeatCount = 1;
    bAnimation.removedOnCompletion = YES;
    
    [bCoverLayer addAnimation:bAnimation forKey:nil];
    // change model layer bounds
    bCoverLayer.bounds = CGRectMake(0, 0, 100, 50);
```

其次，需要控制进度了，上面的方法就没法使用了。
动画：

![easymaskanimating2.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating2.gif)

主要实现变化，其中速度就是slider从0到1的滑动速度。

slider: 0 -> 1

coverLayer width: 0 -> max(宽度的最大值)

```objc
@interface ViewController ()
@property (nonatomic,strong) CALayer *coverLayer;
@property (weak, nonatomic) IBOutlet UISlider *slider;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    CALayer *sContainerLayer = [CALayer layer];
    sContainerLayer.frame = CGRectMake(200, 110, 100, 50);
    sContainerLayer.backgroundColor = [[UIColor orangeColor] CGColor];
    [self.view.layer addSublayer:sContainerLayer];
        
    CALayer *sCoverLayer = [CALayer layer];
    sCoverLayer.frame = CGRectMake(0, 0, 0, 50);
    sCoverLayer.backgroundColor = [[UIColor greenColor] CGColor];
    sCoverLayer.anchorPoint = CGPointMake(0, 0.5);
    [sContainerLayer addSublayer:sCoverLayer];
    self.coverLayer = sCoverLayer;
        
    [self.slider addTarget:self action:@selector(update:) forControlEvents:UIControlEventValueChanged];
}
//这里的slider.value * 100，100是背景图层的宽度
- (void)update:(UISlider *)slider {
    self.coverLayer.frame = CGRectMake(0, 0, slider.value * 100, 50);
}

@end
```

下面来个稍微复杂点的，加入layer的mask属性：

- 一种背景图层使用mask属性，遮罩层不变，还是一个矩形，然后覆盖的时候，只能显示背景图层的形状，超出的部分被剪裁掉
- 另一种mask使用特殊图形，遮盖的时候，显示的是mask的图层的样子
动画：

![easymaskanimating3.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating3.gif)

这是一个不规则图形的注水动画，其中背景图层不规则，遮罩图层是一个矩形。遮罩图层颜色是白色，背景图层是橙色。动画的运动方向是，白色图层从下到上，这里对应的是size.heigh 由 max(94) -> 0 。
如何生成动画中的背景图层图形：

- CAShapeLayer利用属性path，用贝塞尔创建需要的图形。工具paintCode非常好用。
- 设定背景图层的mask属性为上一步生成的shapelayer。这样就生成特定图形的图层了。

```objc
    CALayer *canvasLayer = [CALayer layer];
    canvasLayer.frame = CGRectMake(200, 80, 52, 94);
    canvasLayer.backgroundColor = [[UIColor orangeColor] CGColor];
    [self.view.layer addSublayer:canvasLayer];
    
    CAShapeLayer *ovalShapeLayer = [CAShapeLayer layer];
    ovalShapeLayer.path = [[self createBezierPath] CGPath];
    canvasLayer.mask = ovalShapeLayer;
    
    CALayer *coverLayer = [CALayer layer];
    coverLayer.frame = CGRectMake(0, 0 , 52, 94 );
    coverLayer.anchorPoint = CGPointMake(0, 0);
    coverLayer.position = CGPointMake(0, 0);
    coverLayer.backgroundColor = [[UIColor whiteColor] CGColor];
    [canvasLayer addSublayer:coverLayer];
    
    CABasicAnimation *animation = [CABasicAnimation animation];
    animation.keyPath = @"bounds.size.height";
    animation.fromValue = @(94);
    animation.toValue = @(0);
    animation.duration = 5;
    animation.repeatCount = HUGE;
    animation.removedOnCompletion = YES;
    
    [coverLayer addAnimation:animation forKey:nil];

```

贝塞尔生成的不规则图形，frame为{1，1，52，94}

```objc
- (UIBezierPath *)createBezierPath {
    // W:H = 70:120
    // oval frame {1,1,52,94}
    UIBezierPath* ovalPath = [UIBezierPath bezierPath];
    [ovalPath moveToPoint: CGPointMake(53, 30.53)];
    [ovalPath addCurveToPoint: CGPointMake(27, 95) controlPoint1: CGPointMake(53, 46.83) controlPoint2: CGPointMake(41.36, 95)];
    [ovalPath addCurveToPoint: CGPointMake(1, 30.53) controlPoint1: CGPointMake(12.64, 95) controlPoint2: CGPointMake(1, 46.83)];
    [ovalPath addCurveToPoint: CGPointMake(27, 1) controlPoint1: CGPointMake(1, 14.22) controlPoint2: CGPointMake(12.64, 1)];
    [ovalPath addCurveToPoint: CGPointMake(53, 30.53) controlPoint1: CGPointMake(41.36, 1) controlPoint2: CGPointMake(53, 14.22)];
    [ovalPath closePath];
    return ovalPath;
}
```

再来看看波涛汹涌的动画：

![easymaskanimating5.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating5.gif)

这是一个不断波动并上升的动画。动画主要由两部分组成：
- 时刻滚动的波浪
- 不断上升的水平面

时刻滚动的波浪：创建一个CAShapeLayer层，作为mask。使用CADisplayLink，每桢都重画shapeLayer的path属性，这样子就会产生波浪起伏的效果。

![easymaskanimating6.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating6.gif)

不断上升的水平面，是利用CABasicAnimation动画，修改masklayerd的position的结果。

![easymaskanimating7.gif](http://7xo30v.com1.z0.glb.clouddn.com/easymaskanimating7.gif)

再说下这里图层的包含关系，左面包含右面图层:

view.lyaer->bglayer->canvasLayer->waveLayer

view.layer是当前控制器的view的layer,
bglayer是黑色边框图层,
canvasLayer是背景图层,
waveLayer是遮罩图层.

```objc
#import "ViewController.h"

@interface ViewController ()

@property (nonatomic, strong) CADisplayLink *displayLink;
//背景图层
@property (nonatomic, strong) CALayer *canvasLayer;
//遮罩图层
@property (nonatomic, strong) CAShapeLayer *waveLayer;
//背景图层frame
@property (nonatomic) CGRect frame;
//遮罩图层frame
@property (nonatomic) CGRect shapeFrame;

@end

@implementation ViewController
//初始相位
static float phase = 0;
//相位偏移量
static float phaseShift = 0.25;
- (void)viewDidLoad {
    [super viewDidLoad];
    //shapePointY=200，设定mask其实位置（0，200）
    CGFloat shapePointY = 200;
    CGRect frame = CGRectMake(0, 0, 100, 200);
    CGRect shapeFrame = CGRectMake(0, shapePointY, 100, 200);
    self.frame = frame;
    self.shapeFrame = shapeFrame;
    
    //黑色边框
    CALayer *bglayer = [CALayer layer];
    bglayer.frame = CGRectMake(0, 20, 100, 200);
    bglayer.borderWidth = 1.0;
    bglayer.borderColor = [[UIColor blackColor] CGColor];
    [self.view.layer addSublayer:bglayer];
    
    //创建背景图层
    self.canvasLayer = [CALayer layer];
    self.canvasLayer.frame = frame;
    self.canvasLayer.backgroundColor = [UIColor orangeColor].CGColor;
    [bglayer addSublayer:self.canvasLayer];
    //创建遮罩图层
    self.waveLayer = [CAShapeLayer layer];
    self.waveLayer.frame = shapeFrame;
    //设定mask为waveLayer
    self.canvasLayer.mask = self.waveLayer;
    
    //开始动画
    [self startAnimating];
}

- (void)startAnimating {
    [self.displayLink invalidate];
    self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(update)];
    [self.displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
    
    //获得结束时的position.y值
    CGPoint position = self.waveLayer.position;
    position.y = position.y - self.shapeFrame.size.height;
    CABasicAnimation *animation = [CABasicAnimation animationWithKeyPath:@"position"];
    animation.fromValue = [NSValue valueWithCGPoint:self.waveLayer.position];
    animation.toValue = [NSValue valueWithCGPoint:position];
    animation.duration = 5.0;
    animation.repeatCount = HUGE_VALF;
    animation.removedOnCompletion = NO;
    [self.waveLayer addAnimation:animation forKey:nil];
    
}
//波浪滚动 phase相位每桢变化值：phaseShift
- (void)update {
    CGRect frame = self.frame;
    phase += phaseShift;
    UIGraphicsBeginImageContext(frame.size);
    UIBezierPath *wavePath = [UIBezierPath bezierPath];
    CGFloat endX = 0;
    for(CGFloat x = 0; x < frame.size.width ; x += 1) {
        endX=x;
        //正弦函数，求y值
        CGFloat y = 5 * sinf(2 * M_PI *(x / frame.size.width)  + phase) ;
        if (x==0) {
            [wavePath moveToPoint:CGPointMake(x, y)];
        }else {
            [wavePath addLineToPoint:CGPointMake(x, y)];
        }
    }
    CGFloat endY = CGRectGetHeight(frame);
    [wavePath addLineToPoint:CGPointMake(endX, endY)];
    [wavePath addLineToPoint:CGPointMake(0, endY)];
    //修改每桢的wavelayer.path
    self.waveLayer.path = [wavePath CGPath];
    UIGraphicsEndImageContext();
}
@end
```

最后，让波浪在特定容器中运动。即文章开始的动效。思路有两种：
- 直接添加一个背景图片到bglayer上
- 将bglayer设定为CAShapeLayer，path为想要图形的值，动画在CAShpaeLayer中显示。

最后**Demo**见[这里](https://github.com/EvoIos/AnimationOneDemo)


