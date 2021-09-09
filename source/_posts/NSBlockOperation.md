---
title: NSBlockOperation
date: 2020-11-16 15:04:33
Tags: ios
---



在日常开发中，我们可能会用到 NSBlockOperation 来做一些多线程的操作。

```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
NSBlockOperation *operation = [MyBlockOperation blockOperationWithBlock:^{
    // do something
}];
[queue addOperation:operation];
```

因为其使用简单，所以也被广泛使用。

但这个方法也有一个缺点，当 operation 开始执行后，[operation cancel]; 并不能及时取消。

同时，blockOperationWithBlock 没有 operation 的回调参数。有些同学想要通过 operation.isCancelled 进行任务的及时取消，不太清楚应该怎么做，要是使用不当甚至会引入 循环引用 问题。

下面，我们通过几个面试题来一步步了解清楚这个问题。

### 1.1 

下面的代码输出什么？

- cancelled
- finish

```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    sleep(2);
    if (operation.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```

**答案**

- finish

**分析**

初始化时传入了一个 block，block 中含有 operation 对象的访问，即发生了对象的捕获。但是现在是对象的创建过程中，operation 还没有被初始化出来，block这时捕获的是 operation 当前的值，即 operation = nil。

<!--more-->

### 1.2 

下面的代码输出什么？

- cancelled
- finish

```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
__block NSBlockOperation *operation = [NSBlockOperation blockOperationWithBlock:^{
    sleep(2);
    if (operation.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```

**答案**

- cancelled

**分析**

这次加入了 `__block` 修饰符，捕获变为了指针引用，所以这次 [operation cancel]; 生效了。
简单来说就是：

未使用 `__block` 修饰符，捕获的是当前值。

```
int i = 0;
void(^block)(void) = ^{
    i = 1;
    printf("%d", i); // 1
};
i = 2;
block();
```

上面这个问题已经有很多文章了，这里不深入说明。

```
NSString *str = @"0";
void(^block)(void) = ^{
    NSLog(@"%@", str); // 0
};
str = @"1";
block();
```

部分同学可能对这个有疑问。其实，两种捕获的原理都是对变量的 “值引用” 进行捕获，捕获的是当前对象的值。之后的修改，不会影响到 block 内部的值。

### 1.3 

下面的代码输出什么？

- cancelled, -[MyBlockOperation dealloc]
- 只有 cancelled

```
@interface MyBlockOperation : NSBlockOperation
@end

@implementation MyBlockOperation
- (void)dealloc {
    NSLog(@"%s", __FUNCTION__);
}
@end

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
__block MyBlockOperation *operation = [MyBlockOperation blockOperationWithBlock:^{
    sleep(2);
    if (operation.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```

**答案**

- 只有 cancelled

**分析**

刚刚分析了 `__block` 修饰符，捕获变为了指针引用。现在 operation 对象持有 block ，同时 block 通过指针引用捕获了 operation 对象，所以发生了循环引用。

那可能有些同学就要说了，加上 `__weak` 不就好了？

### 1.4 

下面的代码输出什么？

- cancelled, -[MyBlockOperation dealloc]
- 只有 cancelled
- 只有 -[MyBlockOperation dealloc]

```
@interface MyBlockOperation : NSBlockOperation
@end

@implementation MyBlockOperation
- (void)dealloc {
    NSLog(@"%s", __FUNCTION__);
}
@end

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
__block __weak MyBlockOperation *operation = [MyBlockOperation blockOperationWithBlock:^{
    sleep(2);
    if (operation.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```

**答案**

- 只有 -[MyBlockOperation dealloc]

这里加入了 __weak 修饰符，对象创建后，发现引用计数为 0 就被释放了，所以 block 不执行。这部分属于内存管理的知识点。



那么我们要怎么做，才能既捕获到 operation 又不会导致循环引用呢？

### 1.5 

下面的代码输出什么？

- cancelled, -[MyBlockOperation dealloc]
- 只有 cancelled
- 只有 -[MyBlockOperation dealloc]

```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
__block __weak MyBlockOperation *wkOp;
MyBlockOperation *operation = [MyBlockOperation blockOperationWithBlock:^{
    sleep(2);
    if (wkOp.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
wkOp = operation;
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```

**答案**

- cancelled, -[MyBlockOperation dealloc]

**分析**

为了让 block 访问到初始化之后的值，wkOp 用了 `__block` 修饰符；同时，为了避免循环引用问题，wkOp 还用了 `__weak` 修饰符。

operation 初始化后被赋值，这里会有一个隐式的 `__strong` 修饰，即这里会对 operation 进行强引用，引用计数 +1 。wkOp 为弱引用对象，wkOp = operation; 只是用一个弱引用指针指向 operation ，引用计数不变。与之前不同的是，此时的 operation 的引用计数大于 0，所以不会被立刻释放。

## 2. 封装

为了日常使用方便封装一下就更香了～

```
@interface MyBlockOperation : NSBlockOperation

+ (instancetype)blockOperationWithBlock:(void (^)(void))block NS_UNAVAILABLE;
+ (instancetype)blockOperationWithOperationBlock:(void (^)(MyBlockOperation *))block;

@end

@implementation MyBlockOperation
- (void)dealloc {
    NSLog(@"%s", __FUNCTION__);
}

+ (instancetype)blockOperationWithOperationBlock:(void (^)(MyBlockOperation *))block;
{
    __block __weak MyBlockOperation *wkOp;
    MyBlockOperation *op = [super blockOperationWithBlock:^{
        !block ?: block(wkOp);
    }];
    wkOp = op;
    return op;
}

@end

NSOperationQueue *queue = [[NSOperationQueue alloc] init];
MyBlockOperation *operation = [MyBlockOperation blockOperationWithOperationBlock:^(MyBlockOperation *op){
    sleep(2);
    if (op.isCancelled) {
        NSLog(@"cancelled");
    } else {
        NSLog(@"finish");
    }
}];
[queue addOperation:operation];
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(1.0 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    [operation cancel];
});
```