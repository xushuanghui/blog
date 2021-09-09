---
title: app启动时间优化
date: 2021-03-21 16:36:16
tags:
---

[《Optimizing App Startup Time》议题](https://link.zhihu.com/?target=https%3A//developer.apple.com/videos/play/wwdc2016/406/)上，提到了启动分为冷启动和暖启动两种。

暖启动：是指内存中包含 App 的相关数据，与我们日常提到的热启动不太一样，连续杀掉 App 的启动也可能是暖启动。

冷启动：是指系统的内核缓存区里没有 App 相关数据。冷暖启动的启动时间会差异比较大，冷启动的数据更能反映 App 真实的启动时长。保证 App 是冷启动的方法，就是通过重启设备，清理系统内核缓存区。

常见的 iOS 启动时长测试方法，主要有以下几种

1. **Xcode Developer Tool**： 使用 Instruments 的 Time Profiler 插件，可以检测 App CPU 的使用情况。能看到 App 的启动时间和各个方法消耗的时间；
2. **客户端计算统计**： 通过 hook 关键函数的调用，计算获得性能数据。

 **App总启动流程 = pre-main (+load,initializer)+ main函数代理（didFinishLaunchingWithOptions）+ 首屏渲染（viewDidAppear）**，后两个阶段都属于 `main函数` 执行阶段。

![image-20210310151027524](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210310151027524.png)

### pre-main阶段

- 加载 dyld
- 创建启动闭包（更新 App/重启手机需要）
- 加载动态库
- Bind & Rebase & Runtime 初始化
- +load 和静态初始化

获得 main() 方法执行前的耗时比较简单，通过 Xcode 自带的测量方法既可以。将 Xcode 中 Product -> Scheme -> Edit scheme -> Run -> Environment Variables 将环境变量 `DYLD_PRINT_STATISTICS` 或 `DYLD_PRINT_STATISTICS_DETAILS`  设为 `1` 即可获得执行每项耗时：

```
Total pre-main time: 977.95 milliseconds (100.0%)
         dylib loading time: 433.20 milliseconds (44.2%)
        rebase/binding time:  97.24 milliseconds (9.9%)
            ObjC setup time:  50.13 milliseconds (5.1%)
           initializer time: 397.36 milliseconds (40.6%)
           slowest intializers :
             libSystem.B.dylib :  11.48 milliseconds (1.1%)
                   AgoraRtcKit : 105.87 milliseconds (10.8%)
                        Hestia : 415.46 milliseconds (42.4%)

```



### main函数代理阶段

#### 1、手动插入代码计算耗时

  在 `man()` 函数开始执行时就开始时间：

  ```
  CFAbsoluteTime StartTime;  //  记录全局变量
  int main(int argc, char * argv[]) {
      @autoreleasepool {
          StartTime = CFAbsoluteTimeGetCurrent();
          return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
      }
  }
  ```

无侵入首屏渲染完成时间我们希望和 `MetricKit` 对齐，即获取到 `CA::Transaction::commit()`方法被调用的时间。

`CA::Transaction::commit()`，`CFRunLoopPerformBlock`，`kCFRunLoopBeforeTimers` 这三个时机的顺序从早到晚依次是：

![image-20210311113820158](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210311113820158.png)

可以通过在 didFinishLaunch 中向 Runloop 注册 block 或者 BeforeTimer 的 Observer 来获取上图中两个时间点的回调，代码如下：

```objective-c
if ([UIDevice currentDevice].systemVersion.floatValue >= 13.0) {
        //注册kCFRunLoopBeforeTimers回调
        CFRunLoopRef mainRunloop = [[NSRunLoop mainRunLoop] getCFRunLoop];
        CFRunLoopActivity activities = kCFRunLoopAllActivities;
        CFRunLoopObserverRef observer = CFRunLoopObserverCreateWithHandler(kCFAllocatorDefault, activities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
            if (activity == kCFRunLoopBeforeTimers) {
                NSTimeInterval stamp = [[NSDate date] timeIntervalSince1970];
                NSLog(@"runloop beforetimers launch end:%f",stamp);
                CFRunLoopRemoveObserver(mainRunloop, observer, kCFRunLoopCommonModes);
                [HSALaunchManage initLaunchService];
            }
        });
        CFRunLoopAddObserver(mainRunloop, observer, kCFRunLoopCommonModes);
    }
    else {
        //    注册block
        CFRunLoopRef mainRunloop = [[NSRunLoop mainRunLoop] getCFRunLoop];
        CFRunLoopPerformBlock(mainRunloop,NSDefaultRunLoopMode,^(){
            NSTimeInterval stamp = [[NSDate date] timeIntervalSince1970];
            NSLog(@"runloop block launch end:%f",stamp);
            [HSALaunchManage initLaunchService];
        });
    }

```

1. iOS13（含）以上的系统采用 `runloop` 中注册一个 `kCFRunLoopBeforeTimers` 的回调获取到的 App 首屏渲染完成的时机更准确。
2. iOS13 以下的系统采用 `CFRunLoopPerformBlock` 方法注入 block 获取到的 App 首屏渲染完成的时机更准确。

### 启动时间统计:

#### main()函数启动时间

**MianStartTime:1617002963.711972**

**1617002963.862700**

#### 首屏结束时间：

**runloop beforetimers launch end:1617002965.720096**

Total pre-main time: 797.23 milliseconds

首屏时间-Mianstart  5.72-3.7119=2.0081 秒 

总时间：2.0081+0.977=2.9851 秒



***对于 main() 调用之前的耗时可以优化的点有***

- 1、减少不必要的 framework，因为动态链接比较耗时

- 2、check framework 应当设为 optional 和 required，如果该 framework 在当前 App 支持的所有 iOS 系统版本都存在，那么就设为 required，否则就设为 optional，因为 optional 会有些额外的检查

- 3、合并或者删减一些 OC 类，关于清理项目中没用到的类

- 4、删减没有被调用到或者已经废弃的方法

- 5、将不必须在 + load 方法中做的事情延迟到 + initialize 中或者放延迟到首屏渲染之后

- 6、高频次方法有些方法的单个耗时不高，但是在启动路径上会调用很多次的，这种累计起来的耗时也不低，比如读 Info.plist 里面的配置：

+ (NSString *)plistChannel{    
  	return [[[NSBundle mainBundle] infoDictionary] objectForKey:@"CHANNEL_NAME"];
  }
  

修改的方式很简单，加一层内存缓存即可，这种问题在 TimeProfiler 里时间段选长一些往往就能看出来。

#### 2、APP Launch 检测

![image-20210312173338839](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210312173338839.png)

![image-20210413111434697](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210413111434697.png)

![image-20210623152404407](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210623152404407.png)

![image-20210623155929385](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210623155929385.png)

![image-20210623154833028](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210623154833028.png)

### 一、阿里云sdk初始化 



### 二、flutter初始化 

[FlutterBoostPlugin.sharedInstance startFlutterWithPlatform:[RTFlutterRouter shareFlutterRouter]

  onStart:^(FlutterEngine *engine) {}]; 把flutter初始化延后至首屏渲染之后



### 三、其他+load方法

1、[self configProgressView];

 [[FRWLocalizableManager sharedInstance] localizabledImageNamed:imgName inBundle:bundle] 先从缓存取，缓存没有从本地取，然后把图片写入缓存，80ms，已没有用移除。

2、[HVSRCLiveChatRoomUtils initializeRongCloudIM];  11ms

3、[FRRWebEngine startEngine]; 59ms

实现一个类，负责首屏渲染之后处理初始化

```objective-c
+(void)initLaunchService {
	//flutter
  [FlutterBoostPlugin.sharedInstance startFlutterWithPlatform:[RTFlutterRouter shareFlutterRouter] onStart:^(FlutterEngine *engine) {}];
  //融云
  [HVSRCLiveChatRoomUtils initializeRongCloudIM];
  //webView
  [FRRWebEngine startEngine];
}
```



### 四、移除部分无用代码和类



#### 其他：

 之前直觉的就把第三方的初始化放到了 didFinishLaunchingWithOptions 方法里，我专门建了一个类来负责启动事件，区分开可以放在首屏显示之后再初始化的第三方库，以后再引入的时候就会判断一下。

```
/**
* 这个类负责所有原来didFinishLaunchingWithOptions和+load中 延迟事件的加载，根据需要减少 didFinishLaunchingWithOptions 里耗时的操作.
* 第一类: 比如日志 / 统计等需要第一时间启动的, 仍然放在 didFinishLaunchingWithOptions 中.
* 第二类: 部分第三方库的启动、数据处理等业务，不需要放在didFinishLaunchingWithOptions中，延后至首屏显示之后
*/
```

图片
启动难免会用到很多图，有没有办法优化图片加载的耗时呢？
用 Asset 管理图片而不是直接放在 bundle 里。Asset 会在编译期做优化，让加载的时候更快，此外在 Asset 中加载图片是要比 Bundle 快的，因为 UIImage imageNamed 要遍历 Bundle 才能找到图。加载 Asset 中图的耗时主要在在第一次张图，因为要建立索引，可以通过把启动的图放到一个小的 Asset 里来减少这部分耗时。

每次创建 UIImage 都需要 IO，在首帧渲染的时候会解码。所以可以通过提前子线程预加载（创建 UIImage）来优化这部分耗时。

有的启动只有到了比较晚的阶段“RootWindow 创建”和“首帧渲染”才会用到图片，**所以可以在启动的早期开预加载的子线程启动任务**。



#### 二进制重排

既然启动的路径上会触发大量的 Page In，那么有没有什么办法优化呢？

启动具有局部性特征，即只有少部分函数在启动的时候用到，这些函数在中的分布是零散的，所以 Page In 读入的数据利用率并不高。如果我们可以把启动用到的函数排列到二进制的连续区间，那么就可以减少 Page In 的次数，从而优化启动时间：

以下图为例，方法 1 和方法 3 是启动的时候用到的，为了执行对应的代码，就需要两次 Page In。假如我们把方法 1 和 3 排列到一起，那么只需要一次 Page In，从而提升启动速度。

![image-20210506165054800](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210506165054800.png)



链接器 ld 有个参数-order_file 支持按照符号的方式排列二进制。获取启动时候用到的符号主流有两种方式：

- 静态扫描获取 +load 和 C++静态初始化，hook objc_msgSend 获取 Objective C 符号。

- LLVM 函数插桩，灰度统计启动路径符号，用大多数用户的符号生成 order_file。

  

Facebook 的 LLVM 函数插桩是针对 order_file 定制，并且代码也是他们自己给 LLVM 开发的，目前已经合并到 LLVM 主分支了。

![image-20210506165105863](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210506165105863.png)

Facebook 的方案更精细化，生成的 order_file 是最优解，但是工程量很大。抖音的方案**不需要源码编译**，**不需要对现有编译环境和流程改造**，侵入性最小，缺点就是只能覆盖 90%左右的符号。

\- 灰度是任何优化都要利用好的一个阶段，因为很多新的优化方案存在不确定性，需要先在灰度上验证。

### 非常规方案

#### 动态库懒加载

最开始我们提到可以通过删代码的方式来减少代码量，那么有没有什么不减少代码总量，就可以减少启动时候要加载代码数量的方式呢？

- 答案就是动态库懒加载。

什么是懒加载的动态库呢？正常动态库都是会被主二进制直接或者间接链接的，那么这些动态库会在启动的时候加载。**如果只打包进 App，不参与链接，那么启动的时候就不会自动加载，在运行时需要用到动态库里面的内容的时候，再手动懒加载**。

懒加载动态库需要在编译期和运行时都进行改造，编译期的架构：

![img](https://static001.infoq.cn/resource/image/53/a2/53277be65a8882bfb437aef8fba065a2.png)



像 A.framework 等动态库是懒加载的，因为并没有参与主二进制的直接 or 间接链接。动态库之间一定会有一些共同的依赖，把这些依赖打包成 Shared.framework 解决公共依赖的问题。

运行时通过`-[NSBundle load]`来加载，本质上调用的是底层的 `dlopen`。那么什么时候触发动态库手动加载呢？

动态库可以分成两种：业务和功能。**业务就是 UI 的入口，可以把动态库加载的逻辑收敛到路由内部，这样外部其实并不知道动态库是懒加载的，也能更好地容错**。功能库（比如上图的 QR.framework）会有些不一样，因为没有 UI 等入口，需要功能库自己维护 Wrapper：

- ![image-20210506165131007](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210506165131007.png)App 对 Wrapper 直接依赖，这样外部并不知道这个动态库是懒加载的
- Wrapper 内部封装了动态调用逻辑，动态调用指的是通过 dlsym 等方式调用

动态库懒加载除了启动加载的代码减少，还能长期防止业务增加代码引起启动劣化，因为业务的初始化在第一次访问的时候完成的。

这个方案还有其他优点，比如动态库化后本地编译时间会大幅度降低，对其他性能指标也有好处，缺点是会牺牲一定程度的包大小，但可以用段压缩等方式优化懒加载的动态库来打平这部分损耗。

#### Background Fetch

Background Fetch 可以隔一段时间把 App 在后台启动，对于时间敏感的 App（比如新闻）可以在后台刷新数据，这样能够提高 Feed 加载的速度，进而提升用户体验。

那么，这种类似“后台保活”的机制，为什么能提高启动速度呢？我们来看一个典型的 case：

![image-20210506165219913](/Users/xushuanghui/Library/Application Support/typora-user-images/image-20210506165219913.png)

1. 系统在后台启动 App
2. 时间长因为内存等原因，后台的 App 被 kill 了
3. 这时候用户立刻启动 App，那么这次启动就是一次**热启动**，因为缓存还在
4. 又一次系统在后台启动 App
5. 这次用户在 App 在后台的时候点了 App，那么这次启动就是一次**后台回前台**，因为 App 仍然活着



通过这两个典型的场景，可以看出来为什么 Background Fetch 能提高启动速度了：

- **提高热启动在冷启动的占比**
- **后台启动回前台被定义为启动，因为用户的角度来说这就是一次启动**

后台启动有一些要注意的点，**比如日活，广告，甚至是 AB 进组逻辑都会受影响**，需要做不少适配。往往需要启动器来支撑，因为正常启动在 didFinishLaunch 执行的任务，在后台启动的时候需要延迟到第一次回前台的时候再执行。

资料：

- 1、去除没有用到的资源： https://github.com/tinymind/LSUnusedResources
- 2、利用AppCode：https://www.jetbrains.com/objc/ 检测未使用的代码：菜单栏 -> Code -> Inspect Code
- 3、可借助第三方工具解析LinkMap文件： https://github.com/huanxsd/LinkMap 生成LinkMap文件，可以查看可执行文件的具体组成



