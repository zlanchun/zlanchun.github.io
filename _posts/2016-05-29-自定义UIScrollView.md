---
layout: single
title: 自定义UIScrollView
date: 2016-05-29 01:36:30 +0800
tags: UIScrollView
---

## 0

学习路径：

- [Dynamic基础](https://www.raywenderlich.com/50197/uikit-dynamics-tutorial)
- [understanding-UIScrollView](http://oleb.net/blog/2014/04/understanding-uiscrollview/)
- [Dynamic实现UIScrollView](http://holko.pl/2014/07/06/inertia-bouncing-rubber-banding-uikit-dynamics/)
- [饿了么官方博客](http://eleme.io/mobilists/2016/03/15/用UIKit-Dynamics模仿UIScrollView/)


## 1 

**bounds**

UISCrollView继承于UIView，有bouncing、rubber-banding（橡皮筋）特效。在UIView上，新增了contentOffset属性，可以显示的内容大大增加。想一个容易一样，可以方进入很多东西。

是如何用UIView实现的呢？如果你还没有看上面链接的文章，强烈推荐你看完后，再回来。我们知道View有两个标定自身大小的属性`frame`和`bounds`。`frame`指定自身位于父视图的位置和大小。而`bounds`则是指显示相对于自身的位置和大小。而UIScrollView则是利用`bounds`属性来动态的显示其contentOffsize的内容。而contentOffset更改的则是其自身的`bounds`值。

[understanding-UIScrollView](http://oleb.net/blog/2014/04/understanding-uiscrollview/)这里说的很明白。

**rubber-banding**
橡皮筋效果，之前一直不明白什么是橡皮筋效果，谷歌翻译后，也还是有些懵。后来看具体代码实现，可以简单的表示为：`y = x * 0.5`。假如你运动了5米，而你得到的距离是2.5米，以此类推，你得到的永远小于你实际运动的距离。

这里有一个公式（d 为屏幕的宽度或高度，c一般为0.55，别问为为什么）：

```
f(x, d, c) = (x * d * c) / (d + c * x)
where,
x – distance from the edge
c – constant (UIScrollView uses 0.55)
d – dimension, either width or height
```

当然最开始的时候，不是这样的。最初的公式是：

y = 0.5 * x

简单吧，简单但是效果很好。这些都是twitter [Grant Paul大神](https://twitter.com/chpwn/status/285540192096497664)提出的。

下面我们来看看效果：

```objectivec
//OC 函数实现： 正规版本
static CGFloat rubberBandDistance(CGFloat offset, CGFloat dimension) {
    
    const CGFloat constant = 0.55f;
    CGFloat result = (constant * fabs(offset) * dimension) / (dimension + constant * fabs(offset));
    // The algorithm expects a positive offset, so we have to negate the result if the offset was negative.
    return offset < 0.0f ? -result : result;
}

//最初版本：
static CGFloat previousRubberBandDistance(CGFloat offset) {
    CGFloat result = fabs(offset) * 0.5;
    return offset < 0.0f ? -result : result;
}
```

如图：手指划过一个屏幕的距离后，两种函数处理后的结果对比:

![](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_Snip20160528_6.png)

![](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_Snip20160528_8.png)


**bouncing**

当视图达到边缘后，还会继续滚动一段距离，然后慢慢回到边界。bouncing就是说的就是这个效果。这里我们用Dynamic实现。

- `UIDynamicItemBehavior` 实现惯性效果。它接收一个速度参数，可以设定阻力值。
- `UIAttachmentBehavior`实现弹性效果，到达边界后，弹出一段距离后再回到边界位置。
- 我们需要自定义一个遵守`UIDynamicItem`协议的类，用作模型层，代替具体的UIView。

弹出一段距离在回到终点的效果是同时使用`UIAttachmentBehavior`和`UIDynamicItemBehavior`。其中：`UIAttachmentBehavior`设定Anchor为终点，`UIDynamicItemBehavior`设定一个相反的很大速度。然后在两者的作用下，就会出现回弹效果。见下面的红色块。


![](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_attachment.gif)

具体实现：

```objectivec
//红色块效果
    LCDynamicItem *dynamicItem =  [[LCDynamicItem alloc] init];
    dynamicItem.center = redView.frame.origin;
            
    UIDynamicItemBehavior *decelerationBehavior = [[UIDynamicItemBehavior alloc] initWithItems:@[dynamicItem]] ;
    [decelerationBehavior addLinearVelocity:CGPointMake(0, -1000) forItem:dynamicItem];
    decelerationBehavior.resistance = 2.0;
            
    __block BOOL onlyOnce = YES;
            
    decelerationBehavior.action = ^{
      CGRect frame = redView.frame;
      frame.origin = dynamicItem.center;
      if (frame.origin.y > 64) {
          redView.frame  = frame;
      }
      if (onlyOnce) {
          //add attachment
          UIAttachmentBehavior *springBehavior = [[UIAttachmentBehavior alloc] initWithItem:dynamicItem attachedToAnchor:CGPointMake(20, self.view.frame.size.height - 50)];
          springBehavior.frequency = 2.0;
          springBehavior.length = 0;
          springBehavior.damping = 1;
          [self.animator addBehavior:springBehavior];
                    
          onlyOnce = NO;
      }
    };
            
    [self.animator addBehavior:decelerationBehavior];

```

**应用**

用自定义UIScrollView + UITableView来实现效果的APP貌似不多。到目前为止，我就看到2款APP。一款是苹果自带的[系统天气]，另一款就是饿了么（商品购买界面）。然后，我有找了其他外卖商品类APP。和饿了么类似的还有美团外卖、百度外卖，不过美团外卖和百度外卖没有实现效果，饿了么实现了一部分效果。实现效果：系统天气 > 饿了么 > 美团和百度外卖。

嗯，效果就是下面这样的。向上滑动手指，tableView到达一定的位置后，才开始滚动；向下滑动手指，当tableView回到初始位置后，才开始滚动。饿了吗实现由个问题，就是在滑动到最底部的时候，再向上滑动会有非常黏的效果。

![](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_%E9%A5%BF%E4%BA%86%E4%B9%88%E5%B1%8F%E5%B9%95%E5%BD%95%E5%88%B6%E6%AD%A3%E5%B8%B8%E7%89%88%E6%9C%AC.gif)

自己写了一个简单的Demo，实现类似的效果。不过在实现过程中，简直痛苦，需要判断各种情况下的状态和位移。常常是修正了这个问题，另一个问题又出来了。

实现思路：

自定义UIScrollView，添加tableView，设定其`scrollEnabled=NO`。

分析tableView的滚动的情况：

- 初始状态向下滚动，直接bouncing效果就可以了。
- 初始窗台向上滚动，未滚动到顶部。此时，tableView不滚动，继续向上位移。
- 初始状态向上滚动，到到顶部。此时位移停止，tableView滚动。
- 接上，继续滚动，未到达tableView的底部，继续滚动。
- 接上，滚动到底部了。tableView不再滚动，而是开始位移动作。并添加attachment，然后慢慢回到终点。

难点在到达边界后，上下滚动的判断和处理。这里感觉异常复杂，需要消息计算状态和位移条件。有的时候还不一定满足。就像下图一样，到达边界后，有4种情况：

- 之前滚动速度方向向上滚动，之后向上滚动
- 之前滚动速度方向向上滚动，之后向下滚动
- 之前滚动速度方向向下滚动，之后向上滚动
- 之前滚动速度方向向下滚动，之后向下滚动

![草图](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_IMG_1994.JPG)                                                                                                           

具体来看看效果如何：

![](http://7xo30v.com1.z0.glb.clouddn.com/CustomScrollView_result.gif)

Demo在[这里](https://github.com/EvoIos/CustomScrollViewDemo.git)

