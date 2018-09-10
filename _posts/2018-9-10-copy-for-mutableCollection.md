---
layout: post
title: 刨根问底：为什么不能用 copy 修饰 NSMutableArray ？
tags: [runtime,copy NSMutableArray]
---

## 前言

```objc
@property (nonatomic,copy) NSMutableArray *imArray;
```

会有什么问题？

copy 修饰的 imArray 在赋值后，运行时会变成 NSArray 也就是不可变类型，所以之后使用 NSMutableArray 的方法时，会产生 `unrecognized selector` 异常。

为什么会这样？

## 底层实现

### 测试

测试类如下：

```objc
@interface FooModel : NSObject
@property (nonatomic,strong) NSString *mString;
@property (nonatomic,copy) NSString *imString;
@property (nonatomic,copy) NSArray *contrastArray;
@property (nonatomic,copy) NSMutableArray *imArray;
@property (nonatomic,strong) NSMutableArray *mArray;
@end

@implementation FooModel
@end
```

测试

```objc
	FooModel *model = [FooModel new];
	
	{	// 1.
		NSLog(@"------1.------");
		NSMutableString *mString = [[NSMutableString alloc] initWithString:@"m string"];
		model.mString = mString;
		NSLog(@"m string: %@",model.mString);
		[mString appendString:@" -end"];
		NSLog(@"m string: %@",model.mString);
	}
	
	{	// 2.
		NSLog(@"------2.------");
		NSMutableString *mString = [[NSMutableString alloc] initWithString:@"m string"];
		model.imString = mString;
		NSLog(@"m string: %@",model.imString);
		[mString appendString:@" -end"];
		NSLog(@"m string: %@",model.imString);
	}
	
	{	// 3.
		NSLog(@"------3.------");
		NSMutableArray *tempArray =  [@[@"m-array-1",@"m-array-2",@"m-array-3"] mutableCopy];
		model.contrastArray =  tempArray;
		NSLog(@"contrastArray: %@",model.contrastArray);
		[tempArray replaceObjectAtIndex:2 withObject:@"replace string"];
		NSLog(@"contrastArray: %@",model.contrastArray);
	}

	{	// 4.
		NSLog(@"------4.------");
		NSMutableArray *tempArray =  [@[@"m-array-1",@"m-array-2",@"m-array-3"] mutableCopy];
		model.imArray =  tempArray;
		[model.imArray replaceObjectAtIndex:2 withObject:@"replace string"];
	}
```

输出结果：

```
------1.------
m string: m string
m string: m string -end
------2.------
m string: m string
m string: m string
------3.------
contrastArray: (
    "m-array-1",
    "m-array-2",
    "m-array-3"
)
contrastArray: (
    "m-array-1",
    "m-array-2",
    "m-array-3"
)
------4.------
-[__NSArrayI replaceObjectAtIndex:withObject:]: unrecognized selector sent to instance 0x100b78010
```


从结论得知：

当 mString 改变时，copy 和 strong 修饰的 NSString 属性变化

- copy 修饰的 NSString 不会随着外部变化而变化
- strong 会随着外部变化而变化

对比 NSMutableArray 属性

- copy 和 strong 修饰的 NSMutableArray 都不会随着外部变化而变化
- copy 修饰的 NSMutableArray 无法找到`replaceObjectAtIndex:withObject:` 方法，会发生异常


### 重写 `clang -rewrite-objc`

让我们看看运行时是如何实现的，重写（clang -rewrite-objc）FooModel 类

```c++
static NSString * _I_FooModel_mString(FooModel * self, SEL _cmd) { return (*(NSString *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_mString)); }
static void _I_FooModel_setMString_(FooModel * self, SEL _cmd, NSString *mString) { (*(NSString *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_mString)) = mString; }

static NSString * _I_FooModel_imString(FooModel * self, SEL _cmd) { return (*(NSString *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_imString)); }
extern "C" __declspec(dllimport) void objc_setProperty (id, SEL, long, id, bool, bool);

static void _I_FooModel_setImString_(FooModel * self, SEL _cmd, NSString *imString) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct FooModel, _imString), (id)imString, 0, 1); }

static NSArray * _I_FooModel_contrastArray(FooModel * self, SEL _cmd) { return (*(NSArray *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_contrastArray)); }
static void _I_FooModel_setContrastArray_(FooModel * self, SEL _cmd, NSArray *contrastArray) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct FooModel, _contrastArray), (id)contrastArray, 0, 1); }

static NSMutableArray * _I_FooModel_imArray(FooModel * self, SEL _cmd) { return (*(NSMutableArray *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_imArray)); }
static void _I_FooModel_setImArray_(FooModel * self, SEL _cmd, NSMutableArray *imArray) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct FooModel, _imArray), (id)imArray, 0, 1); }

static NSMutableArray * _I_FooModel_mArray(FooModel * self, SEL _cmd) { return (*(NSMutableArray *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_mArray)); }
static void _I_FooModel_setMArray_(FooModel * self, SEL _cmd, NSMutableArray *mArray) { (*(NSMutableArray *__strong *)((char *)self + OBJC_IVAR_$_FooModel$_mArray)) = mArray; }
```

发现当设定 copy 属性时，如：

- NSString 用 copy 修饰
- NSArray 用 strong/weak 修饰
- NSMutableArray 用 copy 修饰

都调用了 `void objc_setProperty (id, SEL, long, id, bool, bool)` 方法。

```c++
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
	...
    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }
	...
}
```

因此，当属性设定 copy 时，都会调用 `objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct FooModel, _xxx), (id)xxx, 0, 1)` 方法，同时 copy 值传入的是 1，也就是 true。

然后在实现里，当 copy 是 true 时，设定方法通过 copyWithZone 赋值，然后产出的不可变的新值。

所以 NSArray 和 copy 修饰的 NSString 表现形式是一样的，当外部的可变对象变化时，自身不会变化。

而 NSMutableArray 用 copy 修饰的时候，在赋值之后，自身的类型在运行时被改变了，变成了 NSArray 类型，这时再使用 NSMutableArray 方法就会产生 `unrecognized selector` 异常。

## 总结
当 copy 修饰可变类型集合（例如：NSMutableArray）时，赋值后，会导致可变类型属性变为不可变类型，然后在调用可变类型方法时，会产生异常错误。

产生异常的原因是 copy 属性在运行时赋值时调用了 `-copyWithZone:`赋值，将可变类型转换为不可变类型。

