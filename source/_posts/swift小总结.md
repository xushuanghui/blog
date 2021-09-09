---
title: swift小总结
date: 2020-04-19 14:24:40
tags:
---

### 1、[混编方式](https://developer.apple.com/documentation/swift/imported_c_and_objective-c_apis/importing_swift_into_objective-c)

1、SWIFT_OBJC_BRIDGING_HEADER方式适用简单项目

- 简单的项目可以直接使用`projectName--Bridging-Header.h`文件的方式，将需要暴露给swift的oc类包含进去。

- Swift 访问 Objective-C

  只需要在桥接文件中（Bridging-Header.h）中导入需要暴露给 Swift 模块的 Objective-C 类，即可在 Swift 中访问相应 Objective-C 的类和方法

- Objective-C 访问 Swift

  在 Objective-C 类中导入 `ProductName-Swift.h`，即可访问 Swift 中暴露给 Objective-C 的类和方法

2、含三方库，业务分库的较复杂的项目

- 纯swift构成的pod，使用时直接`@import module`就行

- oc和swift混合构成的pod

  - pod中oc类中使用swift类，需要通过`SWIFT_OBJC_INTERFACE_HEADER_NAME`头文件中先导入需要暴露(*`public`*)给oc使用的swift类, 然后oc中引用`#import "YourPodModule-Swift.h"`, `#import "YourPodModule-Swift.h"`无法直接在主工程引入

  - *`podfile`\*中配置\*`use_modular_headers!`*, 主工程通过`@import module`导入含swift的pod即可访问暴露给外部的swift类。iOS头文件引入变迁`#include->#import->pch->@import`, `@import`接解决了编译时间和引用泛滥的问题，建议项目中大量使用。

  - Swift 访问 Objective-C

    用 Swift Module 系统，需要用到的 Objective-C 类用 import xxx 进行引用，即可在 Swift 中访问相应的 Objective-C 的类和方法

  - Objective-C 访问 Swift

    在 Objective-C 类中导入 `ProductName-Swift.h`，即可访问 Swift 中暴露给 Objective-C 的类和方法

3、继承

- ```
  1、使是继承自NSObject的类，也需要显式添加@objc才能访问。在此之前是默认添加的。
  
  2、swift的类如果想通过动态性动态生成，比如`Class cls = NSClassFromString(clsName);`
  @objc(HSAShortVideoDetailViewController)
  ```

4、[@objc的一些规则](https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md)

- ```
  @objc protocol p {
      func name()
  }
  
  extension p {
      func bar() {
        print("bar")
      }
  }
  
  class C: NSObject, p {
      
     func name() {
         print("name\n")
      }
      
     @objc func aaa() {
          print("aaa")
      }
  }
  
  let c = C()
  
  print(c.responds(to: Selector("name")))  // true
  print(c.responds(to: Selector("bar")))   // false
  print(c.responds(to: Selector("aaa")))   // true
  
  
  ## @objc 并不改变访问级别，默认是Internal; 下面的例子中编译时不允许访问，运行时可访问
  # pod中定义的swift class
  class HSACleanScreenView: UIView {
      @objc func queryBlessInfo() {}
  }
  
  # 主工程
  if ([_cleanSreenView respondsToSelector:@selector(queryBlessInfo)]) {
      [_cleanSreenView performSelector:@selector(queryBlessInfo)];
  }
  ```

5、[避免@objc的滥用](https://www.jessesquires.com/blog/2016/06/04/avoiding-objc-in-swift/)

- swift中调用oc，c类型的函数需要传递block作为参数时。 作为参数时，的修饰生命周期： `@escaping & @nonescaping`

  ```
   // HVSLiveCleanScreenView
   let block: @convention(block) ()->() = {
            self.toSendWishMainPageAction(sender: sender)
        }
    NotificationCenter.default.post(name: NSNotification.Name(rawValue: "HSACallLoginNotification"), object: ["callback" : block])
    
    // 类型转换
    private func getAuthorizationData() -> [String: Any] {
        var data: [String: AnyObject] = [String: AnyObject]()
        data["callback"] = {
            print("Its NOT crashing")
        } as (@convention(block) ()->Void) as AnyObject
        return data
    }
    
    // swift调用带c函数作为参数的
    CGFloat myCFunction(CGFloat (callback)(CGFloat x, CGFloat y)) {
        return callback(1.1, 2.2);
    }
    let swiftCallback : @convention(c) (CGFloat, CGFloat) -> CGFloat = { (x, y) -> CGFloat in
        return x + y
    } 
    myCFunction( swiftCallback )
  ```

6、oc头文件中包含swift类时使用前行声明

- ```
  // MyObjcClass.h
   @class MySwiftClass;
   @protocol MySwiftProtocol;
  
   @interface MyObjcClass : NSObject
   - (MySwiftClass *)returnSwiftClassInstance;
   - (id <MySwiftProtocol>)returnInstanceAdoptingSwiftProtocol;
   // ...
   @end
  ```

7、 oc调用swift闭包

- ```
   @objc public var tapBlock: (() -> ())?
   
   typealias BlockType = (String, String) -> ()
   @objc public var tapBlock: BlockType?
  ```

8、NS_STRING_ENUM

尽量利用swift的优秀特性，比如一些有意义的key，直接用string，容易出错还无法进行类型检查

```
#objectiv-c

## .h文件
typedef NSString * THJStringDicKey NS_STRING_ENUM;
FOUNDATION_EXTERN THJStringDicKey const THJStringDicKeyTitle;
FOUNDATION_EXTERN THJStringDicKey const THJStringDicKeyBody;
FOUNDATION_EXTERN THJStringDicKey const THJStringDicKeyHeader;
## .m文件
THJStringDicKey const THJStringDicKeyTitle  = @"title";
THJStringDicKey const THJStringDicKeyBody   = @"body";
THJStringDicKey const THJStringDicKeyHeader = @"header";

#swift
let dic:  [THJStringDicKey:String] = [.title:"title"]
```

- [关于`@convention` swiftGG](https://www.codenong.com/jsa3e58ba1e6e8/)

### 枚举

- oc中使用swift枚举

  ```
  @objc public enum WishShareType: Int {
      case video
      case live
      
      func name() -> String {
          switch self {
          case .video:
              return "video"
          case .live:
              return "live"
          default:
              return "video"
          }
      }
  }
  
  // swift中WishShareTypeLive这样来使用
  ```

- swift中使用oc枚举

  ```
    typedef NS_ENUM(NSInteger, HSAShortVideoLeftMenuTapActionType) {
        HSAShortVideoLeftMenuActionHotMenu,   // 热门菜品
        HSAShortVideoLeftMenuActionCoupon,    // 优惠券
    };
  
    let type: HSAShortVideoLeftMenuTapActionType = .actionCoupon
    let type = HSAShortVideoLeftMenuTapActionType.actionCoupon
  ```

  注意：swfit使用`Enum.init(rawValue:)`生成oc类型的枚举，如果传入的值不在定义范围不会返回悔nil，执行时可能会出现无法预测的异常

### [协议](https://www.jessesquires.com/blog/2017/06/05/protocol-composition-in-swift-and-objc/)

- oc使用swift协议

  ```
  @objc protocol AlertViewProtocol {
    func submit(_ row: Int) //必须实现的协议
    @objc optional func cancel() //不必实现的协议
  }
  ```

- swift使用oc的协议

  ```
  @objc public var delegate: HVSLiveChatRoomViewDelegate?
  if delegate?.responds(to: #selector(HVSLiveChatRoomViewDelegate.showWishMainPage(_:))) ?? false {
            delegate!.showWishMainPage!(sendWishVc)
  }
  ```

### target-action

- swift2.2版本之前可以通过`Selector("functionToExecute:")`的方式来生成，但该方式无法在编译器发现方法是否被实现，会导致[unrecognized selector sent to instance ](https://learnappmaking.com/unrecognized-selector-sent-to-instance-swift-development/)错误。swift2.2后使用`#selector(functionToExecute(_:))`或`Selector(Target.functionToExecute)`, 之前的`Selector("")`方式被废弃

- `__FUNCTION__` =>`#function`

  ```
  print("%@", #function) // swift2.2之后替换为`#function`
  ```

### 关于`responds`

- ```
  ## oc中使用
  
  ## swift中判断
  func responds(to aSelector: Selector!) -> Bool
  
  ## 动态判断，不确定类型的前提下，如遵循协议的swift对象是否实现了协议内的方法
  if delegate?.responds(to: #selector(HVSLiveChatRoomViewDelegate.showWishMainPage(_:))) ?? false {
       delegate!.showWishMainPage!(sendWishVc)
  } 
  ```

### 循环引用

- block循环引用处理

  ```
  wishVc.animationBlock = { [weak self], [weak obj] animView in
       animView.show(superView: self!, belowView: self!.wishButton, replace: self?.runingWishAimationView)
   }
  ```

- weakproxy

  ```
  // NSTimer 还可以通过weakProxy来解循环引用
  timer = Timer.init(timeInterval: 1.0, target: FRWWeakProxy.init(target: self), selector: #selector(downTime), userInfo: nil, repeats: true)
       RunLoop.main.add(timer, forMode: .common)
       
  // wkWebView引起的循环引用因为协议的原因无法通过weakProxy来作转换
   open func add(_ scriptMessageHandler: WKScriptMessageHandler, name: String)
   
   let userContentController = WKUserContentController.init()
   userContentController.addUserScript(userScript)
   userContentController.add(self, name: _CardAppNative)
   
   // 解决办法
   1、 目前通过主动调用clean来主动解开循环
   func clean() {
       webView.configuration.userContentController.removeScriptMessageHandler(forName: _CardAppNative)
   }
   
   2、 如果是控制器可以通过*`viewWillDisappear:`*来控制解开循环的时机，或则通过自定义`WKScriptMessageHandler`来解决
  ```

### 第三方库

- masonry

  ```
    giftContentView.mas_makeConstraints { (make) in
        //  Fatal error: Unexpectedly found nil while implicitly unwrapping an Optional value
        //  make?.left.mas_offset()(contentBeginX)
        make?.left.offset()(contentBeginX)
        make?.right.equalTo()(self.contentView)?.offset()(-contentEndX)
        make?.top.mas_equalTo()(12)
        make?.height.mas_equalTo()(contentHeight)
    }
    
    oc中可以直接使用`mas_offset()`方式设置，但是swift中这里会导致crash，主要原因是因为swift不支持宏
    导致`#define mas_offset(...)          valueOffset(MASBoxValue((__VA_ARGS__)))`失效，实际返回nil;
    使用时尽量不适用带`mas_`的方法进行设置
  ```

- Reactive-C

  ```
  // 同样因为宏在swfit中无法使用，`RAC(TARGET, ...)`等宏无法使用，需要直接调用对应代码
  model.rac_values(forKeyPath: "buttonStatus", observer: self).take(untilReplacement: self.rac_signal(for: #selector(self.prepareForReuse))).subscribeNext { [weak self] (obj) in
      self?.changeActionButtonStyle()
  }
  ```

### 通知

- 对于oc中通知的key的使用：宏定义和全部变量定义

- 通知使用oc中早先定义的key时``

  ```
  ## 宏定义
  #define kHSANotificateToPlayInlineWishAnimationKey @"notificateToPlayInlineWishAnimation"
  NSNotification.Name(rawValue: kHSANotificateToPlayInlineWishAnimationKey)
  
  ## 全局变量
  ```

## 2、其他问题

  ### 2.1 Framework targets 不支持 Bridging-Header

  通常来讲混编的时候需要在工程中创建 Swift 文件时候，Xcode 会问询是否创建 Bridging-Header 文件，点击是，系统会帮你创建一个 Bridging-Header，你可以将需要引用的 Objective-C 模块的头文件放在里面，然后你可以在 Swift 模块用 Objective-C 的类。但是编译器是不允许在 Framework 中创建 Bridging-header，因此在二/三方库中，我们不能使用桥接文件的方式进行混编 Objective-C 代码的引用，需要用 Swift Module 进行模块间的引用。

  ### 2.2 模块引用

  引用其他 Objective-C 二方库需要增加命名空间（Namespace），否则会报错找不到文件 Swift 的命名空间是以模块划分的，一个模块表示一个命名空间。开发时，默认添加到主 target 的内容是同处于同一个命名空间的；如果用 Cocoapods 导入的第三方库，是以一个单独的 target 存在，不会存在命名冲突。但如果以源码的方式导入工程，很可能发生命名冲突，所以为了安全起见，第三方库都会使用命名空间这种方式来防止冲突。

  ### 2.3 C++ 混编

  Objective-C 是 C++ 的超集，就如同 Objective-C 是 C 的超集，在OS X 上同时被 GCC 和 Clang 支持编译，.mm 是 Objective-C++ 的默认后缀名，Xcode 的编译器可以识别。在.mm 文件中，Objective-C 代码和 C++ 代码都可以正常编译运行。在消息业务模块中中引用了 WCDB 这个 Objective-C++ 的库，因此在引用的时候要将引用到的 WCDB.h 头文件中的类文件的 .h 改成 .mm。

  ### 2.4 链接错误

  我们将上述工作做完后引入到宿主工程中，进行编译的时候会出现链接错误，不要担心，那是因为宿主工程中缺少 Swift 的某些系统库，在宿主工程中建立一个 Swift 文件方可解决。

  ### 2.5 Swift 调用 Objective-C

  将 Swift 模块文件中，用import xxx 的形式进行模块的引用，包括 Objective-C 的二/三方库

  ### 2.6 Objective-C 调用 Swift

  - Swift 类中将需要暴露给 Objective-C 模块引用的类，用 public 申明
  - Swift 类中需要暴露给 Objective-C 的方法要用关键字 @objc
  - 在 Objective-C 类中引用 ProductName-Swift.h 头文件即可引用暴露给 Objective-C 的 Swift 的类和方法