# 自定义属性

- 自定义属性用途
- 自定义属性基本操作和产生数据的本质
- 自定义属性的类型
- 自定义属性的使用方式
- 关于`obtainStyledAttributes`函数和`TypedArray`

## 自定义属性的用途

虽然`Android SDK`中为开发者提供了多种类型的控件，但是自定义控件仍然是一个无法回避的技术点，不管是对于新手开发者用于系统框架的学习还是资深开发者实现需求功能。在创建自定义控件的过程中，为了让逻辑运行过程中使用到的一些参数属性可以在`Layout`文件中直接被访问到并且进行对应的初始化配置，就需要对这些内容进行自定义属性以达到可以像`android:layout_width`一样在布局文件中直接进行初始化设置的使用效果。

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

上面的代码中使用了两种不同的方式进行属性的定义，这两者之间的差别体现在`R.java`中对应属性ID的生成方式；直接使用`<attr/>`进行属性声明的对应该属性会在`R.attr`生成一个特定的访问ID，而使用`<declare-styleable/>`创建属性组的时候不仅会在`R.styleable`中对其中的每一个属性生成访问ID，并且会将它们放置在一个统一的数组当中。具体情况可参见下面的代码。

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

在布局文件或其他的地方使用自定义属性的目的是为了可以在逻辑代码中获得这些初始值的内容。这部分介绍属性值的基本获取方式的同时也会详细解释`TypedArray`不同创建模式，在这里面主要是涉及到`View`构造函数参数和`Context.obtainStyledAttributes`函数的分析。

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

### `Context.obtainStyledAttributes`

使用自定义属性的最后一步是在Java逻辑代码中获取到在各个地方配置到的属性值，通常会用到下面这段代码：

```java
TypedArray a = context.obtainStyledAttributes(set, attrs, defStyleAttr, defStyleRes);

// 调用获取资源的各种接口

a.recycle();
```

在上面的流程中，所有对于属性值的获取都是基于`TypedArray`对象进行的。在`TypedArray`中保存了希望访问的属性在特定属性集合中的属性。这句话里面有两个关键点，第一点是希望访问的属性集合如何去指定，第二个问题是怎么样去配置特定的属性集合。要解答上面的两个问题，首先需要去分析`TypedArray`的获取过程，也就是`context.obtainStyledAttributes`函数到底做了些什么事情。首先看一下这个函数在源代码中的注解：

```java
/**
* @param set The base set of attribute values.  May be null.
* @param attrs The desired attributes to be retrieved.
* @param defStyleAttr An attribute in the current theme that contains a
*                     reference to a style resource that supplies
*                     defaults values for the TypedArray.  Can be
*                     0 to not look for defaults.
* @param defStyleRes A resource identifier of a style resource that
*                    supplies default values for the TypedArray,
*                    used only if defStyleAttr is 0 or can not be found
*                    in the theme.  Can be 0 to not look for defaults.
*
*/

public TypedArray obtainStyledAttributes(AttributeSet set, @StyleableRes int[] attrs, @AttrRes int defStyleAttr, @StyleRes int defStyleRes)

```

上面截取了对于函数四个参数说明部分的注解，根据这些注解可以尝试对参数的意义和作用进行试探性的理解

- `AttributeSet set`

  `set`保存的是特定的基础属性集合，在介绍`View`构造函数的时候已经解释过它的来源，它来资源对于定义控件的布局文件的解析。这个参数可以是`null`，这个时候表示会使用当前系统`theme`中定义的属性集合作为基础属性集。

- `int[] attrs`

  `attrs`是我们尝试去检索的属性集合。这个数组的创建方式可以是直接创建一个`int[]`然后在其中填入特定的属性ID，或者是直接使用定义好的`styleable`属性。

- `int defStyleAttr`

  `defStyleAttr`是定义在`theme`中的一个引用类型的属性，这个属性指向的是一个样式表，在这个样式表定义的属性值可以作为将要检索的属性默认值，如果在这个地方填入的参数是`0`那就表示不会去搜索任何一个属性来完成这个功能。

- `int defStyleRes`

  `defStyleRes`指向的是一个样式表，当`defStyleAttr`为0或者无法在当前主题中找到需要检索的属性时会尝试使用这个样式表的内容作为属性的值，同时如果这个值指定为`0`也是代表不使用这个特性。

在对于上面四个参数的意义和作用进行分析之后，大体上可以将通过`TypedArray`来获取特定属性集合值的方式概括成下面的图例：

```{viz}
digraph fectch_attr{
  edge[style = dashed, color = slategrey, penwidth = .7];
  node[shape = cds, style = dashed, color = red];
  layout[label  = layout];
  defStyleAttr[label = defStyleAttr];
  defStyleRes[label = defStyleRes];
  theme[label = theme];
  node[color = green, shape = box];
  attrs[label = attrs];
  node[color = purple]
  TypedArray[label = TypedArray];
  node[shape = line, color = transparent];
  id[label = attr_id];
  value[label = attr_value];

  rankdir = LR;
  {layout defStyleAttr defStyleRes theme} -> attrs;
  attrs -> TypedArray;
  id -> TypedArray -> value;
  {rankdir = LR; rank = same; id TypedArray value}
}

```

再使用这个函数的时候，不同的属性集合来源通过`attrs`目标集合过滤之后生成对应的`TypedArray`，然后通过属性的ID从这个对象里面获取到属性的对应值。

下面用一个示例来对上面不同的属性集合做一个具体的介绍。

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="CustomView">
        <attr name="attr_1" format="string"/>
    </declare-styleable>

    <attr name="CustomViewStyle" format="reference"/>
</resources>
```

上面的代码再`attrs.xml`文件总定义用于测试的属性`attr_1`，而另外一个属性`CustomViewStyle`会再`theme`中进行引用来指代`defStyleAttr`。

```xml
<resources>

    <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="CustomViewStyle">@style/CustomViewTheme</item>
        <item name="attr_1">From Theme</item>
    </style>

    <style name="CustomViewTheme">
        <item name="attr_1">From Theme.attr</item>
    </style>

    <style name="CustomViewDefault">
        <item name="attr_1">From Style</item>
    </style>

</resources>

```

上面是`styles.xml`中的代码，在这个代码中有三个地方需要说明。首先，在`theme`中为用于测试的`attr_1`属性赋特定值用来观测属性值的来源；然后在`theme`中使用`CustomViewStyle`属性作为`defStyleAttr`；最后定义一个`CustomViewDefault`样式表作为`defStyleRes`。

```xml
<?xml version="2.0" encoding="utf-8"?>
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
        app:attr_1="From Layout"/>
</LinearLayout>

```

最后在布局文件中直接为`attr_1`属性设置特定值，这样上面图例中提及到的所有属性集合的来源就都具备了。

在自定义控件中使用如下的代码测试不同情况的属性值获取状态。

```java

// 1. 直接使用控件构造函数出入的AttributeSet
TypedArray a = context.obtainStyledAttributes(set, R.styleable.CustomView, 0, 0);
String content = a.getString(R.styleable.CustomView_attr_1);    // content: From Layout

// 2. 使用Null作为参数
TypedArray a = context.obtainStyledAttributes(null, R.styleable.CustomView, 0, 0);
String content = a.getString(R.styleable.CustomView_attr_1);    // content: From Theme

// 3. 使用set的同时使用defStyleAttr和defStyleRes
TypedArray a = context.obtainStyledAttributes(set, R.styleable.CustomView, R.attr.CustomViewStyle, R.style.CustomViewDefault);
String content = a.getString(R.styleable.CustomView_attr_1);    // content: From Layout

// 4. 使用null的同时使用后两个参数
TypedArray a = context.obtainStyledAttributes(null, R.styleable.CustomView, R.attr.CustomViewStyle, R.style.CustomViewDefault);
String content = a.getString(R.styleable.CustomView_attr_1);    // content: From Theme.attr

// 5. 使用null的时候使用defStyleRes参数
TypedArray a = context.obtainStyledAttributes(null, R.styleable.CustomView, 0, R.style.CustomViewDefault);
String content = a.getString(R.styleable.CustomView_attr_1);    // content: From Style
```

所以从上面代码执行的结果可以分析出来，不同的属性集合在同时覆盖定义一个属性的时候，优先级遵循下面的一个顺序：

```java
Layout > defStyleAttr > defStyleRes > Theme
```
