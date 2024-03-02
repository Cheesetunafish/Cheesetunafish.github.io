---
layout: simple
title:  "NSNotificationCenter：通知中心"
date:   2024-04-08
---

### 引言

- NSNotificationCenter通知中心是iOS程序内部的一种消息广播的实现机制，通过它能够实现无引用关系的对象之间的通信。
- 通知中心采用的是一对多的方式，一个对象发送的通知可以被多个对象接收。任意对象可以发送通知到中心，同时任意对象也可以监听中心发送的通知。也就是说，把接收到的消息，根据内部的消息转发表，把消息转发给需要的对象。

###  NSNotification

`NSNotification`包含了消息发送的一些信息，包括`name`消息名称、`object`消息发送者、`userinfo`消息发送者携带的额外信息，其类结构如下：

```Objective-C
/****************        Notifications        ****************/

@interface NSNotification : NSObject <NSCopying, NSCoding>
// 通知的名称，有时可能会使用一个方法来处理多个通知，可以根据名称区分
@property (readonly, copy) NSNotificationName name;
// 通知的对象，常使用nil，如果设置了注册的通知监听器的object需要与通知的object匹配，否则接受不到消息
@property (nullable, readonly, retain) id object;
// 字典类型的用户信息，用户可将需要传递的数据放入该字典中
@property (nullable, readonly, copy) NSDictionary *userInfo;

- (instancetype)initWithName:(NSNotificationName)name object:(nullable id)object userInfo:(nullable NSDictionary *)userInfo API_AVAILABLE(macos(10.6), ios(4.0), watchos(2.0), tvos(9.0)) NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)coder NS_DESIGNATED_INITIALIZER;

@end

@interface NSNotification (NSNotificationCreation)

+ (instancetype)notificationWithName:(NSNotificationName)aName object:(nullable id)anObject;
+ (instancetype)notificationWithName:(NSNotificationName)aName object:(nullable id)anObject userInfo:(nullable NSDictionary *)aUserInfo;

- (instancetype)init /*API_UNAVAILABLE(macos, ios, watchos, tvos)*/;        /* do not invoke; not a valid initializer for this class */

@end
```

可以通过实例方法构建`NSNotification`对象，也可以通过类方式构建。

```Objective-C
// 创建NSNotification对象
NSNotification *notification = [NSNotification notificationWithName:@"TestNotification" object:self userInfo:@{@"key": @"value"}];
// 发送NSNotification对象到通知中心
[[NSNotificationCenter defaultCenter] postNotification:notification];
// 创建并发送NSNotification对象
[[NSNotificationCenter defaultCenter] postNotificationName:@"TestNotification" object:self userInfo:@{@"key": @"value"}];
```

### NSNotificationCenter

消息通知中心，全局单例模式(每个进程都默认有一个默认的通知中心，用于进程内通信)，通过如下方法获取通知中心短息：

```Objective-C
+ (NSNotificationCenter *)defaultCenter
```

### NSNotificationQueue

- NSNotificationQueue（通知队列）在NSNotificationCenter起到了一个缓冲的作用。
- 用于管理通知发送的顺序和优先级
- 通知中心默认是采用异步的方式发送通知，如果多个通知同时到达通知中心，它们可能会以随机的顺序被发送到观察者。有时候，我们需要按照一定的顺序或者优先级来发送通知，这时候可以使用 NSNotificationQueue 类

```Objective-C
@interface NSNotificationQueue : NSObject

@property (class, readonly, strong) NSNotificationQueue *defaultQueue;

- (instancetype)initWithNotificationCenter:(NSNotificationCenter *)notificationCenter NS_DESIGNATED_INITIALIZER;

// 将通知添加到队列中
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle;
- (void)enqueueNotification:(NSNotification *)notification postingStyle:(NSPostingStyle)postingStyle coalesceMask:(NSNotificationCoalescing)coalesceMask forModes:(nullable NSArray<NSRunLoopMode> *)modes;

// 从队列中移除符合条件的通知
- (void)dequeueNotificationsMatching:(NSNotification *)notification coalesceMask:(NSUInteger)coalesceMask;

@end
```

其中，添加通知到队列的两个参数枚举如下：

```Objective-C
// NSPostingStyle: 配置通知什么时候发送
typedef NS_ENUM(NSUInteger, NSPostingStyle) {
    NSPostWhenIdle = 1, // 在runloop处于空闲时发出通知
    NSPostASAP = 2,     // 在当前通知调用或者计时器结束发出通知
    NSPostNow = 3       // 在合并通知完成之后立即发出通知
};

// NSNotificationCoalescing：用于配置如何合并通知
typedef NS_OPTIONS(NSUInteger, NSNotificationCoalescing) {
    NSNotificationNoCoalescing = 0,        // 不合并通知
    NSNotificationCoalescingOnName = 1,    // 按照通知名字合并通知
    NSNotificationCoalescingOnSender = 2   // 按照传入的object合并通知
};
```

### 实现原理

![img](https://redrock.feishu.cn/space/api/box/stream/download/asynccode/?code=NjUzODQwMTM4YjlmYWY3NzU0YmZiNTY3MTA4N2I1MWZfZzVRNkFKT01MT283WGo3RktoZnZEdmVBSHBhTm5TMENfVG9rZW46WThpQmJ2a05Ibzd2S2J4aVlTa2NDTkQwbkdiXzE3MDkzNjYwMzU6MTcwOTM2OTYzNV9WNA)

### Table

在NSNotificationCenter内部一共保存了两张表。一张用于保存添加观察者的时候传入了NotifcationName的情况，一张用于保存添加观察者的时候没有传入了NotifcationName的情况，下面分两种情况分析。

#### Named Table

先看一下表中保存的内容及Key，Value类型。

![img](https://redrock.feishu.cn/space/api/box/stream/download/asynccode/?code=MDBlMzE1MzcyNTBmMzkwYjM5ZjQ0NTA3NmY2M2QzYTZfbmxvblRYTjRGdGZiWFpPMWk1aEpTcGV4dWtvU1RHUXVfVG9rZW46U3I5aWJQd1Zvb0ZtRzB4S0pYWGNxb01rbmtmXzE3MDkzNjYwMzU6MTcwOTM2OTYzNV9WNA)

在Named Table中，NotifcationName作为表的key，**因为我们在注册观察者的时候是可以传入一个参数object用于只监听指定该对象发出的通知，并且一个通知可以添加多个观察者，所以还需要一张表来保存object和Observer的对应关系。这张表的是key、Value分别是以object为Key，Observer为value。**

所以Named Table结构大致如下：

- 首先外层有一个Table，以通知名称为Key。其Value同样是一个Table（简称为Table）。
- 为了实现可以传入一个参数object用于只监听指定该对象发出的通知，及一个通知可以添加多个观察者。则内Table的以传入的Object为Key，用链表来保存所有的观察者，并且以这个链表为Value。

#### UnNamed Table

`UnNamed Table`没有`NSNotificationName`作为key，所以只有`object`和`observer`。

![img](https://redrock.feishu.cn/space/api/box/stream/download/asynccode/?code=NWVhZWNiYWM5MzFmOTUwNTEyYjA0NjQwNDIwMmIzOTNfaTlHUzNXSlVnWFVLRms2SndlWXZKR1d4TDBGcDQwMllfVG9rZW46VTJINWJVcGhsb2RTMFR4RjJDOGNqcng4bjhiXzE3MDkzNjYwMzU6MTcwOTM2OTYzNV9WNA)

### 使用通知

1. 在需要接受通知的地方添加注册观察者
2. 在需要的时候发送通知消息
3. 接受通知，以及接受通知后执行的方法
4. 移除观察者

### 通知优缺点：

#### 优势：

1. 不需要编写多少代码，实现比较简单。
2. 对于一个发出的通知，多个对象能够做出反应，即**1对多的方式**实现简单。
3. controller能够传递context对象（dictionary），context对象携带了关于发送通知的自定义的信息。

#### 缺点：

1. 在编译期**不会检查**通知是否能够被观察者正确的处理。
2. 在释放注册的对象时，**需要在通知中心取消注册。**
3. 在调试的时候应用的工作以及控制过程**难跟踪**。
4. 通常需要第三方库来管理控制器与观察者对象之间的联系，来避免内存泄漏和崩溃问题。
5. 控制器和观察者需要提前知道通知名称、UserInfo dictionary keys。如果这些没有在工作区间定义，那么会出现不同步的情况。
6. 通知发出后，控制器不能从观察者获得任何的反馈信息。

### 通知的线程安全问题

1. 通知中心是一个**全局的单例**，可以在**任何线程**中发送和接收通知。但是，如果多个线程同时访问通知中心，可能会出现竞争条件和死锁等问题。
2. 在注册观察者和发送通知时，可能会出现**线程不安全**的情况。例如，在一个线程中注册观察者，在另一个线程中发送通知，可能会导致观察者没有被正确注册，从而无法接收通知。所以一般建议**在主线程内进行注册**观察者和发送通知。
3. 在移除观察者时，可能会出现线程安全的问题。如果一个观察者在多个线程中注册了多次，需要在每个线程中都移除观察者才能保证线程安全。否则，如果一个线程移除了观察者，而另一个线程还在使用该观察者，可能会导致程序崩溃或出现其他问题。
4. 通知中心代码放置地
   1. viewWillApperar中注册，viewWillDisappear中移除，减少对于性能的影响。
   2. viewDidLoad中注册。如果在viewWillApperar中注册却没有在viewWillDisappear中移除那么会造成注册很多次，可能会造成一个事件触发后被多次响应。

### 通知中心与KVO

通知和 KVO 都是 iOS 开发中常用的两种通信机制，它们都可以用于对象间的消息传递。两者区别如下：

1. 监听对象：通知中心通知是一种广播机制，**任何对象**都可以通过通知中心发送通知，而监听通知的对象也可以是任何对象；KVO 是**针对某个具体的对象**进行监听，通过对某个对象的属性进行监听来得知属性值的变化。
2. 监听的内容：通知中心**可以发送任意类型的数据**，而 KVO 只能**对某个对象的属性**进行监听。
3. 实现方式不同

# 手势UIGestureRecognizer

### 引言

在iOS系统中，手势是一个重要的用户交互方式。通过继承UIGestureRecognizer类，可以实现很多手势操作并应用于app中。关于UIGestureRecognizer类，是对iOS中的事件传递机制面向应用的封装，将手势消息的传递抽象为了对象。UIGestureRecognizer是一个手势识别器，通常作为父类，实际操作中要使用的是他的子类，常见的有以下几种：

```Objective-C
UITapGestureRecognizer           // 轻拍手势
UISwipeGestureRecognizer         // 轻扫手势
UILongPressGestureRecognizer     // 长按手势
UIPanGestureRecognizer           // 平移手势
UIPinchGestureRecognizer         //捏合（缩放）手势
UIRotationGestureRecognizer      // 旋转手势
UIScreenEdgePanGestureRecognizer // 屏幕边缘平移
```

属性、常用初始化方法如下：

```Objective-C
@interface UIGestureRecognizer : NSObject
//初始化，手势别后会调用 target 的 action 方法
- (instancetype)initWithTarget:(nullable id)target action:(nullable SEL)action NS_DESIGNATED_INITIALIZER; // designated 
- (instancetype)init;
- (nullable instancetype)initWithCoder:(NSCoder *)coder;
//添加调用方法
- (void)addTarget:(id)target action:(SEL)action;   
//移除方法
- (void)removeTarget:(nullable id)target action:(nullable SEL)action;
//状态
@property(nonatomic,readonly) UIGestureRecognizerState state;  // the current state of the gesture recognizer
//代理
@property(nullable,nonatomic,weak) id <UIGestureRecognizerDelegate> delegate; // the gesture recognizer's delegate
//能否识别
@property(nonatomic, getter=isEnabled) BOOL enabled;
//是哪个 view 的手势
@property(nullable, nonatomic,readonly) UIView *view;         
//获取手势在view中的坐标
- (CGPoint)locationInView:(nullable UIView*)view;                          
@end
```

手势状态的枚举如下：

```Objective-C
typedef NS_ENUM(NSInteger, UIGestureRecognizerState) {
    UIGestureRecognizerStateBegan,      //手势开始，第一次调用 target 方法时就是这个 state
    UIGestureRecognizerStateChanged,   //状态变化的一个状态，如拖拽手势在第一次调用 target 方法后的一系列调用都是这个状态
    UIGestureRecognizerStateEnded,     //最后一次调用 target 方法就是这个状态
};
```

### 常见手势使用

#### UITapGestureRecognizer：轻拍手势

##### 属性

```Objective-C
@property (nonatomic) NSUInteger  numberOfTapsRequired;       // 需要轻拍的次数，默认为1
@property (nonatomic) NSUInteger  numberOfTouchesRequired API_UNAVAILABLE(tvos);    // 需要执行轻拍手势的手指数量，默认为1

@property (nonatomic) UIEventButtonMask buttonMaskRequired // 手势应该检测的触摸事件的类型，默认为单击触摸事件
```

若要给一个控件添加轻拍手势，可以按照以下方法添加。例如，要将一个`testView`通过轻拍手势改变颜色，则需要给这个view添加一个tap手势：

```Objective-C
- (void)addTapGesture {
    // 初始化手势，并添加处理方法
    UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(handleTapGesture)];
    tap.numberOfTapsRequired = 1;    // 需要的轻拍次数
    tap.numberOfTouchesRequired = 1; // 需要手指数量
    // 为view添加上手势识别器
    [self.testView addGestureRecognizer:tap];
}

#pragma mark - action
- (void)handleTapGesture {
    NSLog(@"执行轻拍手势");
    self.testView.backgroundColor = [UIColor systemRedColor];
}
```

#### UISwipeGestureRecognizer：轻扫手势

##### 属性

```Objective-C
@property(nonatomic) NSUInteger numberOfTouchesRequired API_UNAVAILABLE(tvos); // 需要的手指数量，默认为1
@property(nonatomic) UISwipeGestureRecognizerDirection direction; 
```

其中轻扫手势方向的枚举如下：

```Objective-C
// 默认从左往右
typedef NS_OPTIONS(NSUInteger, UISwipeGestureRecognizerDirection) {
    UISwipeGestureRecognizerDirectionRight = 1 << 0,    // 向右
    UISwipeGestureRecognizerDirectionLeft  = 1 << 1,    // 向左
    UISwipeGestureRecognizerDirectionUp    = 1 << 2,    // 向上
    UISwipeGestureRecognizerDirectionDown  = 1 << 3     // 向下
};
```

若需要为`testView`添加一个轻扫手势，则代码如下：

```Objective-C
- (void)addSwipeGesture {
    UISwipeGestureRecognizer *swipe = [[UISwipeGestureRecognizer alloc] initWithTarget:self action:@selector(handleSwipeGesture)];
    swipe.direction = UISwipeGestureRecognizerDirectionRight;
    [self.testView addGestureRecognizer:swipe];
}

#pragma mark - action
- (void)handleSwipeGesture {
    NSLog(@"执行轻扫手势");
    self.testView.transform = CGAffineTransformTranslate(self.testView.transform, 50, 0);// 使view向右移动
}
```

#### UILongPressGestureRecognizer：长按手势

##### 属性

```Objective-C
@property (nonatomic) NSUInteger numberOfTapsRequired; // 要求的触摸次数，默认为0，表示不需要任何额外的手势触发。
@property (nonatomic) NSUInteger numberOfTouchesRequired; // 要求的手指数，默认为1，表示只需要一个手指长按。

@property (nonatomic) NSTimeInterval minimumPressDuration; // 长按的最小时间，默认为0.5秒。
@property (nonatomic) CGFloat allowableMovement; // 手指可以移动的范围，在这个范围内松开手指才会触发手势，默认为10点。
```

若需要为`testView`添加一个长按手势，则代码如下：

```Objective-C
- (void)addLongPressGesture {
    UILongPressGestureRecognizer *longPress = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(handleLongPressGesture)];
    longPress.minimumPressDuration = 1;
    [self.testView addGestureRecognizer:longPress];
}

#pragma mark - action
- (void)handleLongPressGesture {
    NSLog(@"执行长按手势");
    self.testView.backgroundColor = [UIColor blackColor];
}
```

#### UIPanGestureRecognizer：平移手势

##### 属性

```Objective-C
@property (nonatomic) NSUInteger minimumNumberOfTouches; // 识别手势所需的最小触摸点数。 
@property (nonatomic) NSUInteger maximumNumberOfTouches; // 识别手势所需的最大触摸点数。

- (CGPoint)translationInView:(nullable UIView *)view; // 手势识别器的平移量（相对于视图）。
- (void)setTranslation:(CGPoint)translation inView:(nullable UIView *)view; // 设置手势识别器的平移量（相对于视图）。

- (CGPoint)velocityInView:(nullable UIView *)view; // 手势识别器的平移速度（相对于视图）。
```

若需要移动`testView`的位置，使用平移手势代码如下：

```Objective-C
- (void)addPanGesture {
    UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(handlePanGesture:)];
    [self.testView addGestureRecognizer:pan];
}

#pragma mark - action
- (void)handlePanGesture:(UIPanGestureRecognizer *)sender {
    NSLog(@"执行平移手势");
    // 获取手势识别器的平移量
    CGPoint point = [sender translationInView:self.testView];
    // 设置testView的frame
    sender.view.transform = CGAffineTransformTranslate(sender.view.transform, point.x, point.y);
    // 设置手势识别器的平移量
    [sender setTranslation:CGPointZero inView:sender.view];
}
```

#### UIScreenEdgePanGestureRecognizer：屏幕边缘平移手势

##### 属性

```Objective-C
@property (readwrite, nonatomic, assign) UIRectEdge edges; // 手势应该从哪个边缘识别。
```

`UIScreenEdgePanGestureRecognizer`继承自`UIPanGestureRecognizer`，可以识别用户从屏幕边缘开始的平移手势。该手势通常用于呼出侧滑菜单或执行其他与边缘相关的操作。

#### UIPinchGestureRecognizer：捏合手势

##### 属性

```Objective-C
@property (nonatomic) CGFloat scale; // 当前手势的缩放比例，手指距离缩放前后的比值。
@property (nonatomic,readonly) CGFloat velocity; // 当前手势的缩放速度，是一个 CGPoint 类型，表示每秒缩放的比例。
```

若以scale为1的缩放比例对`testView`进行缩放，且每次缩放从当前大小开始，则代码如下：

```Objective-C
- (void)addPinchGesture {
    UIPinchGestureRecognizer *pinch = [[UIPinchGestureRecognizer alloc] initWithTarget:self action:@selector(handlePinchGesture:)];
    [self.testView addGestureRecognizer:pinch];
}

#pragma mark - action
- (void)handlePinchGesture:(UIPinchGestureRecognizer *)sender {
    NSLog(@"执行捏合手势");
    sender.view.transform = CGAffineTransformScale(sender.view.transform, sender.scale, sender.scale);
    sender.scale = 1;
}
```

#### UIRotationGestureRecognizer：旋转手势

##### 属性

```Objective-C
@property (nonatomic) CGFloat rotation; // 当前手势旋转的弧度值。
@property (nonatomic,readonly) CGFloat velocity; // 当前手势旋转的速度，单位为弧度/秒。
```

### 响应链

- 用户手势通过**事件流**传递进入系统，而事件流通过响应链(或响应者链)进行实际传递，故需要了解响应链的内容，从而理解如何进行手势处理。
- 响应链由一系列能够响应用户事件的响应者对象构成，响应者对象为`UIResponder`的子类对象。`UIView`，`UIViewController`，`UIApplication`，甚至是 `UIApplication` 的代理对象(通常是名为 `AppDelegate` 类的对象)均继承自 `UIResponder`。所以响应链就是响应者组成的链式结构。

继承关系如下所示：

```Objective-C
                           UIView -> UIWindow
NSObject -> UIResponder -> UIView -> UIControl -> UIButton
            UIResponder -> UIApplication
            UIResponder -> UIViewController
NSObject -> UIGestureRecognizer                                                                                                                                                                                                                                                                                                                                                                                                                                         

//UIResponder 有一个属性 nextResponder ，这个属性就是该对象的下一响应者。
@property(nonatomic, readonly, nullable) UIResponder *nextResponder;
```

- 程序启动后, 首先在 main 函数中创建了 `UIApplication` 对象和它的代理对象，而后程序代理对象持有 window，window 持有根视图控制器，这一条线上的所有对象都是 `UIResponder`，另外所有的视图和视图控制器都是 `UIResponder`，可以看到，实际上响应者充斥着整个程序。

### **事件处理**

- 在某个响应者上重写了`UIResponder`的处理方法。
- 当某个响应者不处理某个事件时(不重写 `UIResponder` 的触摸处理方法), 该事件就会沿响应链向上传递, 而如果某个响应者能够处理该事件, 则它就将该事件“消费”掉, 从而终止事件的传递。而如果整个响应链都没有处理该事件，那么事件将会被丢弃。

### Hit-Testing

当我们触摸屏幕时，到底应该由哪个对象最先响应这个事件呢？这就需要去探测，这个过程称为`Hit-Testing`，最后的结果称为`hit-test view`。涉及到两个方法是：

```Objective-C
//先判断点是否在View内部，然后遍历subViews
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event; 
//判断点是否在这个View内部
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event;
```

`Hit-Testing`是一个递归的过程，每一步监测触摸位置是否在当前view中，如果是，就递归监测subviews；否则，返回nil。

递归的根节点是UIWindow，对subviews的遍历顺序按照**后添加的先遍历**原则。

例如，如下视图，点击E时的探测步骤：

![img](https://redrock.feishu.cn/space/api/box/stream/download/asynccode/?code=Mjc1NWMyNGU4MzFjNjViZDRmNzFhYTk1ZDAzMjg0MjZfTWJJSmpYMnEzY3NjT3F6V28zNG93R0F4M1NNTTRxaVVfVG9rZW46QVpac2J0TWVwb0xwY1d4VUlEMWN1RENSbkxkXzE3MDkzNjYwMzU6MTcwOTM2OTYzNV9WNA)

1. 触摸点在A的范围内，所以继续探测A的subView，既view B和view C。
2. 触摸点不在view B里，在view C里，所以继续探测C的subView D 和 E。
3.  触摸点不在D里，在E里，E已经是最低级的View，故返回E。

综上，我们可以看出有两个几乎相反的链，一是**响应链**，一是**探测链**。 有触摸事件时，先依赖探测链来确定响应链的开始节点(UIResponder对象)，然后依赖响应链来确定最终处理事件的对象。