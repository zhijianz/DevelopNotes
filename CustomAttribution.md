# 自定义属性

- 自定义属性用途
- 自定义属性基本操作和产生数据的本质
- 自定义属性的类型
- 自定义属性的使用方式
- 关于`obtainStyledAttributes`函数和`TypedArray`

## 自定义属性的用途

虽然`Android SDK`中为开发者提供了多种类型的控件，但是自定义控件仍然是一个无法回避的技术点，不管是对于新手开发者用于系统矿建的学习还是资深开发者实现需求功能。在创建自定义控件的过程中，为了使得逻辑运行过程中使用到的一些参数属性可以在`Layout`文件中直接被访问到并且进行对应的初始化配置，就需要对这些内容进行自定义属性使得可以像`android:layout_width`一样在布局文件中直接进行初始化的设置。

## 自定义属性基本操作

自定义属性操作通常是在`res/value/attrs.xml`文件中使用`<attr />`或者`<declare-styleable />`对属性或者属性组进行声明，在声明的时候指定属性的名称及其类型。

```xml

<declare-styleable name="Customize">
    <attr name="attr_1" format="string" />
    <attr name="attr_2" format="string" />
    <attr name="attr_3" format="string" />
</declare-styleable>

<attr name="custom_attr" format="string" />

```

上面的代码中使用了两种不同的方式进行属性的定义，这两者之间的差别体现在`R.styleable`中对应属性ID的生成方式；直接使用`<attr/>`进行属性声明的对应该属性生成一个特定的访问ID，而使用`<declare-styleable/>`创建属性组的时候不仅会对其中的每一个属性生成访问ID，并且会将它们放置在一个统一的数组当中。具体情况可参见下面的代码。

```java
public static final R {
  public static final class styleable {
    public static final int[] Customize = {
            0x7f010098, 0x7f010099, 0x7f01009a
        };
    public static final int Customize_attr_1 = 0;
  }
}
```

## 自定义属性的类型
