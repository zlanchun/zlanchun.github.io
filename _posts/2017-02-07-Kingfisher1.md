---
layout: single
title: Kingfisher 源码分析（一）：命名空间 kf 实现
date: 2017-02-07 15:19:58 +0800
tags:
---

#### namespacing kf 实现

引用文件：

- Kingfisher.swift
- ImageView+Kingfisher.swift
- UIButton+Kingfisher.swift

实现：

``` swift
//Kingfisher.swift
public final class Kingfisher<Base> {
public let base: Base
public init(_ base: Base) {
self.base = base
}
}

/**
A type that has Kingfisher extensions.
*/
public protocol KingfisherCompatible {
associatedtype CompatibleType
var kf: CompatibleType { get }
}

public extension KingfisherCompatible {
public var kf: Kingfisher<Self> {
get { return Kingfisher(self) }
}
}

extension Image: KingfisherCompatible { }
#if !os(watchOS)
extension ImageView: KingfisherCompatible { }
extension Button: KingfisherCompatible { }

//ImageView+Kingfisher.swift
extension Kingfisher where Base: ImageView {
@discardableResult
public func setImage(with resource: Resource?,
placeholder: Image? = nil,
options: KingfisherOptionsInfo? = nil,
progressBlock: DownloadProgressBlock? = nil,
completionHandler: CompletionHandler? = nil) -> RetrieveImageTask
{
...
}
}

//UIButton+Kingfisher.swift
extension Kingfisher where Base: UIButton {
@discardableResult
public func setImage(with resource: Resource?,
for state: UIControlState,
placeholder: UIImage? = nil,
options: KingfisherOptionsInfo? = nil,
progressBlock: DownloadProgressBlock? = nil,
completionHandler: CompletionHandler? = nil) -> RetrieveImageTask
{
...
}
}
```

调用事例：

``` swift
let imageView = UIImageView()
imageView.kf.setImage(...)

let button = UIButton()
button.kf.setImage(...)
```

**分析**

类 `Kingfisher<Base>` 是一个范型类，类型是 Base 。

``` swift
public protocol KingfisherCompatible {
associatedtype CompatibleType
var kf: CompatibleType { get }
}
public extension KingfisherCompatible {
public var kf: Kingfisher<Self> {
get { return Kingfisher(self) }
}
}

extension ImageView: KingfisherCompatible { }
extension Button: KingfisherCompatible { }
```

定义协议 KingfisherCompatible，声明属性 kf，类型是范型 CompatibleType 。并要求遵守协议的一方，实现该属性的 get 方法。

在协议扩展中，协议自身实现了属性。这样就不必在每个遵守该协议的类里实现该属性了。参见苹果[文档](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Protocols.html#//apple_ref/doc/uid/TP40014097-CH25-ID267)：

>Protocols can be extended to provide method and property implementations to conforming types. This allows you to define behavior on protocols themselves, rather than in each type’s individual conformance or in a global function.

协议里的 kf 是一个 Kingfisher 类的实例，调用的方法是 Kingfisher 类的方法。根据类型的不同，调用不同类型里的方法。如：对应 UIImageView / UIButton 的 Kingfisher 里的setImage 方法。

综上，也就实现了命名空间 kf 。

如图：

![命名空间 kf](http://upload-images.jianshu.io/upload_images/1163763-d968964d8f35302b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们详细说说 KingfisherCompatible 。

`public var kf: Kingfisher<Self>` : Self 用在协议里面，代表的是遵守协议的 对象（类／结构体／枚举） 类型。如：`extension ImageView: KingfisherCompatible { }`，用在这里，Self 是 ImageView ； `extension Button: KingfisherCompatible { }`，用在这里， Self 是 Button 。

如果我们不用 `public extension KingfisherCompatible` ，也可以在遵守协议的时候，单独实现 kf 属性。

如单独实现 UIImageView 的协议扩展，像这样：

``` swift
extension UIImageView: KingfisherCompatible {
public var kf: Kingfisher<UIImageView> { return Kingfisher(self) }
}

extension Kingfisher where Base: UIImageView  {
public func setImage() { print("imageView setImage") }
}
```

最后，样例如下：

``` swift
public final class Kingfisher<Base> {
public let base: Base
public init(_ base: Base){
self.base = base
}
}
public protocol KingfisherCompatible {
associatedtype CompatibleType
var kf: CompatibleType { get }
}
extension UIImageView: KingfisherCompatible {
public var kf: Kingfisher<UIImageView> { return Kingfisher(self) }
}
extension Kingfisher where Base: UIImageView  {
public func setImage() { print("imageView setImage") }
}
extension UIButton: KingfisherCompatible {
public var kf: Kingfisher<UIButton> { return Kingfisher(self) }
}
extension Kingfisher where Base: UIButton {
public func setImage() { print("button setImage") }
}

let imageView = UIImageView()
imageView.kf.setImage()
let button = UIButton()
button.kf.setImage()

//输出
imageView setImage
button setImage
```

