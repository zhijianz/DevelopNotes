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

```xml

<declare-styleable name="CustomAttr">
    <attr name="custom_attr_boolean" format="boolean"/>
    <attr name="custom_attr_integer" format="integer"/>
    <attr name="custom_attr_dimension" format="dimension"/>
    <attr name="custom_attr_float" format="float"/>
    <attr name="custom_attr_color" format="color"/>
    <attr name="custom_attr_string" format="string"/>
    <attr name="custom_attr_reference" format="reference"/>
    <attr name="custom_attr_fraction" format="fraction"/>
    <attr name="custom_attr_enum">
        <enum name="enum_01" value="1"/>
        <enum name="enum_02" value="2"/>
    </attr>
    <attr name="custom_attr_flag">
        <flag name="flag_01" value="1"/>
        <flag name="flag_02" value="2"/>
        <flag name="flag_03" value="3"/>
    </attr>
    <attr name="custom_attr_composite" format="dimension|enum">
        <enum name="enum_01" value="1"/>
        <enum name="enum_02" value="2"/>
    </attr>
</declare-styleable>

```

上面的示例代码中枚举了自定义属性可以使用的数据类型，在定义属性的时候可以使用其中的一种或者是多种类型定义当前属性。除了常见的基础属性之外，还有下面的一些特殊的属性需要介绍一下。

- enum 枚举类型定义的类型在使用的时候只能固定使用枚举值的其中之一
- flag 位或类型定义的类型在使用的时候可以使用多个值，最后的结果是多个值的位或运算结果

## 自定义属性使用方式

自定义属性可以在`layout/style/theme` 直接使用，但是前提需要声明并且使用自定义的命名空间`xmlns:app="http://schemas.android.com/apk/res-auto"`。

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/activity_main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <com.example.zhijianz.democustomattr.CustomView
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:custom_attr="custom attr"/>
</LinearLayout>

```

## 关于`obtainStyledAttributes`函数和`TypedArray`

在布局文件或其他的地方使用自定义属性的目的是为了可以在逻辑代码中获得这些初始值的内容。这部分介绍基本的获取方式的同时也会详细解释`TypedArray`不同的获取方式，在这里面主要是涉及到`View`构造函数参数和`Context.obtainStyledAttributes`函数的分析。

### View 构造函数参数

一般情况下我们在出现`View`构造函数的时候都会遇到四种不同的参数的构造函数，不同的构造函数调用的时机以及其中不同参数的意义在大多数情况下都不会太过于深究。但实际上这些不同的选择对于自定义属性或者是自带属性的覆盖有着深渊的影响。

#### `View(Context context)`

这个构造函数是四个中最简单的一个，参数只有一个创建控件的上下文`context`，通常的使用常见是在Java代码中动态的创建一个新的控件。从这个构造函数的参数来看并没有涉及到任何和属性相关的内容，所以如果不在该函数中手动去进行另外的属性相关获取操作，那么这个新产生的控件使用的都是父类在当前主题中可以获取到的基础属性（依赖于该构造函数的父类实现）。

#### `View(Context context, AttributeSet attrs)`

如果希望自定义的控件可以直接在布局文件中进行定义，那么就必须要实现这个构造函数，因为这个构造函数会在`Layoutinflator.inflate`从布局文件创建控件的过程中调用。

在这个函数里面有两个参数，第一个是必须的上下文然后是一个包含所有在布局文件中定义的该控件的属性集合`AttributeSet`，可以从`Layoutinflator.inflate`执行的源代码中观测到它的创建方式。

```java
public View inflate(XmlPullParser parser, ViewGroup root, boolean attachToRoot){
  ...
  final AttributeSet attrs = Xml.asAttributeSet(parser);
  ...
}
```

#### `View(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)`

直接跳到四参是因为相比于三参只是多了最后的`int defStyleRes`，通常在三参内部直接将四参最后一个参数设置为0然后调用是一样的效果。三参四参的构造函数对于自定义控件来说系统并不会在创建的时候自动调用，通常是在合适的位置手动调用或者是实现的基类的构造函数有调用。后两个参数指代的内容会在下一个小结的时候进行介绍。
