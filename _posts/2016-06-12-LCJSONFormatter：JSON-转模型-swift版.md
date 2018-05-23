---
layout: post
title: LCJSONFormatter：JSON 转模型 swift版
date: 2016-06-12 19:59:35 +0800
tags: JSONFormatter
---

## [LCJSONFormatter](https://github.com/EvoIos/LCJSONFormatter)

![](http://7xo30v.com1.z0.glb.clouddn.com/LCJSONFormatter-LCJSONFormatterGif.gif)

这是一个纯粹的swift版本的JSON转模型工具。就像MJExtension一样，在swift中，我们用Argo.

样例JSON1，以字典开始：

```json
{
    "id": 1, 
    "name": "JohnSnow", 
    "dic": [
        {
            "key": "hello world", 
            "value": "where are you"
        }, 
        {
            "key": "hello world", 
            "value": "where are you"
        }
    ], 
    "commentsInfo": {
        "id": 100, 
        "value": "good man"
    }, 
    "price": 4.39, 
    "woman": false, 
    "optionalString": null, 
    "optionalArray": [ ]
}
```

样例JSON2，以数组开始：

```json
[
    {
        "id": 1, 
        "name": "JohnSnow", 
        "dic": [
            {
                "key": "hello world", 
                "value": "where are you"
            }, 
            {
                "key": "hello world", 
                "value": "where are you"
            }
        ]
    }, 
    {
        "id": 1, 
        "name": "JohnSnow", 
        "dic": [
            {
                "key": "hello world", 
                "value": "where are you"
            }, 
            {
                "key": "hello world", 
                "value": "where are you"
            }
        ]
    }
]
```
##使用

Alcatraz中搜索LCJSONFormatter，重启Xcode.

在swift文件中，呼出LCJSONFormatter窗口。将上面的JSON粘贴进去。一路回车就行了。

最后，转换后的文件如下：

```swift
//对于样例1，以字典开始
import UIKit

import Argo
import Curry

struct LCJSONFormatterDemo: Decodable { 
 
    let optionalString: String?
    let id: Int
    let price: Double
    let optionalArray: [String]?
    let dic: [Dic]
    let commentsInfo: Commentsinfo
    let woman: Bool
    let name: String 

    static func decode(json: JSON) -> Decoded<LCJSONFormatterDemo> {
          return curry(self.init)
        <^> json <|? "optionalString"
        <*> json <| "id"
        <*> json <| "price"
        <*> json <||? ["optionalArray"]
        <*> json <|| ["dic"]
        <*> json <| "commentsInfo"
        <*> json <| "woman"
        <*> json <| "name"
     }
}
 

struct Dic: Decodable { 
 
    let key: String
    let value: String 

    static func decode(json: JSON) -> Decoded<Dic> {
          return curry(self.init)
        <^> json <| "key"
        <*> json <| "value"
     }
}
 

struct Commentsinfo: Decodable { 
 
    let id: Int
    let value: String 

    static func decode(json: JSON) -> Decoded<Commentsinfo> {
          return curry(self.init)
        <^> json <| "id"
        <*> json <| "value"
     }
}

```

```swift
//对于样例2，以字典开始
import UIKit

import Argo
import Curry

struct LCJSONFormatterDemo: Decodable { 
 
    let lcArray: [Lcarray] 

    static func decode(json: JSON) -> Decoded<LCJSONFormatterDemo> {
          return curry(self.init)
        <^> json <|| ["lcArray"]
     }
}
 

struct Lcarray: Decodable { 
 
    let id: Int
    let name: String
    let dic: [Dic] 

    static func decode(json: JSON) -> Decoded<Lcarray> {
          return curry(self.init)
        <^> json <| "id"
        <*> json <| "name"
        <*> json <|| ["dic"]
     }
}
 

struct Dic: Decodable { 
 
    let key: String
    let value: String 

    static func decode(json: JSON) -> Decoded<Dic> {
          return curry(self.init)
        <^> json <| "key"
        <*> json <| "value"
     }
}

```
