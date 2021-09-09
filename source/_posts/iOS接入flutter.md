---
title: iOS接入flutter
date: 2020-09-05 15:44:02
Tags: ios
---

### 1. flutter module 创建与配置

#### 1.1 下载 flutter 基础包

从 github 下载代码 `git clone -b beta https//github.com/flutter/flutter.git`

但是 github 可能会很慢，可以从码云上下 `git clone https://gitee.com/mirrors/Flutter.git`

#### 1.2 配置 flutter 环境变量

如果使用的 bash

```
第一步：
cd ~           // 在终端进入用户目录，一般打开终端默认就是

第二步：
open .bash_profile    // 打开 bash 配置文件
// 另 如果 .bash_profile文件不存在需先创建再打开，具体如下：
// touch .bash_profile
// open .bash_profile

第三步：
# for flutter
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
export PATH=/Users/hubery/dev_supports/flutter/bin:$PATH

第四步：
source .bash_profile      // 执行文件 使命令生效
```

如果使用的 zsh

```
第一步：
cd ~           // 在终端进入用户目录，一般打开终端默认就是

第二步：
open .zshrc    // 打开 zsh 配置文件

第三步：
# for flutter
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
export PATH=/Users/hubery/dev_supports/flutter/bin:$PATH

第四步：
source .zshrc      // 执行文件 使命令生效
```

> 其中 `/Users/hubery/dev_supports/flutter` 为 1.1 中下载下来的 flutter 路径

然后在终端中输入 `flutter` 验证环境变量是否配置成功，出现如下类似信息时表示配置是OK的

#### 1.3 flutter module 创建

可以使用 Android Studio 创建 flutter module

或者在目标目录下使用命令创建 `flutter create -t module flutter_demo_module_ios`

创建的过程需要梯子从外网获取 flutter 相关资源。 这里推荐使用 Android Studio，操作方便，flutter 相关的插件支持的也挺好

#### 1.4 Android Studio 上 flutter 项目配置

- 使用 Android Studio 打开刚才创建的 module, 在 Project 栏可以看到项目目录结构

  Android Studio –> Preferences –> Plugins –> Marketplace 搜索 `Dart` 和 `Flutter` 插件并下载

- 配置 `Dart SDK path`, 设置入口为： Android Studio –> Preferences –> Languages & Frameworks –> Dart

  配置完成后，Android Studio 工具栏就可以看到选择 devices 的选项啦，如果没出现可以重启 Android Studio

#### 1.5 检查 flutter 运行环境是否可用

执行命令 `flutter doctor`

#### 1.6 如果flutter中插件安装，可在 pubspec.yaml 文件中添加需要的插件

- 安装: `flutter pub get`
- 更新: `flutter pub upgrade`

#### 1.7 执行编译

- debug: `flutter build ios --debug --no-codesign`
- release: `flutter build ios --release --no-codesign`

### 2. iOS 引入 flutter module

#### 2.1 创建 iOS 项目 & Pod Init

如何创建 iOS项目以及 CocoaPods 的使用，此处就省略不表了 。。。

如需了解 CocoaPods，请戳传送门：[CocoaPods 攻略](https://www.jianshu.com/p/6d51362b7e64)

#### 2.2 使用 Cocoapods 引入 flutter module

- 先执行编译该 `flutter_demo_module_ios` 项目, 终端执行 `flutter build ios --debug --no-codesign`

- iOS 项目中 Podfile 文件中添加如下命令:

  ```
    # Flutter
    flutter_application_path = '../flutter_demo_module_ios/'
    load File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')
  	
  	
    target 'FlutterDemoIOSApp' do
      # Comment the next line if you don't want to use dynamic frameworks
      use_frameworks!
  	
      # Flutter
      install_all_flutter_pods(flutter_application_path)
  	
  	
    end
  ```

  然后执行 `pod install` 即可引入，在 pod 可以看到新增3个flutter相关的 framework

  <!--more-->

### 3. iOS 调用 flutter 页面

#### 3.1 多引擎调用

多引擎调用就是每打开一个页面就实例化一个 `FlutterViewController`

iOS 中的实现:

```
@objc
    private func onOpenPageA() {
        let flutterVC = FlutterViewController()
        flutterVC.setInitialRoute("one")
        self.navigationController?.pushViewController(flutterVC, animated: true)
    }
    
    @objc
    private func onOpenPageB() {
        let flutterVC = FlutterViewController()
        flutterVC.setInitialRoute("two")
        self.navigationController?.pushViewController(flutterVC, animated: true)
    }
```

flutter 中的实现:

```
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_demo_module_ios/pages/first_page.dart';
import 'package:flutter_demo_module_ios/pages/second_page.dart';

void main() => runApp(MyApp(pageIndex: window.defaultRouteName));

class MyApp extends StatelessWidget {
  final String pageIndex;

  const MyApp({Key key, this.pageIndex}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: rootPage(pageIndex),
    );
  }

  rootPage(String pageIndex) {
    switch (pageIndex) {
      case 'one':
        return FirstPage();
      case 'two':
        return SecondPage();
    }
  }

}
```

在实例化 `FlutterViewController` 时为 flutter 指定对应的页面，flutter 根据传入的 key 创建对应的页面

起初 flutter 官方提供的是多引擎调用方式，简单来说是每打开一个页面创建一个 `FlutterViewController`, 每个 `FlutterViewController` 会对应一个 `FlutterEngine`, 而在页面退出时内存却没有完全释放，这就导致每打开一次页面内存会逐步增长

#### 3.2 单引擎调用

所谓的单引擎就是全局只使用一个 `FlutterEngine`，再绑定唯一的 `FlutterViewController`，这样多引擎的内存问题也就基本得到了解决，这也是目前官方比较推荐方式。

由于全局只有 `FlutterViewController` ，那么在打开 flutter 页面前，需要通过消息通知 flutter 替换当前的 main page

iOS 中的实现:

```
// 在 AppDelegate 中创建全局 FlutterEngine，并启动
// 之所以在APP启动时就启动，是因为 FlutterEngine 启动时会有短暂的延迟，放在打开页面之前启动会明显感觉到卡顿

var window: UIWindow?
var flutterEngine: FlutterEngine?
    

 func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        
        if #available(iOS 13.0, *) {
            // SceneDelegate 中处理
        } else {
            window = UIWindow(frame: UIScreen.main.bounds)
            window?.backgroundColor = .white
            let nav = UINavigationController(rootViewController: ViewController())
            window?.rootViewController = nav
            window?.makeKeyAndVisible()
        }

        
        self.flutterEngine = FlutterEngine(name: "flutter-demo", project: nil)
        self.flutterEngine?.run()
        
        return true
    }
class ViewController: UIViewController {

    private var flutterVC: FlutterViewController!
    private var msgChannel: FlutterBasicMessageChannel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        self.view.addSubview(button(frame: CGRect(x: 60, y: 150, width: 260, height: 40), title: "Page A", selector: #selector(onOpenPageA)))
        self.view.addSubview(button(frame: CGRect(x: 60, y: 250, width: 260, height: 40), title: "Page B", selector: #selector(onOpenPageB)))
        
        let flutterEngine = (UIApplication.shared.delegate as? AppDelegate)?.flutterEngine
        self.flutterVC = FlutterViewController(engine: flutterEngine!, nibName: nil, bundle: nil)
        self.msgChannel = FlutterBasicMessageChannel(name: "message-channel", binaryMessenger: flutterVC as! FlutterBinaryMessenger)
        
        self.msgChannel.setMessageHandler({ (message, reply) in
            print("message: \(String(describing: message))")
            print("reply: \(String(describing: reply))")
        })
    }

    private func button(frame: CGRect, title: String, selector: Selector) -> UIButton {
        let btn = UIButton()
        btn.frame = frame
        btn.backgroundColor = .red
        btn.setTitle(title, for: .normal)
        btn.setTitleColor(.white, for: .normal)
        btn.addTarget(self, action: selector, for: .touchUpInside)
        return btn
    }
    
    @objc
    private func onOpenPageA() {
        self.pushFlutterPage(pageName: "one")
    }
    
    @objc
    private func onOpenPageB() {
        self.pushFlutterPage(pageName: "two")
    }

    private func pushFlutterPage(pageName: String) {
    
        // 创建信息通道，name 需与 flutter 中保持一致
        let methodChannel = FlutterMethodChannel(name: "method-channel", binaryMessenger: flutterVC as! FlutterBinaryMessenger)
        // 通知 flutter 将要展示的页面
        methodChannel.invokeMethod(pageName, arguments: nil)
        self.navigationController?.pushViewController(flutterVC, animated: true)
        
        // 监听 flutter 页面内的消息
        methodChannel.setMethodCallHandler { (call, result) in
            let action = call.method
            switch action {
            case "back":
                self.flutterVC.navigationController?.popViewController(animated: true)
                
            case "changeBackgroundColor":
                let colors: [UIColor] = [.black, .gray, .yellow, .orange, .brown]
                let idx = arc4random()%4
                self.view.backgroundColor = colors[Int(idx)]
                
            default:
                break
            }
            
        }
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        // 向 flutter 发送信息
        self.msgChannel?.sendMessage(NSDate().description)
    }
    
}
```

flutter 中的实现

```
import 'dart:ui';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:flutter_demo_module_ios/pages/first_page.dart';
import 'package:flutter_demo_module_ios/pages/second_page.dart';

void main() => runApp(MyApp());

class MyApp extends StatefulWidget {
  _MyApp createState() => _MyApp();
}

class _MyApp extends State<MyApp> {
  String pageIndex = 'one';
  
  // 注册消息通道
  final MethodChannel _oneChannel = MethodChannel('method-channel');
  final BasicMessageChannel _msgChannel = BasicMessageChannel('message-channel', StandardMessageCodec());

  @override
  void initState() {
    super.initState();

    // 监听消息
    _oneChannel.setMethodCallHandler((call) {
      setState(() {
        pageIndex = call.method;
      });
      return null;
    });

    _msgChannel.setMessageHandler((message) {
      print('收到 native 信息: $message');
      return null;
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Just a demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: rootPage(pageIndex),
    );
  }
  
  rootPage(String pageIndex) {
    switch (pageIndex) {
      case 'one':
        return FirstPage();
      case 'two':
        return SecondPage();
    }
  }

}
class FirstPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('First Page 01')),
      body: Center(
        child: RaisedButton(
          onPressed: (){
            // 点击事件, 返回上个页面
            MethodChannel('method-channel').invokeMapMethod('back');
          },
          child: Text('Back'),
        ),
      ),
    );
  }
}
class SecondPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Second Page 02')),
      body: Center(
        child: RaisedButton(
          onPressed: (){
            // 点击事件, 修改上个页面的背景色
            MethodChannel('method-channel').invokeMapMethod('changeBackgroundColor');
          },
          child: Text('ChangeBackgroundColor'),
        ),
      ),
    );
  }
}
```

#### 3.3 使用 FlutterBoost

> 官方描述
>
> FlutterBoost 是一个Flutter插件，它可以轻松地为现有原生应用程序提供Flutter混合集成方案。FlutterBoost的理念是将Flutter像Webview那样来使用。在现有应用程序中同时管理Native页面和Flutter页面并非易事。 FlutterBoost 帮你处理页面的映射和跳转，你只需关心页面的名字和参数即可（通常可以是URL）

FlutterBoost 的使用比较简单，但需要 native 与 flutter 都使用该插件，同时 flutter 内部页面的打开与关闭最好也使用该插件提供的方法

[FlutterBoost 集成文档](https://github.com/alibaba/flutter_boost/blob/master/INTEGRATION.md)