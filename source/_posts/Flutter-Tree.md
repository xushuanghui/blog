---
title: Flutter Tree
date: 2020-10-17 17:47:01
tags: flutter
---



![2020-12-17-5.47.46.png](https://i.loli.net/2020/12/17/vh2TFclGnkx1mDC.png)

我们也可以看到上图中每个控件所形成的树结构中隐含了一些关系，例如在上图中，我们可以说 Text 组件是 Column 组件的子组件，Scaffold 是 AppBar 的父组件，这样的层级关系使得每个控件都清晰的连接到了一起，树结构由此而来（俄罗斯套娃）。

在 Flutter 中，Container、Text 等组件都属于 Widget，所以我们将这种树称为 Widget 树，也可以叫做控件树，它就表示了我们在 dart 代码中所写的控件的结构。

![2020-12-17-5.47.54.png](https://i.loli.net/2020/12/17/aI3GmoWswkR1dHr.png)

然而，在 Flutter 体系结构中，真正做组件渲染在屏幕上这个任务的并非在 控件层（Widget）层，而是在渲染（Rendering）层，那么我们在代码中所写组件又是怎么通过渲染层显示的呢？Flutter 中又引入了 Element 树和 RenderingObject 树两棵树。

Element 是什么，我们可以把它称之为 Widget 另一种抽象。读者也可以把它看作一个更为实际控件，因为在我们的手机屏幕上显示的控件并非我们在代码中所写的 Widget，我们在代码中所使用的像 Container、Text 等这类组件和其属性只不过是我们想要构建的组件的配置信息，当我们第一次调用 `build()` 方法想要在屏幕上显示这些组件时，Flutter 会根据这些信息生成该 Widget 控件对应的 Element，同样地，Element 也会被放到相应的 Element 树当中。在 Flutter 中，一个 Widget 通过多次复用可以对应多个 Element 实例，Element 才是我们真正在屏幕上显示的元素。

> Element 与 Widget 另一个区别在于，Widget 天然是不可变的（immutable），它如要更新便需要重建，如果想要把可变状态与 Widget 关联起来，可以使用 StatefulWidget，StatefulWidget 通过使用StatefulWidget.createState 方法创建 State 对象，并将之扩充到 Element 以及合并到树中；

这里，为了更为深刻的理解以上描述的含义，我们可以举一个更为形象的例子。Widget 作为大 Boss，他把近期的战略部署，即配置信息，写在纸上下发给经理人 Element，Element 看到详细的配置信息开始真正的开起活来了。我们还需要注意一点，大 Boss 随时会改变战略部署，然后不会在原有的纸上修改而是重新写下来，这时经理人为了减少工作量需要将新的计划与旧的计划比较来作出相应的更新措施。这也是 Flutter 框架层做的一大优化。下面又来了，Element 作为经理人也很体面，当然不会把活全干完，于是又找了一个 RenderObject 的员工来帮它做粗重的累活。

RenderObject 在 Flutter 当中做组件布局渲染的工作，其为了组件间的渲染搭配及布局约束也有对应的 RenderObject 树，我们也称之为渲染树。

熟悉了 Flutter 中的上述三颗树，相信读者会对组件的渲染过程有了一个清晰的认识，这对我们之后学习常用组件有很大的帮助，我们需要用不同的眼光去看待我们所建立的布局和控件，之后我们也会更加深入的去理解其中更不为人知的奥秘。

<!--more-->

## 组件渲染过程简述

从上文中，我们知道控件树中的每个控件都会实现一个 RenderObject 对象做渲染任务，并将所有的RenderObject 组成渲染树。Flutter 渲染组件的过程如下：

![2020-12-17-5.48.06.png](https://i.loli.net/2020/12/17/X14JaTKv2iqyLZs.png)

Flutter 的渲染过程由用户的输入开始，当接受到用户输入的信号时，就会触发动画的进度更新，例如我们第一次渲染时的启动动画，或者我们在滚动手机屏幕时单个列表项复用时的移动动画。之后便需要开始视图数据的构建（build），这一步中 Flutter 创建了前文所描述的三棵视图树。

在这之后，视图才会进行布局（layout），计算各个部分的大小，然后进行绘制（paint），生成每个视图的视觉数据，这部分的任务主要就是由 RenderObject 所做。这里，Flutter 中的布局过程可用下图表示，在上述构建完成渲染树后，父渲染对象会将布局约束信息向下传递，子渲染对象根据自己的渲染情况返回 Size，Size 数据会向上传递，最终父渲染对象完成布局过程。

![2020-12-17-5.48.27.png](https://i.loli.net/2020/12/17/T5KdizEHlDCPtUx.png)

最后一步进行“光栅化”（Rasterize），前一步得到合成的视图数据其实还是一份矢量描述数据，光栅化帮助把这份数据真正地生成一个一个的像素填充数据。在 Flutter 中，光栅化这个步骤被放在了 Engine 层中。

在日常开发学习中，我们只需要在代码层配置好我们的 Widget 树，了解各种 Widget 特性及使用方法，其余的工作都可以交给我们的框架层去实现。

## 元素树详解

我们已经知道了各类控件的作用及其使用方法，这些 Widget 被我们开发人员配置了多个属性来定义它的展现形式，例如配置 Text 组件需要显示的字符串，配置输入框组件需要显示的内容。我们 Element 树会记录这些配置信息。熟悉 React 的读者可能了解过其中的 “虚拟 DOM” 这个概念，上述 Flutter 这种操作也正体现了这一概念。Widget 是不可变，它的改变就意味着要重建，而其重建也非常频繁，如果我们将更多的任务都交给它将会对性能造成很大的损伤，因此我们把 Widget 组件当作一个虚拟的组件树，而真正被渲染在屏幕上的其实是 Elememt 这棵树，它持有其对应 Widget 的引用，如果他对应的 Widget 发生改变，它就会被标记为 dirty Element，于是下一次更新视图时根据这个状态只更新被修改的内容，从而达到提升element-3845797性能的效果。

每次，当控件挂载到控件树上时，Flutter 调用其 createElement() 方法，创建其对应的 Element。Flutter 再将这个 Element 放到元素树上，并持有创建它控件的引用，如下图：

![2020-12-17-5.48.46.png](https://i.loli.net/2020/12/17/Qk581zKSOLBUZfJ.png)



子控件也会创建相应 Element 被放在元素树上：

![2020-12-17-5.48.53.png](https://i.loli.net/2020/12/17/7P6cIbrZfQWKBRj.png)

### Element 中的状态

我们上文提到了 Widget 的不可变性，相应的 Element 就有其可变性，正如我们前文所说的它被标记为 dirty Element 便是作为需要更新的状态，另外一个我们需要格外注意的是，有状态组件（statefulWidget）对应的 State 对象其实也被 Element 所管理，如下图所示。

{% asset_img 可变的元素-3848662.svg %}

Flutter 中的 Widget 一直在重建，每次重建之后，Element 都会采用相应的措施来确定是否我对应的新控件跟之前引用旧控件是否有所改变，如果没改变则只需要做更新操作，如果前后不同则会重创建。那么，Element 根据什么来确定控件是否改变呢？它会比较 Widget 以下两个属性：

- 组件类型
- Widget 的 Key （如果有）

组件类型即前后控件的是否是同一个类所创建的，Key 即为每个控件的唯一标识。

## 渲染树详解

我们已经大致知道 Flutter 中的三棵重要的树及 Element 树的工作原理，其中第三棵渲染树的任务就是做组件的具体的布局渲染工作。

渲染树上每个节点都是一个继承自 RenderObject 类的对象，其由 Element 中的 renderObject 或  RenderObjectWidget 中的 createRenderObject 方法生成，该对象内部提供多个属性及方法来帮助框架层中的组件如何布局渲染。

> 我们在本章之前已经介绍了 StatelessWidget 和 StatefulWidget 两种直接继承自 Widget 的类，在 Flutter 中，还有另一个类 RenderObjectWidget 也同样直接继承自 Widget，它没有 build 方法，可通过 createRenderObject 直接创建 RenderObject 对象放入渲染树中。Column 和 Row 等控件都间接继承自RenderObjectWidget。

主要属性和方法如下：

- constraints 对象，从其父级传递给它的约束
- parentData 对象，其父对象附加有用的信息。
- performLayout 方法，计算此渲染对象的布局。
- paint 方法，绘制该组件及其子组件。

RenderObject 作为一个抽象类。每个节点需要实现它才能进行实际渲染。扩展 RenderOject 的两个最重要的类是RenderBox 和 RenderSliver。这两个类分别是应用了 Box 协议和 Sliver 协议这两种布局协议的所有渲染对象的父类，其还扩展了数十个和其他几个处理特定场景的类，并实现了渲染过程的细节，如 RenderShiftedBox 和 RenderStack 等等。

### 布局约束

在上面，我们介绍组件渲染流程时，我们了解到了 Flutter 中的控件在屏幕上绘制渲染之前需要先进行布局（layout）操作。其具体可分为两个线性过程：从顶部向下传递约束，从底部向上传递布局信息，其过程可用下图表示。

![2020-12-17-5.49.10.png](https://i.loli.net/2020/12/17/UpYBTI5ahEFgeRP.png)

第一个线性过程用于传递布局约束。父节点给每个子节点传递约束，这些约束是每个子节点在布局阶段必须要遵守的规则。就好像父母告诉自己的孩子 ：“你必须遵守学校的规定，才可以做其他的事”。常见的约束包括规定子节点最大最小宽度或者子节点最大最小的高度。这种约束会向下延伸，子组件也会产生约束传递给自己的孩子，一直到叶子结点。

第二的线性过程用来传递具体的布局信息。子节点接受到来自父节点的约束后，会依据它产生自己具体的布局信息，如父节点规定我的最小宽度是 500 的单位像素，子节点按照这个规则可能定义自己的宽度为 500 个像素，或者低于 500 像素的任何一个值。这样，确定好自己的布局信息之后，将这些信息告诉父节点。父节点也会继续此操作向上传递一直到最顶部。

下面我们具体介绍有哪些具体的布局约束可在树中传递。Flutter 中有两种主要的布局协议：Box 盒子协议和 Sliver 滑动协议。这里我们先以盒子协议为例展开具体的介绍。

在盒子协议中，父节点传递给其子节点的约束为 BoxConstraints。该约束规定了允许每个子节点的最大和最小宽度和高度。如下图，父节点传入 Min Width 为 150，Max Width 为 300 的 BoxConstraints：

![2020-12-17-5.49.18.png](https://i.loli.net/2020/12/17/DAj6H8GMX3cyPWs.png)

当子节点接受到该约束，便可以取得上图中绿色范围内的值，即宽度在 150 到 300 之间，高度大于 100，当取得具体的值之后再将取得具体的大小的值上传给父节点，从而达到父子的布局通信。

### 自定义一个 Center 控件

之后更新，大家也可以看各组件的源码探究其如何应用上面提到的原理。

---2019.07.03 更新

现在，我们可以应用前文中提到的布局约束与渲染树相关的概念自己定义一个类似居中布局的组件 RenderObject 对象渲染在屏幕上。

所以我们称自己自定义组件为 CustomCenter：

```
void main() {
  runApp(MaterialApp(
    home: Scaffold(
      body: Container(
        color: Colors.blue,
        constraints: BoxConstraints(
            maxWidth: double.infinity,
            minWidth: 100.0,
            maxHeight: double.infinity,
            minHeight: 100.0),
        child: CustomCenter(
          child: Container(
            color: Colors.red,
          ),
        ),
      ),
    ),
  ));
}
```

现在我们来实现我们的 CustomCenter：

```
class CustomCenter extends SingleChildRenderObjectWidget {
  Stingy({Widget child}) : super(child: child);

  @override
  RenderObject createRenderObject(BuildContext context) {
    // TODO: implement createRenderObject
    return RenderCustomCenter();
  }
}
```

`CustomCenter` 继承了 `SingleChildRenderObjectWidget`，表明这个 Widget 只能有一个子控件， 其中，`createRenderObject(...)` 方法用于真正创建并返回我们的 `RenderObject` 对象实例， 我们的 RenderObject 为 `RenderCustomCenter`，代码如下：

```
class RenderCustomCenter extends RenderShiftedBox {
  RenderStingy() : super(null);

  // 重写绘制方法
  @override
  void paint(PaintingContext context, Offset offset) {
    // TODO: implement paint
    super.paint(context, offset);
  }

  // 重写布局方法
  @override
  void performLayout() {
    // 布局子元素并向下传递布局约束
    child.layout(
        BoxConstraints(
            minHeight: 0.0,
            maxHeight: constraints.minHeight,
            minWidth: 0.0,
            maxWidth: constraints.minWidth),
        parentUsesSize: true);

    print('constraints: $constraints');

    // 指定子元素的偏移位置
    final BoxParentData childParentData = child.parentData;
    childParentData.offset = Offset((constraints.maxWidth - child.size.width)/2,
        (constraints.maxHeight - child.size.height)/2);
    print('childParentData: $childParentData');

    // 定义自己（CustomCenter）的大小，这里选择约束对象的最大值
    size = Size(constraints.maxWidth, constraints.maxHeight);
    print('size: $size');
  }
}
```

`RenderCustomCenter` 继承自 `RenderShiftedBox`，该类是继承自 `RenderBox`。`RenderShiftedBox` 满足盒子协议，并且提供了 `performLayout()` 方法的实现。我们需要在 `performLayout()` 方法中布局我们的子元素。??

我们在使用 `child.layout(...)` 方法布局 *child* 的时候传递了两个参数，第一个为 *child* 的布局约束，而另外一个参数是 `parentUserSize`， 该参数如果设置为 `false`，则意味着 *parent* 不关心 *child* 选择的大小，这对布局优化比较有用；因为如果 *child* 改变了自己的大小，*parent* 就不必重新 `layout` 了。但是在我们的例子中，我们的需要把 *child* 放置在 *parent* 的中心，就是 *child* 的**大小（Size）一旦改变，则其对应的偏移量（Offset）** 也会改变，于是 *parent* 需要重新布局，所以我们这里传递了一个 `true`。

当 `child.layout(...)` 完成了以后，*child* 就确定了自己的 *Layout Details*。然后我们就还可以为其设置偏移量来将它放置到我们想放的位置。在我们的例子中为 **居中**。

最后，和 *child* 根据 *parent* 传递过来的约束选择了一个尺寸一样，我们也需要为 **CustomCenter** 选择一个尺寸。



## 应用视图的构建

Flutter App 入口的部分发生于如下代码：

```
import 'package:flutter/material.dart';

// 这里的 MyApp是一个 Widget
void main() => runApp(new MyApp());
```

`runApp`函数接受一个 Widget类型的对象作为参数，也就是说在 Flutter的概念中，只存在 View，而其他的任何逻辑都只为 View的数据、状态改变服务，不存在 ViewController(或者叫 Activity）。接下来看 `runApp`做了什么：

```
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..attachRootWidget(app)
    ..scheduleWarmUpFrame();
}

class WidgetsFlutterBinding extends BindingBase with GestureBinding, ServicesBinding, SchedulerBinding, PaintingBinding, RendererBinding, WidgetsBinding {
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      new WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

在 `runApp` 中，传入的 widget 被挂载到根 widget 上。这个 `WidgetsFlutterBinding` 其实是一个单例，通过 mixin 来使用框架中实现的其他 binding 的 Service，比如手势、基础服务、队列、绘图等等。然后会调用 `scheduleWarmUpFrame` 这个方法，从这个方法注释可知，调用这个方法会主动构建视图数据。这样做的好处是因为 Flutter 依赖 Dart 的 MicroTask 来进行帧数据构建任务的 schedule，这里通过主动调用进行整个周期的 “热身”，这样最近的下次 VSync 信号同步时就有视图数据可提供，而不用等到 MicroTask 的 next Tick。

然后我们再来看 `attachRootWidget` 这个函数干了什么：

```
void attachRootWidget(Widget rootWidget) {
    _renderViewElement = new RenderObjectToWidgetAdapter<RenderBox>(
      container: renderView,
      debugShortDescription: '[root]',
      child: rootWidget
    ).attachToRenderTree(buildOwner, renderViewElement);
}
```

`attachRootWidget` 把 widget交给了 `RenderObjectToWidgetAdapter`这座桥梁，通过这座桥梁，Element 被创建，并且同时能持有 Widget 和 RenderObject的引用。然后我们从上文就知道后面发生的就是第一次的视图数据构建了。