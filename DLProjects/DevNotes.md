# Plan

现在有如下四个参考技术方案，需要在周三的时候提出一个技术结论

## 方案1

使用代理activity的方式，项目地址 [代理Activty](https://github.com/singwhatiwanna/dynamic-load-apk)

搁浅，工作重复

## 方案2

ing
使用动态生成Activity的方式，项目地址 [Activity动态生成](https://github.com/houkx/android-pluginmgr)

已经成功将Demo运行起来，现在剩下两个任务

1. 阅读源码理解原理
2. 写一个覆盖率够高的测试demo，plugin包含下面的内容`Activity/Fragment/Gradle依赖/SO/其他的重要组件/和宿主交互`

## 方案3

[Small项目](https://github.com/wequick/Small)

后续观察

## 方案4

[Frontia项目](https://github.com/zhijianz/android-dynamical-loading)

遗弃

# Basic Knowledge

1. Context
2. Resource
3. Package
4. The process of launch activity

# Current Work

## 加载优化

# BugFixed

## Inflate & Resources

为了可以让宿主可以获取到自己的运行时上下文和资源，在开发的过程中，我们实现了一个自定义的`PluginContext`并且重写其中的`getResources()`函数来返回自定义的`Resources`对象。这种方法可以基本可以应对所以显示获取资源的使用场景，但是在使用`Inflater`来创建`View`的时候会出现`ResourcesNotFound`的错误。在正常开发的情况下，出现这种问题唯一的可能性就是`Resources`对象并不是当前插件项目对应的资源管理对象而是错误的将主程序的资源管理对象传入了进来。这就表明在`Inflate`的过程中使用了`getResources`之外的途径去获取当前插件程序的资源管理对象。在这个流程中资源的获取是通过`TypedArray`来完成的，而这个`TypedArray`对象使用下面的代码进行创建

```java
// Context.java
public final TypedArray obtainStyledAttributes(
        @StyleRes int resid, @StyleableRes int[] attrs) throws Resources.NotFoundException {
    return getTheme().obtainStyledAttributes(resid, attrs);
}

// ContextImpl
public Resources.Theme getTheme() {
    if (mTheme != null) {
        return mTheme;
    }

    mThemeResource = Resources.selectDefaultTheme(mThemeResource,
            getOuterContext().getApplicationInfo().targetSdkVersion);
    initializeTheme();

    return mTheme;
}

private void initializeTheme() {
    if (mTheme == null) {
        mTheme = mResources.newTheme();
    }
    mTheme.applyStyle(mThemeResource, true);
}

public TypedArray obtainStyledAttributes(AttributeSet set,
     @StyleableRes int[] attrs, @AttrRes int defStyleAttr, @StyleRes int defStyleRes) {
   final int len = attrs.length;
   final TypedArray array = TypedArray.obtain(Resources.this, len);
   ...
}
```

分析上面的代码，`TypedArray`来自于`mTheme`的创建，而`mTheme`的来源有两种不同的路径。对于`Activity`来说，在创建的某个阶段就会创建自己的主题，这个主题中携带的资源对象是主程序的；另外一种来源是通过`ContextImpl.mResources`对象进行创建，在之前有提到过插件自定义的`Context`只是重写了`getResources`方法返回正确的资源对象并没有对`mResources`变量进行改动，所以上面的源码中可以分析解决`ResourcesNotFound`错误有两个地方需要去更改，一个是`Activity.mTheme`另外一个则是`ContextImpl.mResources`。现在项目里对于这个问题，前者是直接通过反射将其置为`null`，后者则是通过反射设置成插件的资源对象。
