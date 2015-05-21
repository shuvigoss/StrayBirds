---
layout: post
title: 【IOS多线程01】- 初识Run Loop
comments: false
---

###什么是Run Loop？
这里，我们通过一个栗子来认识Run Loop。

有一天，接到产品经理的一个需求，我们要做秒杀！需要在我们首页展示秒杀商品，需要有个倒计时，告诉用户活动什么时候结束。

这个BT，想到什么就马上要做。没办法，那我开始想想怎么做。

* 可以把秒杀活动做到Web页面里，通过js控制倒计时。然后拿WebView展示，简单明了。但是我们的首页是Native的，没办法，pass。
* 那就考虑Native实现方式吧。
** 首先在商品上要加个Label，用于展示倒计时。
** 其次写个线程，来计算时间，并且将时间展示在Label上，多简单。

那么开始编码吧。

一个Xib，里边有2个元素，1：UILabel，负责展示剩余时间，2：UIScrollView，首页流式布局。并且建立相应的IBOutlet。

![d191024b-8d2b-4ccd-81ca-5d02bab574bb](https://cloud.githubusercontent.com/assets/3062921/7721158/61ed7738-ff07-11e4-89b7-ddb27d891fc6.png)

```
//ViewController里添加
@property (weak, nonatomic) IBOutlet UILabel *timesLabel;
@property (weak, nonatomic) IBOutlet UIScrollView *scrollView;
@property (nonatomic, strong) NSTimer *timer;
@property (nonatomic) NSInteger maxCount;

- (void)viewDidLoad {
  [super viewDidLoad];
  //初始化内容，高度2000方便滚动
  self.scrollView.contentSize = CGSizeMake(self.view.frame.size.width, 2000);
  //倒计时暂时用个NSInteger代替，从100开始每1秒-1
  self.maxCount = 100;
}

- (void)viewDidAppear:(BOOL)animated{
  [super viewDidAppear:animated];
  self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(showCount:) userInfo:nil repeats:YES];
}

- (void)showCount:(id) obj{
  if (self.maxCount <= 0) {
    [self.timer invalidate];
    self.timer = nil;
  }
  self.timesLabel.text = [NSString stringWithFormat:@"%ld", (long)self.maxCount];
  self.maxCount -- ;
}

```

代码写完了，看看效果，确实不错。越来越觉得自己是个天才。:)

但是出现了个问题，每次滚动scrollView的时候，倒计时就不管用了，停下来后才继续倒计时。fuck，怎么弄？是不是代码写的有问题？

于是开始找问题解决的方法。。。

* [http://www.dreamingwish.com/frontui/article/default/ios-multithread-program-runloop-the.html](http://www.dreamingwish.com/frontui/article/default/ios-multithread-program-runloop-the.html)
* [http://blog.cnbluebox.com/blog/2014/07/01/cocoashen-ru-xue-xi-nsoperationqueuehe-nsoperationyuan-li-he-shi-yong/](http://blog.cnbluebox.com/blog/2014/07/01/cocoashen-ru-xue-xi-nsoperationqueuehe-nsoperationyuan-li-he-shi-yong/)
* [http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/](http://chun.tips/blog/2014/10/20/zou-jin-run-loopde-shi-jie-%5B%3F%5D-:shi-yao-shi-run-loop%3F/)

终于在读完这几篇文章后，恍然大悟，原来不是代码写得有问题。那么怎么改呢？

```
- (void)viewDidAppear:(BOOL)animated{
  [super viewDidAppear:animated];
//  self.timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(showCount:) userInfo:nil repeats:YES];
  self.timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(showCount:) userInfo:nil repeats:YES];
  [[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
}
```

将scheduledTimerWithTimeInterval改为timerWithTimeInterval 并且把timer加入 currentRunLoop中，perfect！

forMode:NSRunLoopCommonModes一定要注意，这才是核心。

好了，我们通过一个demo以及推荐的3篇文章知道了Run Loop是什么了，接下来继续往下看。