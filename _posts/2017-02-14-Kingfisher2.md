---
layout: post
title: Kingfisher 源码分析（二）：reserved Swift keyword 以及 单例
date: 2017-02-14 00:50:58 +0800
tags: 源码分析
---

### 引用文件：

- KingfisherManager.swift
- ImageView+Kingfisher.swift
- ImageCache.swift

### reserved swift keyword
ImageCache.swift 里有这么一段：

```swift
open class ImageCache { 
...
public static let `default` = ImageCache(name: "default")
...
}
```

当时蒙了，变量 `default` 在 backticks 里，是什么鬼？

查了下，因为 default 是 swift 里的关键字，所以不能直接用它作为变量或者常量。需要加反单引号才可以用。

使用起来也简单，把关键字加上双引号，就可以当做正常的变量或常量来使用了。

```swift
let `var` = "var"
print(`var`)
```

### static 修饰的静态变量

注意 default 是 static 修饰的变量。又想到，在类里面，都有哪些情况用到静态变量呢？

参考 [When to use static constant and variable in Swift](http://stackoverflow.com/questions/37701187/when-to-use-static-constant-and-variable-in-swift)，有2种情况：

- 共享信息
- 单例

```swift
//共享信息
class Animal {
static var nums = 0
init() {
Animal.nums += 1
}
}
let dog = Animal()
Animal.nums // 1
let cat = Animal()
Animal.nums // 2

//单例
class Singleton {
static let sharedInstance = Singleton()
private init() { }
func doSomething() { }
}
```

**类里面访问静态变量？**

2种方式，一种是类前缀，另一种是调用type(of: self) 函数。

```swift
public class KingfisherManager {
public static let someone = "老王！"
func log() {
//swift 3 里可用
print(type(of: self). someone)
//类前缀
print(KingfisherManager.someone)
}
}	
```

参考 [stackoverflow](http://stackoverflow.com/a/29954684)

**类外面使用静态变量？**

能吗？能的！就是单例的时候使用。

到这里马上就要进入今天的主题了，单例。

### 单例

先让我们来看看 Kingfisher 中使用单例的代码片段：

```swift
//ImageCache.swift
open class ImageCache { 
public static let `default` = ImageCache(name: "default")
public init(name: String, ...)
{
...
}
//KingfisherManager.swift
public class KingfisherManager {
public static let shared = KingfisherManager()
public var cache: ImageCache
//这里进行赋值了：cache = .default
convenience init() {
self.init(downloader: .default, cache: .default)
}
init(downloader: ImageDownloader, cache: ImageCache) {
self.downloader = downloader
self.cache = cache
}
}
```

其中 : `self.init(downloader: .default, cache: .default)`,这里的 .default 是咋回事？

command + 左键点进去后，就跳到了 ImageCache 类里 default 变量那了。

`.default` 是 `ImageCache.default` 的省略写法。

给 cache 赋值的是单例！这里迷惑了，后来找来找去，发现是带参数的单例。豁然开朗。

首先，这里面赋值动作是在 ImageCache 类外面，赋值对象是 chache，值是 ImageCache 的静态变量 default，也是  ImageCache 的单例。因此，上面说的在类的外面也可以使用静态变量。

其次，学到了如何创建一个单参数的单例，并如何去使用这样的单例。为什么要这么做？因为，想要获得带有附加信息的类的实例，而不是光秃秃的类实例。所以，这个时候，带有参数的单例就有了用武之地。

因此，带条件的单例可能想这样。代码片段：

```swift
open class ImageCache {
public static let `default` = ImageCache(name: "default")   
public init(name: String){
let cacheName = "com.onevcat.Kingfisher.ImageCache.\(name)"
print(cacheName)
}
}
var cache: ImageCache!
cache = .default

//输出
com.onevcat.Kingfisher.ImageCache.default
```
