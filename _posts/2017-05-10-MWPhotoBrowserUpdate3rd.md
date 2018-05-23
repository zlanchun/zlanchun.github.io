---
layout: post
title: MWPhotoBrowser 更新其依赖的第三方库
date: 2017-05-10 16:58:58 +0800
tags: iOS
---

#### 前因

项目里面用到了环信的 `EaseUI` 库，而其中又包含了第三方库的引用。`MWPhotoBrowser` 就是其中的一个。而 `MWPhotoBrowser` 自身又包含了第三方的依赖，而且上次更新已经很早了。作者不维护了。

`MWPhotoBrowser` 依赖的第三方库：

- MBProgressHUD
	- 0.9 （最新：1.0.0）
- SDWebImage
	- 3.7 （最新：4.0.0）
- DACircularProgress
	- 2.3 （最新：2.3.1）

尤其是 `SDWebImage` 这么大众的的库，竟然没有更新。不能忍。于是就想着 `fork` 后，更新下依赖，自用。

本文的内容，大体上是这次更新中遇到的问题的总结。

#### 修改源文件 

fork 后，MWPhotoBrowser 会在自己的 github 项目下。

git clone 下来。

**修改 `MWPhotoBrowser.podspec` 文件**

主要修改三个地方：

- s.homepage （项目首页地址）
- s.source（项目源地址）
- s.dependency （项目依赖的第三方库，我们主要改动的是这里）

修改的内容如下 （可飘过）

```spec
Pod::Spec.new do |s|

  s.name = 'MWPhotoBrowser'
  s.version = '2.1.2'
  s.license = 'MIT'
  s.summary = 'A simple iOS photo and video browser with optional grid view, captions and selections.'
  s.description = <<-DESCRIPTION
                  MWPhotoBrowser can display one or more images or videos by providing either UIImage
                  objects, PHAsset objects, or URLs to library assets, web images/videos or local files.
                  The photo browser handles the downloading and caching of photos from the web seamlessly.
                  Photos can be zoomed and panned, and optional (customisable) captions can be displayed.
                  DESCRIPTION
  s.screenshots = [
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser1.png',
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser2.png',
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser3.png',
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser4.png',
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser5.png',
    'https://raw.github.com/mwaterfall/MWPhotoBrowser/master/Screenshots/MWPhotoBrowser6.png'
  ]

 //修改这里
  s.homepage = 'https://github.com/EvoIos/MWPhotoBrowser'
  s.author = { 'Michael Waterfall' => 'michaelwaterfall@gmail.com' }
  s.social_media_url = 'https://twitter.com/mwaterfall'
 //修改这里
  s.source = {
    :git => 'https://github.com/EvoIos/MWPhotoBrowser.git',
    :tag => '2.1.2'
  }
  s.platform = :ios, '7.0'
  s.source_files = 'Pod/Classes/**/*'
  s.resource_bundles = {
    'MWPhotoBrowser' => ['Pod/Assets/*.png']
  }
  s.requires_arc = true

  s.frameworks = 'ImageIO', 'QuartzCore', 'AssetsLibrary', 'MediaPlayer'
  s.weak_frameworks = 'Photos'
 
 //修改这里
  s.dependency 'MBProgressHUD', '~> 1.0'
  s.dependency 'DACircularProgress', '~> 2.3.1'

  # SDWebImage
  s.dependency 'SDWebImage', '~> 4.0.0'

end
```

**修改 MWPhoto.h**

因为新旧版本语法的不同，所以需要手动更改源项目的文件。目前，只有 SDWebImage 相关的需要修改。

头文件引入 `SDWebImage/SDWebImageDownloader.h`

修改 `downloadImageWithURL` 方法，因为旧版和新版的方法不一样了，所以这里需要更改下。

```objectivec
// Load from local file
- (void)_performLoadUnderlyingImageAndNotifyWithWebURL:(NSURL *)url {
    @try {
        [[SDWebImageDownloader sharedDownloader] downloadImageWithURL:url options:0 progress:^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            if (expectedSize > 0) {
                float progress = receivedSize / (float)expectedSize;
                NSDictionary* dict = [NSDictionary dictionaryWithObjectsAndKeys:
                                      [NSNumber numberWithFloat:progress], @"progress",
                                      self, @"photo", nil];
                [[NSNotificationCenter defaultCenter] postNotificationName:MWPHOTO_PROGRESS_NOTIFICATION object:dict];
            }
        } completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, BOOL finished) {
            if (error) {
                MWLog(@"SDWebImage failed to download image: %@", error);
            }
            _webImageOperation = nil;
            self.underlyingImage = image;
            dispatch_async(dispatch_get_main_queue(), ^{
                [self imageLoadingComplete];
            });

        }];
    } @catch (NSException *e) {
        MWLog(@"Photo from web: %@", e);
        _webImageOperation = nil;
        [self imageLoadingComplete];
    }
}
```

#### 提交

更改完后，提交到 GitHub 上。然后就可以用了。

#### 使用

使用的时候，有些不同。因为我们是 fork 别人的库，自己修改了，所以，不好传到 pods 上公开使用。但是我们还想要使用改过的库，怎么办？

集成的时候，标定远程项目库的地址和 commit 的版本。至于为什么要有 commit 版本，stackOverflow 上有个提问，见[这里](http://stackoverflow.com/questions/20936885/cocoapods-and-github-forks)。

最后 Podfile 里的文件内容如下：

```
pod "MWPhotoBrowser", :git => 'https://github.com/EvoIos/MWPhotoBrowser.git', :commit => 'de697e101195557ddca85661ebb266fd3f10776c'
```

**附录**

fork 的 [MWPhotoBrowser](https://github.com/EvoIos/MWPhotoBrowser) 项目地址