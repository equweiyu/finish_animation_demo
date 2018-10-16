# Flutter 中如何绘制动画
### 首先是绘图
在Flutter中绘图非常简单。关键词`CustomPainter`, `CustomPaint`, `Canvas`。

在iOS/Android中我们继承`UIView/View`重写`draw/onDraw`方法在里面执行画图操作。
在flutter中稍微有点不一样，我们使用`CustomPaint`(这是一个widget)，它需要一个参数`painter`，这个参数的类型是一个抽象类`CustomPainter`。
我们需要实现这个类的两个关键方法: `paint`,`shouldRepaint`。画什么就由`paint`决定，而只有`shouldRepaint`返回true的时候才会重绘。

实现`void paint(Canvas canvas, Size size)`这个方法，在iOS中我们使用`UIBezierPath`和`Core Graphics`绘图，在Flutter具体的绘制方法用这个`canvas`, 具体的API可以查看[官方文档](https://docs.flutter.io/flutter/dart-ui/Canvas-class.html)

下面这个例子就是画一段圆弧
```dart
class DemoPainter extends CustomPainter {
  final double _arcStart;
  final double _arcSweep;

  DemoPainter(this._arcStart, this._arcSweep);

  @override
  void paint(Canvas canvas, Size size) {
    double side = math.min(size.width, size.height);
    Paint paint = Paint()
      ..color = Colors.blue
      ..strokeCap = StrokeCap.round
      ..strokeWidth = 4.0
      ..style = PaintingStyle.stroke;
    canvas.drawArc(
        Offset.zero & Size(side, side), _arcStart, _arcSweep, false, paint);
  }

  @override
  bool shouldRepaint(DemoPainter other) {
    return _arcStart != other._arcStart || _arcSweep != other._arcSweep;
  }
}
```
使用的时候把`DemoPainter`的实例当做参数传给`CustomPaint`就可以使用了，比如
```dart
Container(
  child: CustomPaint(painter: DemoPainter(0.0, math.pi)),
  height: 200.0,
  width: 200.0,
  color: Colors.deepOrange,
  padding: EdgeInsets.all(30.0),
)
```

显示效果

![image](https://lh3.googleusercontent.com/-ZVkaiJmFcAs/W8QGDj7pdKI/AAAAAAAAAME/QJb_f6MKR5gZjbeXizeepoeF9l4Lu7pzQCHMYCw/I/15395236079374.jpg)




### 然后加动画
Flutter的动画也不复杂，关键词`AnimationController`。

Flutter中的动画是基于`Animation`，这个对象本身是一个抽象类，在一段时间内依次产生一些值。我们使用封装好的`AnimationController`来做动画，它在屏幕刷新的每一帧，产生一个新的值，默认情况是在给定的时间段内线性的生成0.0到1.0的数字。

`AnimationController`有个参数`vsync` 可以绑定到一个`widget`(需要widget扩展`SingleTickerProviderStateMixin`)，当widget不显示时，动画定时器将会暂停，当widget再次显示时，动画定时器重新恢复执行。`duration`属性可以设置持续时间。还有一些方法可以控制动画`forward`启动,`reverse`反转,`repeat`重复。

`AnimationController`有`addListener`和`addStatusListener`方法可以添加监听，一个是值监听一个是状态监听。值监听常用在调用`setState`来触发UI重建来实现动画，状态监听用在动画状态变化的时候执行一些方法，比如在动画结束时反转动画。

至此我们已经可以绘制动画了，代码如下
```dart
class DemoWidget extends StatefulWidget {
  @override
  _DemoWidgetState createState() => _DemoWidgetState();
}

class _DemoWidgetState extends State<DemoWidget>
    with SingleTickerProviderStateMixin {
  AnimationController _controller;

  @override
  void initState() {
    _controller =
        AnimationController(vsync: this, duration: Duration(milliseconds: 1500))
          ..repeat()
          ..addListener(() {
            setState(() {});
          });
    super.initState();
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      painter: DemoPainter(0.0, _controller.value * math.pi * 2),
    );
  }
}

```
![image](https://lh3.googleusercontent.com/-hNxxHenQ8WE/W8QGDtMV-9I/AAAAAAAAAMI/hRjlhg0QtjUB7GN1gMZjvfjOljQ0LhQzwCHMYCw/I/2018-10-14%252B21-44-24.2018-10-14%252B21_45_53.gif)

可以借助`AnimatedBuilder`改写上文的`initState`和`build`方法，使视图层级更加清楚，有助于封装
```dart
  @override
  void initState() {
    _controller =
        AnimationController(vsync: this, duration: Duration(milliseconds: 1500))
          ..repeat();
    super.initState();
  }
  
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(animation: _controller, builder: (context, child) {
      return CustomPaint(
        painter: DemoPainter(0.0, _controller.value * math.pi * 2),
      );
    });
  }
```

### Tween与Curve

[Tween](https://docs.flutter.io/flutter/animation/Tween-class.html)和[Curve](https://docs.flutter.io/flutter/animation/Curve-class.html)可以帮我们更好地控制Animation的值
一般的Animation会在给定的时间内线性的产生0.0到1.0的值

Tween可以把这些转变成我们想要的类型或者是范围
比如`Tween(begin: math.pi * 1.5, end: math.pi * 1.5 + math.pi * 2).evaluate(_controller),`就可以把值的范围转成1.5pi到3.5pi。

Curve是一个抽象类表示生成值的曲线, [Curves](https://docs.flutter.io/flutter/animation/Curves-class.html)已经定义了许多常用的曲线。

这里`Tween`,`Curve`可以使用`chain`,`evaluate`,`transform`和Animation串起来使用

我们可以使用这些更改我们上文的例子，代码如下
```dart
  @override
  Widget build(BuildContext context) {
    return AnimatedBuilder(
        animation: _controller,
        builder: (context, child) {
          return CustomPaint(
            painter: DemoPainter(
              Tween(begin: math.pi * 1.5, end: math.pi * 3.5)
                  .chain(CurveTween(curve: Interval(0.5, 1.0)))
                  .evaluate(_controller),
              math.sin(Tween(begin: 0.0, end: math.pi).evaluate(_controller)) *
                  math.pi,
            ),
          );
        });
  }
```
当然这里第二个参数有更简洁的写法
```
  math.sin(_controller.value*math.pi) *math.pi
```
显示效果

![image](https://lh3.googleusercontent.com/-JlYLIHDfyF0/W8QGD9wiICI/AAAAAAAAAMQ/oL3UeI2MxjU_XenZukEusNF_ASWm4BVSwCHMYCw/I/2018-10-14%252B23-18-32.2018-10-14%252B23_19_57.gif)

### 示例--完成动画
[Github](https://github.com/equweiyu/finish_animation_demo)

![image](https://lh3.googleusercontent.com/-b7VipeCJkb8/W8QGDyh5qTI/AAAAAAAAAMM/Pa6uLJn5bowuTfmmmAvHZs-weRz9kSesQCHMYCw/I/Jietu20181014-232520-HD.2018-10-14%252B23_28_02.gif)
