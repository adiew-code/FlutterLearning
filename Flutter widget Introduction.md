# Flutter widget Introduction



- ### Container是什么？自己宽高如何确定呢？


内部其实就是做了一层层包装，你给Container设置的每个属性，内部自动帮你裹上一层widget，避免出现“金字塔”形代码，可读性差。

**QA：**其自身size如何确定呢？

```dart
@override
  Widget build(BuildContext context) {
    Widget? current = child;

    if (child == null && (constraints == null || !constraints!.isTight)) {
      current = LimitedBox(
        maxWidth: 0.0,
        maxHeight: 0.0,
        child: ConstrainedBox(constraints: const BoxConstraints.expand()),
      );
    } else if (alignment != null) {
      current = Align(alignment: alignment!, child: current);
    }

    final EdgeInsetsGeometry? effectivePadding = _paddingIncludingDecoration;
    if (effectivePadding != null) {
      current = Padding(padding: effectivePadding, child: current);
    }

    if (color != null) {
      current = ColoredBox(color: color!, child: current);
    }

    if (clipBehavior != Clip.none) {
      assert(decoration != null);
      current = ClipPath(
        clipper: _DecorationClipper(
          textDirection: Directionality.maybeOf(context),
          decoration: decoration!,
        ),
        clipBehavior: clipBehavior,
        child: current,
      );
    }

    if (decoration != null) {
      current = DecoratedBox(decoration: decoration!, child: current);
    }

    if (foregroundDecoration != null) {
      current = DecoratedBox(
        decoration: foregroundDecoration!,
        position: DecorationPosition.foreground,
        child: current,
      );
    }

    if (constraints != null) {
      current = ConstrainedBox(constraints: constraints!, child: current);
    }

    if (margin != null) {
      current = Padding(padding: margin!, child: current);
    }

    if (transform != null) {
      current = Transform(transform: transform!, alignment: transformAlignment, child: current);
    }

    return current!; 
  }
```



- ### Column widget 如何布局child？自己宽高如何确定？【Row同理】


child分为2种，flexible、固定尺寸的

- ###### 宽度

  1、如果设置了 crossAxisAlignment: CrossAxisAlignment.stretch，宽度为parent widget允许的最大宽度

  2、否则宽度能包裹住所有child即可（最大的child的宽度，当然必须满足parent的宽度约束）

- ###### 高度

  1、如果child 全是flexible 则为parent maxHeight，然后各个child再按比例分割

  2、如果设置了mainAxisSize: MainAxisSize.max，主轴上尽可能大，高度取parent widget允许的最大高度
  
  3、否则：MainAxisSize.min，主轴上尽可能小，高度能包裹住所有child即可。（必须满足parent的高度约束，否则溢出）

**QA：**如果child全是flexible、且设置了MainAxisSize.min，高度怎么办？

###### 布局步骤：

1、将约束传递给固定尺寸的 child widget，child必须返回其size给Row（高度unconstrained，宽度是放松后parent的约束）
2、剩下的总空间已经确定，根据所有Flex子组件的弹性系数分配剩余空间（高宽已经确定，紧约束，宽度是放松后parent的约束）

- ### UnconstrainedBox 允许child无限大，DEBUG模式下超了会有警告，child太大会被裁剪

  ```dart
  /// Creates a widget that imposes no constraints on its child, allowing it to
    /// render at its "natural" size. If the child overflows the parents
    /// constraints, a warning will be given in debug mode.
    const UnconstrainedBox({
      super.key,
      this.child,
      this.textDirection,
      this.alignment = Alignment.center,
      this.constrainedAxis,
      this.clipBehavior = Clip.none,
    });
  
  ```

  

- ### ConstraintsTransformBox 约束自定义转换—允许对parent传递过来的约束进行自定义转换后传给child

  1、允许对parent传过来的约束进行自定义转换后，再传给child

  2、自身的大小，在满足自身约束前提下，尽可能包裹child

  ```dart
  /// Creates a widget that uses a function to transform the constraints it
    /// passes to its child. If the child overflows the parent's constraints, a
    /// warning will be given in debug mode.
    ///
    /// The `debugTransformType` argument adds a debug label to this widget.
    ///
    /// The `alignment`, `clipBehavior` and `constraintsTransform` arguments must
    /// not be null.
  const ConstraintsTransformBox({
      super.key,
      super.child,
      this.textDirection,
      this.alignment = Alignment.center,
    ///typedef BoxConstraintsTransform = BoxConstraints Function(BoxConstraints);
      required this.constraintsTransform, ///约束转换方法，将parent给的约束进行自定义转换
      this.clipBehavior = Clip.none,
      String debugTransformType = '',
    }) : _debugTransformLabel = debugTransformType;
  
  
  @override
    void performLayout() {
      final BoxConstraints constraints = this.constraints;
      final RenderBox? child = this.child;
      if (child != null) {
        final BoxConstraints childConstraints = constraintsTransform(constraints);///将parent约束进行自定义转换
        assert(childConstraints.isNormalized, '$childConstraints is not normalized');
        _childConstraints = childConstraints;
        child.layout(childConstraints, parentUsesSize: true);///child使用转换后的约束，并返回child的大小
        size = constraints.constrain(child.size);///自身的大小尽可能包裹child，当然必须满足自己约束
        alignChild();
        final BoxParentData childParentData = child.parentData! as BoxParentData;
        _overflowContainerRect = Offset.zero & size;
        _overflowChildRect = childParentData.offset & child.size;
      } else {
        size = constraints.smallest;
        _overflowContainerRect = Rect.zero;
        _overflowChildRect = Rect.zero;
      }
      _isOverflowing = RelativeRect.fromRect(_overflowContainerRect, _overflowChildRect).hasInsets;
    }
  ```

  

- ### OverflowBox 允许child溢出

  1、自身尺寸取parent能允许的最大值

  2、把参数中指定的约束直接传给child（忽略parent给的约束，字段为空则取parent的值），允许child溢出

  ```dart
  /// A widget that imposes different constraints on its child than it gets
  /// from its parent, possibly allowing the child to overflow the parent.
  class OverflowBox extends SingleChildRenderObjectWidget {
    /// Creates a widget that lets its child overflow itself.
    const OverflowBox({
      super.key,
      this.alignment = Alignment.center,
      this.minWidth, ///有的话，约束不做任何处理直接传给child；如果是null，则使用parent的值，下同
      this.maxWidth,
      this.minHeight,
      this.maxHeight,
      super.child,
    });
    
    ///核心布局代码：
     BoxConstraints _getInnerConstraints(BoxConstraints constraints) {
      return BoxConstraints(
        minWidth: _minWidth ?? constraints.minWidth,///使用用户指定的约束值，为空则使用parent的值，下同
        maxWidth: _maxWidth ?? constraints.maxWidth,
        minHeight: _minHeight ?? constraints.minHeight,
        maxHeight: _maxHeight ?? constraints.maxHeight,
      );
    }
  
    @override
    bool get sizedByParent => true;
  
    @override
    Size computeDryLayout(BoxConstraints constraints) {
      return constraints.biggest;///使用parent给的约束最大值
    }
  
    @override
    void performLayout() {
      if (child != null) {
        child?.layout(_getInnerConstraints(constraints), parentUsesSize: true);
        alignChild();
      }
    }
  ```

  

- ### SizedOverflowBox 指定自己的size，且允许child溢出

  1、允许child超过自己的size

  2、child的尺寸必须满足原始约束（不是unconstraint）

  3、自身的尺寸尽可能靠近指定的size（当然是在满足parent给的约束条件下）。

  ```dart
  const SizedOverflowBox({
      super.key,
      required this.size,///接收一个指定的size，尽可能当成自己的size
      this.alignment = Alignment.center,///child对齐方式
      super.child,
    });
  
  @override
    void performLayout() {
      size = constraints.constrain(_requestedSize);///将自己的size设置为满足父约束的基础上尽可能靠近指定的size
      if (child != null) {
        child!.layout(constraints, parentUsesSize: true);///将原始约束直接传给child
        alignChild();
      }
    }
  ```

  

- ### FitedBox

- ### Offstage隐藏与显示

- ### Card

- ### Wrap

- ### LimitedBox 作用


1. Parent widget 如果传递的是无边界，则使用limitedbox 指定的值

2. 否则limitedbox 无效

   

- ### Align、Center如何布局？其自身大小如何确定？


这两个控件想让child对齐，理论上自身size当然越大越好（海阔凭鱼跃，取parent允许的最大值），如此才能有足够的空间给child摆布！，除非

- parrent的最大值是无穷大，那就没办法了，因为size必须有确定的值，又不能取parent的最小值（太小就无法摆下child），所以只能往child的大小上收缩（尽可能包裹住child），

- 强制指出了factor字段，告诉我不能太大，最大只能是child的factor倍

  从代码角度看，的确如此：

```dart
 void performLayout() {
    final BoxConstraints constraints = this.constraints;
    final bool shrinkWrapWidth = _widthFactor != null || constraints.maxWidth == double.infinity;
    final bool shrinkWrapHeight = _heightFactor != null || constraints.maxHeight == double.infinity;

    if (child != null) {
      child!.layout(constraints.loosen(), parentUsesSize: true);
      size = constraints.constrain(Size(
        shrinkWrapWidth ? child!.size.width * (_widthFactor ?? 1.0) : double.infinity,
        shrinkWrapHeight ? child!.size.height * (_heightFactor ?? 1.0) : double.infinity,
      ));
      alignChild();
    } else {
      size = constraints.constrain(Size(
        shrinkWrapWidth ? 0.0 : double.infinity,
        shrinkWrapHeight ? 0.0 : double.infinity,
      ));
    }

  }
```

[widthFactor] and [heightFactor]，代表Align or Center是其child的倍数

###### 布局步骤：

1、判断是否需要收缩（即尽可能包裹child）【设置factor 或者 parent约束为无界】

2、对parent传递过来的constrain进行loose后传递给其child

3、有孩子的情况下，需要收缩则尽可能包裹child，否则为parent最大值

4、无孩子，需要收缩则取parent最小值，否则为parent最大值

###### 两种alignment

1、Alignment(0, 0) 以中心点为坐标系原点，范围【-1，1】

2、FractionalOffset(0.2, 0.6)，以左上角为坐标系原点，范围【0，1】



- ### FractionalSizeBox 设置child的宽高比例


可以设置child的宽是parent宽的 widthFactor 倍（即将parent的最大宽高乘以Factor），得到**紧约束**，传递给其child，如果factor>1 那么child就比自身大，但是自己的size 必须满足parent的约束。

```dart
///若某个维度设置了factor，则取其最大值的factor倍并为紧约束向下传递，否则维持parent约束不变
BoxConstraints _getInnerConstraints(BoxConstraints constraints) {
    double minWidth = constraints.minWidth;
    double maxWidth = constraints.maxWidth;
    if (_widthFactor != null) {///有则取最大值的_widthFactor 倍，
      final double width = maxWidth * _widthFactor!;///可能超过parent的宽度
      minWidth = width;///minWidth == maxWidth为紧约束
      maxWidth = width;
    }
    double minHeight = constraints.minHeight;
    double maxHeight = constraints.maxHeight;
    if (_heightFactor != null) {///_heightFactor 倍，
      final double height = maxHeight * _heightFactor!;///可能超过parent的高度
      minHeight = height;///minHeight == maxHeight为紧约束
      maxHeight = height;
    }
    return BoxConstraints(
      minWidth: minWidth,
      maxWidth: maxWidth,
      minHeight: minHeight,
      maxHeight: maxHeight,
    );
}

@override
void performLayout() {
    if (child != null) {
      child!.layout(_getInnerConstraints(constraints), parentUsesSize: true);//得到紧约束，可能比自己大
      size = constraints.constrain(child!.size);///但是自己的size必须遵守parent的约束，不能比parent大
      alignChild();
    } else {
      size = constraints.constrain(_getInnerConstraints(constraints).constrain(Size.zero));
    } 
}
```



- ### CustomeMutulChildLayout 自定义布局


###### 优点：

可传递给child任意约束，需要通过layoutId 组件包裹

###### 自身size；

可以通过重写getSize 方法返回给其Parent（默认返回parent.biggest），无法根据child变化

###### 缺陷：

1、本身大小无法随child 而变化
2、child不能无中生有，必须从外面传递过来，用LayoutId 包裹

​	

- ### Stack 如何布局child？自身的尺寸如何确定？


child widget 分为两种：有位置的（被Position包裹的）、无位置的（没有被Position包裹的）

###### 布局步骤：

1、先布局无位置的child，其中最大的size作为自身size

2、自身size确定后，才能确定剩下的有位置posioned的组件

###### 自身尺寸：

1、如果child widget 全是有位置的（Position），则自身size为parent widget 允许的的最大尺寸
2、否则取所有无位置的child中的最大尺寸

###### Stack.fit属性：

1、expand 将parent的constraint进行expend后传给child（size={maxWidth，maxHeight}）紧约束

2、passthrough 将parent的constraint 透传给child

3、loose 将parent的constraint进行loosen后传给child（minWidth=0、minHeight=0）

###### 特点：

1、允许child widget之间叠加在一起

2、默认所有的child都按照alignment属性进行排练，如果child之间有对其、间距等布局要求，可以配合其他widget实现





- ### Key 分为GLobal Key、 LocalKey


**QA：**为什么需要Key？Widget 发生变化时（位置移动、层级移动、增加、删除），会根据class & key二者去匹配原始的Element（含state信息），没有key可能会匹配错误，或者丢失，为了使得能重新匹配到原始的State信息，需要Key。

Note：Widget在同一层级位置变化时，使用LocalKey即可让Flutter找回她的状态，如果是移动到不同层级时，只能使用GlobalKey

- Local Key（只判断同一层级）

  1、ValueKey 判断两个对象是否相等

  2、ObjectKey 判断是否同一个对象

  3、UniqueKey 唯一值

- GLobal Key

  1、widget移动到不同层级时保存状态

  2、类似getElementById,可直接找到该Widget进行操作 【不推荐】
  
  

- ### ConstrainedBox给child增加额外的约束


比较简单，在满足parent constrain的基础上，增加额外的约束，并传递给child，将自身size设置为child的size

```dart
void performLayout() {
    final BoxConstraints constraints = this.constraints;
    if (child != null) {
      child!.layout(_additionalConstraints.enforce(constraints), parentUsesSize: true);
      size = child!.size;
    } else {
      size = _additionalConstraints.enforce(constraints).constrain(Size.zero);
    }
  }
```

###### 自身size:

1、有child，跟child一样大小

2、否则取parent允许的最小宽高
