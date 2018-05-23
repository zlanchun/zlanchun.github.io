---
layout: post
title: CocoaPods 集成私有库
date: 2017-05-28 
tags: Cocoapods
---

### 前言

Private Spec Repo, 我把它叫做私有库。有更好的叫法，请一定告知我，不胜感谢。

CocoaPods version: 1.2.1.

**什么时候用**

>CocoaPods is a great tool not only for adding open source code to your project, but also for sharing components across projects. You can use a private Spec Repo to do this.

[官方](https://guides.cocoapods.org/making/private-cocoapods.html)说法，想要要在不同的项目中分享组件，这时你该创建私有库了。

**怎么做**

官方文档写的真是简洁，本以为实际操作也会如此! 是我太年轻了。

按照文档上说就三步：

- Create a Private Spec Repo
- Add your Private Repo to your CocoaPods installation
- Add your Podspec to your repo

创建一个 repo, 添加 repo 到你的 CocoaPods 里, 然后推送 podspec 到 repo 里。

当然，这只是概述。会有各种细节在等着你，让你无所事事，不得寸进。

会在下面来说下最近一段时间我的历程和爬过的坑。

**私有库和公有库的异同**

Podfile 里继承 AFNetworking: `pod 'AFNetworking'`

当我们执行 pod install 后，CocoaPods 做了什么？

CocoaPods 会去 ~/.cocoapods/repos/master/Specs/ 目录下寻找是否有 AFNetworking 的 podspec？如果没有，则退出，并给出提示无法找到；如果有，则按照 podspec 中记录的第三方库的 GitHub 地址去下载并继承到工程里。

其中，master/Specs 相当于一个索引文件，里面记录的都是各个第三方库的url，然后当我们需要安装的时候，再去通过这些 url 去下载具体的第三方库。

而 master/Specs 是一个 CocoaPods 的 [github repo](https://github.com/CocoaPods/Specs)，专门用来负责存储第三方库 url 的索引文件。

我们平时发布的库的操作有2个步骤。一个是创建库并封装功能上去，然后将封装好的库的url告知CocoaPods。这样，其他人就能获得我们发布的三方库了。

所以，我们明晰了想要一个第三方库生效的2个条件：

- 必定有一个 repo 存储在服务器上，他人可以获得.
- 有一个中心化的 Specs repo 用来存储第三方库的索引文件 podspec.

针对上面，公有库和私有库的区别就很明显了。


| Item      |    公有库 | 私有库  |
| :-------- | :--------:| :--: |
| 库 repo  | github - 公开项目 |  随意（想放哪就放哪）   |
| 索引中心 Specs    | github - CocoaPods/Specs |  Specs (想放哪就放哪)  |

私库库 repo 的位置不定，可以在任何位置，只要能够索引到就行。有的时候也可以不是 repo，而是可下载的文件。私有/Specs 代替 CocoaPods/Specs 用来作为私有库的中心.

###  创建私有库具体过程

下面将要交大家如何一步步去创建私有库。已 gitlab 为事例服务器。当然你也可以在别的git服务器上创建，如：coding.net，或者公司内部的git服务器。

#### 1.gitlab 上创建 lib repo 和  Specs repo （Specs 是用来管理私有 lib repo 的）

私有的 lib repo :

![cocoapodsDemo1.png](https://raw.githubusercontent.com/zlanchun/blogImages/master/Cocoapos%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%93/Cocoapods%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%931.png)

Specs repo :

![specs repo](https://raw.githubusercontent.com/zlanchun/blogImages/master/Cocoapos%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%93/Cocoapods%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%932.png)

这里已经有内容了，其实就是建立一个名字为 Specs 的空 repo.

#### 2.创建 pod lib repo

    $ pod lib create PrivateLibraryRepo
    
**添加功能实现文件**

在 PrivateLibraryRepo/Classes 下添加功能类 Foo. 目录结构如下：

```
├── PrivateLibraryRepo
│   ├── Assets
│   └── Classes
│       ├── Foo.h
│       ├── Foo.m
│       └── ReplaceMe.m
```

进入到 Example 木下，执行：`pod install` 安装刚刚我们添加的帮助类。

Foo 内容如下：

```
#import "Foo.h"

@implementation Foo
- (void)foo { NSLog(@"Foo say hello!"); }
@end
```

在 PrivateLibraryRepo.xcworkspace 的 PZViewController 里，引入Foo并调用。

```objectivec
- (void)viewDidLoad
{
    [super viewDidLoad];
	// Do any additional setup after loading the view, typically from a nib.
    Foo *f = [Foo new];
    [f foo];
}
```

输出结果：Foo say hello!

说明Foo库函数没有问题。可以进行下一步了。

**修改 podspec **

找到 PrivateLibraryRepo.podspec 并修改。podspec 语法参见 [Podspec Syntax Reference](https://guides.cocoapods.org/syntax/podspec.html).

因为是私有库，所以我们要指定私有库的url地址，有可能在任何地方。然后，可能根据个人心意去修改版本号。有的时候还要修改源文件目录或者添加资源文件夹等等。根据个人情况更改。

下面是改好的，有备注。

```ruby
Pod::Spec.new do |s|
  s.name             = 'PrivateLibraryRepo'
  #版本号，默认从0.1.0开始
  s.version          = '0.1.0'
  s.summary          = 'A short description of PrivateLibraryRepo.'

  s.description      = <<-DESC
TODO: Add long description of the pod here.
                       DESC
  s.homepage         = 'https://gitlab.com/zlanchun/PrivateLibraryRepo'
  s.license          = { :type => 'MIT', :file => 'LICENSE' }
  s.author           = { 'feimengchang' => 'zlanchun@gmail.com' }
  #source 地址，当前私有库的git地址                                              
  s.source           = { 
                        :git => 'https://zlanchun@gitlab.com/zlanchun/PrivateLibraryRepo.git', 
                        :tag => s.version.to_s 
                       }

  s.ios.deployment_target = '7.0'
  s.platform     = :ios, '7.0'
  s.source_files = 'PrivateLibraryRepo/Classes/**/*'
end
```

#### 3.验证 pod lib repo

**lib repo 验证**

在目录 PrivateLibraryRepo 根下执行命令。

    $ pod lib lint --allow-warnings
    
`--allow-warnings` 的意思是忽略所有警告，并通过校验。

pod lib lint 用于检测本地 lib repo 的有效性。如果无效，则会有报错 error 出现。有过有效，就会给出提示信息，如：PrivateLibraryRepo passed validation.

**spec 验证**

索引文件 podspec 验证。注意这个时候，因为我们的库文件都在本地，而podspec 验证是需要去和服务器通信的，所以此步骤会出错是正常的。

    $ pod spec lint 
    
报错：

fatal: Remote branch 0.1.0 not found in upstream origin

留待我们后面解决。

#### 4.上传 lib repo 到服务器

    $ git remote add origin git@gitlab.com:zlanchun/PrivateLibraryRepo.git
    $ git add .
    $ git commit -m "Initial commit"
    $ git push -u origin master
    //tag 值要和podspec中的version一致
    $ git tag 0.1.0
    //推送tag到服务器上
    $ git push --tags

这里除了tag 指定版本外，还可以新建一个分支，分支名字和version值一致，同样能起到作用。如果什么都没有，则会报错。报错也是这样的：

fatal: Remote branch 0.1.0 not found in upstream origin

#### 5.podspec 再次验证

  $ pod spec lint --allow-warnings
  
这一次会验证通过：pod spec lint --allow-warnings。

#### 6.发布 podspec

**指定管理 lib repo 的 Specs repo 的 url**

    $ pod repo add NAME URL [BRANCH]

如：

    $ pod repo add PrivateLibraryRepo https://zlanchun@gitlab.com/zlanchun/Specs.git

**推送 podspec 到 Specs repo**

    $ pod repo push REPO [NAME.podspec] --sources=xxx,yyy
    
注意：

`--sources` 用来指定 Specs 的 repo 地址。默认是推送的github上。因为这里我们是私库，用的是gitlab，所以需要设定这个值。

如：

    $ pod repo push PrivateLibraryRepo PrivateLibraryRepo.podspec --sources=https://zlanchun@gitlab.com/zlanchun/Specs.git --allow-warnings

**验证**

新建一个工程，在其Podfile 里，加入如下内容：

```ruby

source 'https://zlanchun@gitlab.com/zlanchun/Specs.git'
source 'https://github.com/CocoaPods/Specs.git'

target 'Test' do
  # Uncomment the next line if you're using Swift or would like to use dynamic frameworks
  # use_frameworks!
pod 'PrivateLibraryRepo'

end

```

然后，执行 pod install:

![cocoapodsDemo2.png](https://raw.githubusercontent.com/zlanchun/blogImages/master/Cocoapos%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%93/Cocoapods%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%933.png)

注意：

Podfile 里头部的2个source。pod 会按照上下顺序去搜索，如果搜索到lib则进行下一步动作，如果没找到，则切换到下一个source url继续搜索。

这里我们指定先搜索我们自己的 Specs repo，然后再去搜索 CocoaPods的 Specs. 

### 私有库依赖私有库

当我们有2个私有库，其中一个依赖另一个的时候，就会有各种问题出来阻挠你，说多了都是泪。

#### Podspec 语法：dependency

podspec 的 `dependency` 用来指定依赖关系。

eg: spec.dependency 'AFNetworking', '~> 1.0'

> ~> 1.0.1 is equivalent to >= 1.0.1 combined with < 1.1. Similarly, ~> 1.0 will match 1.0, 1.0.1, 1.1, but will not upgrade to 2.0.

#### Command-line 语法：`pod lib lint --sources`

>The sources from which to pull dependent pods (defaults to https://github.com/CocoaPods/Specs.git). Multiple sources must be comma-delimited..

sources 用于指定 pods 地址，默认指向 master ,如果有多个源用逗号分隔。下面会用到。

#### 具体实现

我们新建一个私库 PrivateLibraryDependency：
  
    $ pod lib create PrivateLibraryDependency

PrivateLibraryDependency 依赖2个库，一个私库PrivateLibraryRepo，一个SDWebImage。

更改 PrivateLibraryDependency.podspec

- 修改 repo 地址
- 添加依赖
  - s.dependency 'SDWebImage', '~> 4.0.0'
  - s.dependency 'PrivateLibraryRepo', '~> 0.1.0'
  
在 Example 目录下的 Podfile 添加source ,然后执行 pod install。

```ruby
#Podfile
source 'https://zlanchun@gitlab.com/zlanchun/Specs.git'
source 'https://github.com/CocoaPods/Specs.git'
```


![cocoapodsDemo3.png](https://raw.githubusercontent.com/zlanchun/blogImages/master/Cocoapos%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%93/Cocoapods%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%934.png)

根目录下验证lib，这里要添加`--sources`指定私库Specs地址，因为我们的依赖里有私库，所以要指定。否则会报错。

    $ pod lib lint --sources=https://zlanchun@gitlab.com/zlanchun/Specs.git,https://github.com/CocoaPods/Specs.git
  
上传库到服务器，打tag。然后验证spec，同样需要指定`--sources`

    $ pod spec lint --sources=https://zlanchun@gitlab.com/zlanchun/Specs.git,https://github.com/CocoaPods/Specs.git

发布到私Specs repo里，注意这里的source值是 private,master url.因为，我们依赖的既有私有库也有公有库。

    $ pod repo add PrivateLibraryDependency https://zlanchun@gitlab.com/zlanchun/Specs.git
    $ pod repo push PrivateLibraryDependency PrivateLibraryDependency.podspec --sources=https://zlanchun@gitlab.com/zlanchun/Specs.git,https://github.com/CocoaPods/Specs.git
    
![cocoapodsDemo4.png](https://raw.githubusercontent.com/zlanchun/blogImages/master/Cocoapos%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%93/Cocoapods%E9%9B%86%E6%88%90%E7%A7%81%E6%9C%89%E5%BA%935.png)

 ### subspec 依赖 subspec
  
  **subspec** 用来指定子模块的。
  
  在 PrivateLibraryDependency 目录下新建 ModuleA 和 ModuleB 目录。然后创建 ModuleA.{h,m} 和 ModuleB.{h,m}。
  
  目录结构：
  
  ```
├── PrivateLibraryDependency
│   ├── Assets
│   ├── Classes
│   │   └── ReplaceMe.m
│   ├── ModuleA
│   │   ├── ModuleB.h
│   │   └── ModuleB.m
│   └── ModuleB
│       ├── ModuleB.h
│       └── ModuleB.m
  ```
  
  这里，我们假定 子模块 ModuleA 依赖 ModuleB , podspec 如下：
  
  ```ruby
  
  s.subspec 'ModuleA' do |a|
    a.source_files = 'PrivateLibraryDependency/ModuleA/**/*.{h,m}'
    a.dependency 'PrivateLibraryDependency/ModuleB'
  end

  s.subspec 'ModuleB' do |b|
    b.source_files = 'PrivateLibraryDependency/ModuleB/**/*.{h,m}'
  end
  ```
  
  注意：子模块依赖的时候，依赖里一定要加上当前库前缀。如：a.dependency 'PrivateLibraryDependency/ModuleB'

参考 [PrivateLibraryDependency](https://zlanchun@gitlab.com/zlanchun/PrivateLibraryDependency.git) 我已经把 ModuleA 和 ModuleB 加进去了。

###  集成静态库和动态库（.a 和 framework）
  
####  集成 framework 到工程里
  
  集成第三方的 framework (vendored_frameworks)：
  
  framework 文件要单独集成到一个repo里，同时这个repo里不能包含别的代码文件。
  
  这里以支付宝SDK为例，提取支付宝SDK的 AlipaySDK.bundle 和 AlipaySDK.framework 放到单独的 repo 里。
  
  podspec 相应内容如下：
  
  ```ruby
  s.resources = "**/*.bundle"
  s.vendored_frameworks = "**/*.framework"
  ```
  
  整个 repo 目录结构：
  
  ```
.
├── AlipaySDK.bundle
├── AlipaySDK.framework
├── AlipaySDKIniOS.podspec
├── LICENSE
└── README.md
  ```
  
  集成系统自带的framework (frameworks / libraries):
  
  ```ruby
  
  # 系统的
  s.frameworks = 'SystemConfiguration', 'CoreTelephony', 'QuartzCore', 'CoreText'  , 'CoreGraphics', 'UIKit', 'Foundation', 'CFNetwork', 'CoreMotion'
  s.libraries = 'c++', 'z'
  ```
  
  注意：libc++.tbd 集成的时候，要去掉 lib 前缀。
  
  参考[这里 AlipaySDKIniOS](https://github.com/EvoIos/AlipaySDKIniOS)
  
  #### 集成 .a 库到工程里
  
  .a 集成的时候，稍微麻烦些！需要同时修改 podspec 和 Podfile 文件。
  
  以集成 openssl 为例，podspec 中需要更改如下内容：
  
  ```ruby
  s.subspec 'OpenSSL' do |openssl|
    openssl.source_files = 'AlipayWrapper/Openssl/**/*.h'
    openssl.public_header_files = 'AlipayWrapper/Openssl/**/*.h'
    openssl.ios.preserve_paths      = 'AlipayWrapper/StaticLibrary/libcrypto.a', 'AlipayWrapper/StaticLibrary/libssl.a'
    //指定 .a 文件为 vendored_libraries
    openssl.ios.vendored_libraries  = 'AlipayWrapper/StaticLibrary/libcrypto.a', 'AlipayWrapper/StaticLibrary/libssl.a'
    //需要指定为库名字
    openssl.libraries = 'ssl', 'crypto'
    //需要配置 HEADER_SEARCH_PATHS
    openssl.xcconfig = { 'HEADER_SEARCH_PATHS' => "${PODS_ROOT}/#{s.name}/AlipayWrapper/Openssl/**" }
  end
  ```
  
  Podfile 里需要添加如下内容，否则会报错：
  
  ```ruby
  pre_install do |installer|
    # workaround for https://github.com/CocoaPods/CocoaPods/issues/3289
    def installer.verify_no_static_framework_transitive_dependencies; end
end
  ```
  
  参考[AlipayWrapper](https://github.com/EvoIos/AlipayWrapper)，我把 AlipaySDK Demo 的帮助类集成到 pods 了，这样就省去了直接导入工程添加依赖库的步骤了。
  
   ### 各种错误集合
  
  #### Generated duplicate UUIDs
  
  ```
  [!] [Xcodeproj] Generated duplicate UUIDs:

PBXBuildFile -- /targets/buildConfigurationList:buildConfigurations:baseConfigurationReference:|,buildSettings:|,display
  ```
  
  解决办法：
  
  这时一个无害的报错，可以在 Podfile 头部加入如下内容修复：
  
  ```ruby
  install! 'cocoapods', :deterministic_uuids => false
  ```
  
  #### 2. <PBXResourcesBuildPhase UUID=`2E1007321C278F4300BCFC15`>` attempted to initialize an object with an unknown UUID

```
[!] `<PBXResourcesBuildPhase UUID=`2E1007321C278F4300BCFC15`>` attempted to initialize an object with an unknown UUID. `70CED53B1EC996BB008DF9D5` for attribute: `files`. This can be the result of a merge and  the unknown UUID is being discarded.
```
  
解决办法：

General -> Linked Frameworks and Libraries 删除 libPods-xxx.a

然后，重新执行 pod install

#### 3.Unable to find a specification for depended upon by


```
[!] Unable to find a specification for `TESTUtilitis (~> 1.0.0)` depended upon by `TESTVendors`
```

这个问题是更新库之后，没有更新本地的 repo，所以执行：

    $ pod repo update TESTUtilitis
    
#### 4.target has transitive dependencies that include static binaries

```
[!] The 'Pods-AlipayWrapper_Example' target has transitive dependencies that include static binaries: (/Users/z/Desktop/github/Alipay/AlipaySDK/AlipayWrapper/Example/Pods/AlipaySDKIniOS/AlipaySDK.framework)
```

这个问题是当我们要用 cocoapods 集成 xxx.a 文件的时候出现的。需要改2个地方：podspec 和 Podfile.

podspec

```ruby
 s.dependency 'AlipaySDKIniOS', '~> 15.2.0'
//添加 target_xconfig
  s.pod_target_xcconfig = {
    'FRAMEWORK_SEARCH_PATHS' => '$(inherited) $(PODS_ROOT)/AlipaySDKIniOS',
    'OTHER_LDFLAGS'          => '$(inherited) -undefined dynamic_lookup'
  }
```

Podfile

```ruby
#在最后面添加如下内容：
pre_install do |installer|
    # workaround for https://github.com/CocoaPods/CocoaPods/issues/3289
    def installer.verify_no_static_framework_transitive_dependencies; end
end
```

详情看这里：https://github.com/CocoaPods/CocoaPods/issues/3289

#### 5. Use the `$(inherited)` flag

```
[!] The `TEST [Debug]` target overrides the `FRAMEWORK_SEARCH_PATHS` build setting defined in `Pods/Target Support Files/Pods-TEST/Pods-TEST.debug.xcconfig'. This can lead to problems with the CocoaPods installation
    - Use the `$(inherited)` flag, or
    - Remove the build settings from the target.
```

解决：

>Go into the Build Settings for "Framework Search Paths" and change the value for your target to be "$(inherited)"

[参考](https://stackoverflow.com/questions/40599454/use-the-inherited-flag-or-remove-the-build-settings-from-the-target-c)

#### 6. [!] Unable to find a pod with name, author, summary, or description matching `AlipaySDKIniOS`

问题：trunk 成功后，无法搜索到。同时已经 pod repo udpate 到最新了，还是无法搜索到。

原因是：

~/Library/Caches/CocoaPods/search_index.json pod setup生成的索引文件没有更新。

解决：

删除后，从新 search 就好了。