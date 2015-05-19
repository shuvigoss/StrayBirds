---
layout: post
title: self vs underscore(ARC)
comments: false
---

## self.xxxx 与 _xxxx的正确用法
之前在写代码时我为了图省事，所有的通过property声明的变量都使用_xxx = xxx的方式来赋值，但是这样做会有很严重的问题，我们看看下边的这段代码。

我在一个ViewController类中声明了2个属性。

```
@property (nonatomic, strong) NSMutableString *stringStrong;
@property (nonatomic, copy) NSMutableString *stringCopy;
```

意图很清楚，我想stringStrong对象是个强引用，而stringCopy则是个深copy的NSMutableString。

接下来我在viewDidLoad内这么初始化：

```
- (void)viewDidLoad {
  [super viewDidLoad];
  NSMutableString *temp = [NSMutableString stringWithFormat:@"a"];
  _stringStrong = temp;
  _stringCopy = temp;
  [_stringStrong appendString:@"b"];
  NSLog(@"%@", _stringCopy);
  NSLog(@"%@", temp);
}
```

猜猜打印结果是什么？

```
[78196:568691] ab
[78196:568691] ab
```

怎么回事？stringCopy不是copy对象吗？其实这是个很愚蠢的问题，但是我在写MRC时都是这么写的，因为我知道我这个对象到底应该是retain、还是copy。

那应该怎么写呢？

```
NSMutableString *temp = [NSMutableString stringWithFormat:@"a"];
self.stringStrong = temp;
self.stringCopy = temp;
[self.stringStrong appendString:@"b"];
NSLog(@"%@", self.stringCopy);
NSLog(@"%@", temp);
```

```
[78265:572002] a
[78265:572002] ab
```

这样的输出才是对的。为了不犯错误，所有属性都通过self.xxx来调用就好了。`_xxxx = xxx`就是一个retain操作。在使用时注意。