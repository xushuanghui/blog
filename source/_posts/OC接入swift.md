---
title: OC接入swift
date: 2020-03-04 17:20:14
tags: iOS
---

# 工程配置

## 桥接文件

当我们在Swift项目（OC项目）中首次添加OC文件（Swift文件）时，Xcode会自动弹出桥接文件创建的提示，

这里我们可以选择Xcode自动创建或者手动创建，创建完成后系统会自动生成2个桥接文件,

$(SWIFT_MODULE_NAME)-Bridging-Header.h，这个头文件直接在目录中

$(SWIFT_MODULE_NAME)-Swift.h在目录中不可见.

如果我们需要手动创建的也是可以的，只要保证文件名与Build Setting中path一直即可，不过推荐使用系统默认的创建.

## 头文件导入

一般混编项目中常用的头文件有-Bridging-Header.h，-Swift.h，-pch等，在OC与swift混编时，如果Swift需要引用OC的类，那么需要再-Bridging-Header.h中引入OC头文件，

如果OC类需要引用Swift类时，需要在OC类中import $(SWIFT_MODULE_NAME)-Swift.h

导入后OC类即可使用项目中所有继承于NSObject的Swift类，这里需要注意的时，未继承NSObject的Swift类无法被OC调用.

这里点进$(SWIFT_MODULE_NAME)-Swift.h看一看，除了顶部一大堆宏定义宏方法外，Xcode还帮我们把所有继承自NSObject的Swift类编译成OC的接口，所以OC才能正常调用Swift类的属性和方法.

# cocoaPod

先说结论，现在（cocoaPod 1.5之后）集成OC和Swift库非常简单，无需考虑use_framework，static framework等等问题，仅需要把要导入的库名添加到podfile文件中即可，但是，初次导入后pod可能会遇到一些警告.

```
[!] The `Project [Debug]` target overrides the `SWIFT_INCLUDE_PATHS` build setting defined in `Pods/Target Support Files/Pods-Project/Pods-Project.debug.xcconfig'. This can lead to problems with the CocoaPods installation    - Use the `$(inherited)` flag, or    - Remove the build settings from the target.[!] The `Project [Release]` target overrides the `SWIFT_INCLUDE_PATHS` build setting defined in `Pods/Target Support Files/Pods-Project/Pods-Project.release.xcconfig'. This can lead to problems with the CocoaPods installation    - Use the `$(inherited)` flag, or    - Remove the build settings from the target.
```

可以看到，Pods-DadaStaff.debug.xcconfig文件中的SWIFT_INCLUDE_PATHS被项目tage的buildSeetingt覆盖了，如果不解决的话，会导致Swift三方库无法正常编译成module.

找到target中的Swift Complier - Search Path，在Import Paths项中添加$(inherited)

重新pod，消除警告后，即可使用Swift三方库.

在cocoaPod更新到1.5之前，OC与Swift混编比较头疼的一个问题就是pod库的集成，历史原因这里就不赘述了语法混编

<!--more-->

## OC调用Swift

简单说下OC调用Swift需要注意的点：

1.import $(SWIFT_MODULE_NAME)-Swift.h

```
#import "OCAndSwift-Swift.h"
```

2.调用Swift类时，Swift类必须继承自NSObject

```
class SwiftClass: NSObject { var claseName = "SwiftClass"}
```

3.调用Swift类的方法，属性时，Swift类的方法属性都必须加上@objc关键字

```
@objc var retrainConfirmBlock: (() -> Void)?@objc convenience init()
```

4.如果使用KVC，KVO等监听，需要对监听的属性加上dynamic关键字，因为Swift并不是动态派发的语言，dynamic关键字会让编译器进行动态分发而不是静态分发.

```
@objc dynamic var orderNumber: NSNumber?
```

## Swift调用OC

Swift调用OC需要注意的点：

1.$(SWIFT_MODULE_NAME)-Bridging-Header.h引入OC类头文件.

2.OC类的相关属性及方法在Swift类中都可以直接调用，只需要注意Swift语法即可.

```
UIView.animate(withDuration: TimeInterval(time), delay: 0, options: UIView.AnimationOptions.curveLinear, animations: {// animate }) {(finish) in//finish}
```

3.OC类的多数基本类型如NSArray，NSDictionay，NSError等对象(Object) 都可以通过 as? 和Swift中的基础类型Array，Dictionary，Error等值类型互换.

```
let contentAry = info["content"] as? [String]titleLabel.text = title as? String ?? ""if let failedWith = error as? Error {}
```

4.OC宏的使用，Swift使用OC的宏，只能直接使用非常简单的值类型宏，对于复杂的宏方法则需要自己重写一个全局方法替代，全局静态常量可以直接使用.

```
#define kExpireTime 60  // √#define sharedAppDelegate ((AppDelegate *)([UIApplication sharedApplication].delegate)) // X
```

<!--more-->

## Swift特性

### Optional

1.简单介绍一下Optional-可选类型，Optional实际上是一个枚举类型：

```
public enum Optional<Wrapped> : ExpressibleByNilLiteral {case nonecase some(Wrapped)...}
```

表示一个数据可能有值也可能为空即nil，类型可以用Optional或者T?（常用）表示.

2.如果一个可选类型有值，那么这个值也是被包装起来的，例如optional(“test”)，如果需要取出值就需要解包

解包方式有 if let 以及！，！表示强解，一般在确定值不为空的时候使用，否则会造成崩溃.

```
if let text = OptionalText {print(text)}let text = OptionalText!
```

3.在与OC混编过程中，因为OC没有可选类型的概念，需要注意可选类型带来的问题.例如

```
@objc init(offset: CGFloat? = nil,  text: String? = "") {super.init(frame: .zero)self.type = typeself.offset = offset}
```

这个方法如果想给OC调用则会报错

```
Method cannot be marked @objc because the type of the parameter 1 cannot be represented in Objective-C
```

提示第一个参数无法用OC表示，因为在OC中CGFloat是值类型，不能为nil，而对于OC中的对象类型NSString （String自动转换）则可以接受.

另外，Swift使用OC定义类型时也要注意是否为可空类型

```
@property (nonatomic, strong) NSNumber *number1;@property (nonatomic, strong, nullable) NSNumber *number2;
```

带nullable关键字的属性可以被Swift转为可选类型，但是不带nullable的属性则默认被！强解，这里需要注意使用OC网络数据等，如果不做判断可以会导致崩溃.

## 泛型

泛型是Swift中的一大利器，许多swift的标准库都是通过泛型代码构建的，例如数组和字典都是泛型集，在系统库或者一些知名三方库中，随处可见的Element，<T>，Where等等占位符或关键字，都是使用泛型在构建代码，引用一段Swift文档对于泛型的描述.

> *Generic code* enables you to write flexible, reusable functions and types that can work with any type, subject to requirements that you define. You can write code that avoids duplication and expresses its intent in a clear, abstracted manner.
>
> Generics are one of the most powerful features of Swift, and much of the Swift standard library is built with generic code. In fact, you’ve been using generics throughout the *Language Guide*, even if you didn’t realize it. For example, Swift’s `Array` and `Dictionary` types are both generic collections. You can create an array that holds `Int` values, or an array that holds `String` values, or indeed an array for any other type that can be created in Swift. Similarly, you can create a dictionary to store values of any specified type, and there are no limitations on what that type can be.

概括泛型的关键字：自定义，任意类型，灵活，重用，清晰，抽象.篇幅关系，Swift泛型的应用以后再详细总结.

在Xcode7之后，为了“迎合”Swift，苹果对OC也做了许多提升，其中包括Nullability，轻量级泛型和__kindof等等，nullable上文我们提到过，可以在Swift使用OC属性时自动编译为Optional类型，那么在在泛型的混编上应该注意什么呢？

#### 1.OC种泛型的使用

在OC当中对泛型比较常用的方式，例如对数组元素类型的定义

```
NSArray<NSString *> *messageArray
```

一方面可以限制集合类型，一方面可以直接使用点语法，并且在添加，遍历时会有类型提示.

#### 2.OC泛型的定义

OC中2种自定义泛型定义：

```
 __covariant:协变, 子类转父类 __contravariant:逆变 父类转子类
```

那么什么是协变，逆变呢，我们用一个例子来说明

```
@interface Generic<ObjectType> : NSObject@end@interface ViewController ()@property (nonatomic, strong) Generic * generic;@property (nonatomic, strong) Generic<NSString*> * stringGeneric;@property (nonatomic, strong) Generic<NSMutableString*> * mutableGeneric;@end@implementation ViewController- (void)viewDidLoad {    [super viewDidLoad];    self.generic = self.stringGeneric;    self.generic = self.mutableGeneric;    self.stringGeneric = self.generic;    self.stringGeneric = self.mutableGeneric; // Incompatible pointer types assigning to 'Generic<NSString*> *' from 'Generic<NSMutableString*> *'    self.mutableGeneric = self.generic;    self.mutableGeneric = self.stringGeneric; // Incompatible pointer types assigning to 'Generic<NSMutableString*> *' from 'Generic<NSString*> *'}@end
```

从代码警告我们可以看出：

1. 不指定泛型类型的对象generic可以和任意泛型类型对象转换.
2. 指定了泛型类型的对象generic不能和不同泛型类型对象转换(只是警告).

如果你需要主动控制转换关系则需要添加 __covariant 或 __contravariant，效果如下：

```
@interface Generic<__covariant ObjectType> : NSObjectself.stringGeneric = self.mutableGeneric; // 子类转父类self.mutableGeneric = self.stringGeneric; // Incompatible pointer types assigning to 'Generic<NSMutableString*> *' from 'Generic<NSString*> *'
@interface Generic<__contravariant ObjectType> : NSObjectself.stringGeneric = self.mutableGeneric; // Incompatible pointer types assigning to 'Generic<NSString*> *' from 'Generic<NSMutableString*> *'self.mutableGeneric = self.stringGeneric; // 父类转子类
```

#### 3.OC泛型在Swift中的使用

1.OC中定义的泛型，在Swift中使用时，必须指定泛型类型

```
let g = Generic<AnyObject>()
```

2.使用泛型类型的子类时，如果想要使用父类的方法或属性，建议先使用as转换类型，获取到方法列表或属性列表后，再把as去掉，因为子类无法直接获取到父类的方法属性列表，有点坑.

```
let gChild = GenericChild<AnyObject>()(gChild as GenericChild<AnyObject>).genericFunc()gChild.genericFunc()
```

3.swift并不吃OC泛型的协变，逆变这一套，它只有一个基本原则：类型固定.只要你在初始化时说明了对象的泛型类型，那么他不管怎么转换类型，指针赋值，始终只能转换为他自己的类型.

```
let gString = Generic<NSString>()var gMutableString = Generic<NSMutableString>()gMutableString = gString as! Generic<NSMutableString>
```

#### 4.Swift泛型在OC中的使用

不行（可以查看一下$(SWIFT_MODULE_NAME)-Swift.h中是没有将Swift泛型类编译成OC类的哦）.

### closure

1.OC中的blcok与Swift中closure都是经常使用的代码类型，Swift调用OC block会自动转化为closure类型.

```
void (^completionBlock)(NSData *, NSError *) = ^(NSData *data,NSError *error) {/* ... */}
let completionBlock: (NSData, NSError) -> Void = {data, errorin /* ... */}
```

2.Swift中closure和function是同一种类型，所以可以将Swift的方法名作为block参数传递给OC.

```
let completionBlock: (NSData, NSError) -> Void = {data, error    in /* ... */}func completionBlockFunc(_ data:NSData, _ error:NSError) -> Void {        /* ... */    }
```

3.block默认截获变量，如果需要截获引用，则需要加上__block关键字，closure默认截获变量的指针，也就是说closure默认就添加了__block关键字，但是在变量没有改变的情况下，closure会做优化只持有变量的值而不是指针.

> As an optimization, Swift may instead capture and store a copy of a value if that value is not mutated by or outside a closure.

4.循环引用及Weak-Strong Dance

OC中未避免循环引用，会对持有对象做weak处理，Swift中修饰对象为弱引用有2种方式.

```
// 对此对象进行弱引用，此对象引用计数不会增加，修饰的对象为可选类型weak var object// 对此对象进行弱引用，此对象引用计数不会增加，修饰的对象不能为nilunowned var objectself.closure = {    [unowned self] in    // self 不能为空，否则会造成崩溃    self.viewDidLoad() } self.closure = {     [weak self] in     // self 是可选类型，此处可以解包self     self?.viewDidLoad()}
```

### Enum

OC中NS_ENUM枚举定义和Swift中的Enum默认是相互转化的，但是秉着简洁清晰的代码原则，Swift把OC中枚举类型名称给yan割了，例如

```
typedef NS_ENUM(NSInteger, UITableViewCellStyle) { UITableViewCellStyleDefault, UITableViewCellStyleValue1, UITableViewCellStyleValue2, UITableViewCellStyleSubtitle};
```

等价于

```
enum UITableViewCellStyle: Int {    case Default    case Value1    case Value2    case Subtitle}
```

Swift用点语法来使用枚举：（许多系统的枚举类也被改了，如果不知道怎么写，可以把OC的系统枚举写上，会有错误提示帮你更正）

```
let cellStyle: UITableViewCellStyle = .Default
```

Swift中的枚举比起OC来强大了很多，OC中的枚举只能定义类型，但是Swift中的枚举可以添加方法和属性，类似于Struct，可以实现很多自定义的功能.

```
enum NDDErrCode: Int, CustomStringConvertible {    case success = 0    case fail    case noData    case noMoreData    var description: String {        switch self {        case .success:            return "成功"        case .fail:            return "失败"        case .noData:            return "没有数据"        case .noMoreData:            return "没有更多数据"        }    }}
```

# 其他问题

#### Protocol

OC中Protocol更倾向于代理功能，由Protocol协议，代理对象delegate以及委托者组成，主要用于事件回调，页面传值等功能；

Swift中的Protocol更多的是面向协议编程，指抽象出不同类的相同行为（方法），特点（属性）等等，实现模块化解耦.还可以通过extension，实现协议的默认方法（OC不行），也无需生命代理对象等.

Swift调用OC的Protocol和调用方法一样，OC调用Swift协议时，需要注意创建Protocol时，@objc，optional等关键字和协议类型（any，class，anyObject）等协议遵循限制.

#### RAC and RxSwift

Swift可以使用RAC（不建议，毕竟有正宫娘娘RxSwift），除了语法外，需要注意RAC大部分类也是OC泛型，需要类型转换以获取父子类方法及属性列表.

OC无法使用RxSwift.

#### 单例

OC单例

```
+ (instancetype)sharedInstance {    static Singleton *shared = nil;    static dispatch_once_t onceToken;    dispatch_once(&onceToken, ^{        shared = [[Singleton alloc] init];    });    return _shared;}
```

Swift单例

```
static let sharedInstance = Singleton()
```

#### json转模型

OC中常用json转对象工具MJExtension，Swift中常用工具较多，如HandyJson，ObjectMapper等等，另外Swift4引入的Codable协议也是非常方便的.

#### OC变参方法

OC的变参方法无法直接在Swift中被调用，需要对变参方法进行修改，使用va_list对变参进行拼接.

原方法

```
+ (void)stringParams:(NSString *)params,...;
```

重写方法

```
+ (void) stringParams:(NSString *)params args:(va_list)args {    va_list args_copy;    __va_copy(args_copy,args);    NSMutableString* format = [NSMutableString stringWithString:@""];    while (va_arg(args, NSString*))    {        [format appendString:@"%@,"];    }    va_end(args);    if(format.length>0)        [format deleteCharactersInRange:NSMakeRange(format.length-1,1)];    NSString* newFormat = [NSString stringWithFormat:@"%@",format];    NSString * result = [[NSString alloc]initWithFormat:newFormat arguments:args_copy];    va_end(args_copy);    NSLog(@"%@", result);}
```

swift调用

```
let args: [CVarArg] = ["i'm", " funny"]        withVaList(args) {                (pointer: CVaListPointer) in            return ViewController.stringParams("%@,%@", args: pointer)                }
```