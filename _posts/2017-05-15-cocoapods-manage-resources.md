---
layout: post
title:  CocoaPods 管理图片资源和字体库
date: 2017-05-15
tags: CocoaPods
---

### 前言

当 iOS 程序到一定程度后，种种原因，想要解耦，组件化或者模块化。这里绕不开的就是基础模块独立，其他模块依赖基础模块。

有的时候，就需要把资源单独拿出来。可能会有各种方法。这里将要说的是如何用 cocoapods 管理图片资源和字体库。

大体上的流程是这样的：

- 用 cocoapods 创建私库。
- 提取项目里的资源文件（这里指图片资源和字体库）到 cocoapods 中，配置 spec。
- 上传库到公司内部服务器。
- 项目里 pod 该库

### cocoapods 创建私库

创建私库过程和创建库过程一致，只是在 spec 配置文件里，将连接地址填写为私有服务器地址。

创建 pods 库见[官方文档](https://guides.cocoapods.org/making/making-a-cocoapod.html)

pods 库创建初始化：`pod lib create ZLCResources`.  根据提示输入内容。

![Alt text](https://raw.githubusercontent.com/zlanchun/blogImages/master/CocoaPods_manage_resources/manage_resource.png)

创建成功后，会自动弹出工程界面。工程如下：

![Alt text](https://raw.githubusercontent.com/zlanchun/blogImages/master/CocoaPods_manage_resources/manage_resource1.png)

### 配置  ZLCResources.podspec

因为要用 pods 管理图片资源和字体库，所以我们要告诉 pods ，资源文件在哪。

又因为我们要创建私库，链接地址并不在 github 上，所以需要修改指定的链接地址，使其指向我们的服务器的地址。

同时，我们想要支持 iOS 7，所以我们要指定版本。

具体的如下所示：

```ruby

#链接地址指向我们的服务器，这里假设我的地址在coding.net上
s.homepage         = 'https://git.coding.net/zlanchun/ZLCResources.git'
s.source           = { :git => 'https://git.coding.net/zlanchun/ZLCResources.git', :tag => s.version.to_s }

#指定支持的版本，这里支持 iOS 7.0 及以上的版本
s.ios.deployment_target = '7.0'

#我在ZLCResources/ZLCResources下面创建了2个新文件夹：Iconfonts 和 Images。一个放图片，一个放字体库。然后把bundle地址指向这2个目录。
  s.resource_bundles = {
     'ZLCResources' => ['ZLCResources/Images/*.*','ZLCResources/Iconfonts/*.*']
  }
```

最后修改完后的 podspec 文件内容：

```ruby
#
# Be sure to run `pod lib lint ZLCResources.podspec' to ensure this is a
# valid spec before submitting.
#
# Any lines starting with a # are optional, but their use is encouraged
# To learn more about a Podspec see http://guides.cocoapods.org/syntax/podspec.html
#

Pod::Spec.new do |s|
  s.name             = 'ZLCResources'
  s.version          = '0.1.0'
  s.summary          = 'ZLCResources for images and iconfonts'

# This description is used to generate tags and improve search results.
#   * Think: What does it do? Why did you write it? What is the focus?
#   * Try to keep it short, snappy and to the point.
#   * Write the description between the DESC delimiters below.
#   * Finally, don't worry about the indent, CocoaPods strips it!

  s.description      = <<-DESC
TODO: Add long description of the pod here.
                       DESC

  s.homepage         = 'https://git.coding.net/zlanchun/ZLCResources.git'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'zlanchun' => 'zlanchun@gmail.com' }
  s.source           = { :git => 'https://git.coding.net/zlanchun/ZLCResources.git', :tag => s.version.to_s }
  # s.social_media_url = 'https://twitter.com/<TWITTER_USERNAME>'

  s.ios.deployment_target = '7.0'

  s.source_files = 'ZLCResources/Classes/**/*'
  
  s.resource_bundles = {
     'ZLCResources' => ['ZLCResources/Images/*.*','ZLCResources/Iconfonts/*.*']
  }

  s.public_header_files = 'ZLCResources/Classes/**/*.h'
  # s.frameworks = 'UIKit', 'MapKit'
  # s.dependency 'AFNetworking', '~> 2.3'
end

```

然后，在 ZLCResources 里创建2个分类，用于提供加载 pods 里的图片和字体库用。

加载图片的分类：

```objc
+ (UIImage *)zlc_imageNamed:(NSString *)name ofType:(nullable NSString *)type {
    NSString *mainBundlePath = [NSBundle mainBundle].bundlePath;
    NSString *bundlePath = [NSString stringWithFormat:@"%@/%@",mainBundlePath,@"ZhuanResourcesBundle.bundle"];
    NSBundle *bundle = [NSBundle bundleWithPath:bundlePath];
    if (bundle == nil) {
        bundlePath = [NSString stringWithFormat:@"%@/%@",mainBundlePath,@"Frameworks/ZhuanResources.framework/ZhuanResourcesBundle.bundle"];
        bundle = [NSBundle bundleWithPath:bundlePath];
    }
    if ([UIImage respondsToSelector:@selector(imageNamed:inBundle:compatibleWithTraitCollection:)]) {
        return [UIImage imageNamed:name inBundle:bundle compatibleWithTraitCollection:nil];
    } else {
        return [UIImage imageWithContentsOfFile:[bundle pathForResource:name ofType:type]];
    }
}
```

加载字体库的分类：

```objc
#import "UIFont+ZLCFontLoader.h"
#import <CoreText/CTFontManager.h>

implementation UIFont (ZLCFontLoader)
+ (UIFont *)iconFontOfSize:(CGFloat)size
{
    NSString *fontName = @"unsplashFont";
    UIFont *font = [UIFont fontWithName:fontName size:size];
    if (!font) {
        [[self class] dynamicallyLoadFontNamed:fontName];
        font = [UIFont fontWithName:fontName size:size];
        if (!font) font = [UIFont systemFontOfSize:size];
    }
    return font;
}

+ (void)dynamicallyLoadFontNamed:(NSString *)name
{
    NSString *fontfileName = @"iconfont.ttf";
   
    NSString *mainBundlePath = [NSBundle mainBundle].bundlePath;
    NSString *bundlePath = [NSString stringWithFormat:@"%@/%@",mainBundlePath,@"ZLCResources.bundle"];
    NSBundle *bundle = [NSBundle bundleWithPath:bundlePath];
    if (bundle == nil) {
        bundlePath = [NSString stringWithFormat:@"%@/%@",mainBundlePath,@"Frameworks/ZLCResources.framework/ZLCResources.bundle"];
        bundle = [NSBundle bundleWithPath:bundlePath];
    }
    NSString *resourcePath = [NSString stringWithFormat:@"%@/%@",bundle.bundlePath,fontfileName];
    
    NSURL *url = [NSURL fileURLWithPath:resourcePath];
    
    NSData *fontData = [NSData dataWithContentsOfURL:url];
    if (fontData) {
        CFErrorRef error;
        CGDataProviderRef provider = CGDataProviderCreateWithCFData((CFDataRef)fontData);
        CGFontRef font = CGFontCreateWithDataProvider(provider);
        if (! CTFontManagerRegisterGraphicsFont(font, &error)) {
            CFStringRef errorDescription = CFErrorCopyDescription(error);
            NSLog(@"Failed to load font: %@", errorDescription);
            CFRelease(errorDescription);
        }
        CFRelease(font);
        CFRelease(provider);
    }
}
@end

```

使用：

```objc

//ZLCViewController 里，引入头文件
#import "ZLCViewController.h"
#import <ZLCResources/UIImage+ZLCImageLoader.h>
#import <ZLCResources/UIFont+ZLCFontLoader.h>

//viewdidload 里调用字体库和图片资源
self.iconFontLabel.font = [UIFont iconFontOfSize:24];
self.iconFontLabel.text = @"Download \U0000e607";
    
self.imageView.image = [UIImage zlc_imageNamed:@"wallpaper83.5" ofType:nil];

```

**注意**

字体库的名字，不是文件的名字，而是Font Family。这里容易混。

![iconfont](https://raw.githubusercontent.com/zlanchun/blogImages/master/CocoaPods_manage_resources/manage_resource2.png)

Podfile 里，`use_frameworks!` 表示是否生成 framework。如果没有 `use_frameworks!`，则表示bundle直接生成在工程当前目录下，和mainBundle 是一个目录地址。如果有，则表示生成在当前 framwork 的目录下。

看图：

![](https://raw.githubusercontent.com/zlanchun/blogImages/master/CocoaPods_manage_resources/manage_resource3.png)

![](https://raw.githubusercontent.com/zlanchun/blogImages/master/CocoaPods_manage_resources/manage_resource4.png)


**最后**

都修改完后：

- pod lib lint ，检验spec是否合乎规则
- pod install ，重新安装修改该后的本地库


然后，验证通过后，将工程上传到服务器上。

新建一个工程，在 Podflie 里集成新创建的库，并指定git地址，测试效果。

```ruby
pod "ZLCResources", :git => 'https://git.coding.net/zlanchun/ZLCResources.git'
```

#### 参考文章

- [Sharing assets across iOS projects with CocoaPods, Resource Bundle, and dynamically loaded fonts](http://www.mokacoding.com/blog/sharing-assets-with-cocoapods-resource-bundle-and-dynamically-loaded-fonts/)

#### Demo 

[链接](https://git.coding.net/zlanchun/ZLCResources.git)