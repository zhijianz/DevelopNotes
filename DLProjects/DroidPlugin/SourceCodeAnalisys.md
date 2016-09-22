# 关键类

```{puml}
class IPackageManagerImpl{
  插件操作相关接口提供类
  --
  - Map<String, PluginPackageParser> mPluginCache
  --
  + installPackage(String filePath, int flag)
}

class PluginPackageParser{

}

class PackageParser{

}

class PluginDirHelper{
  插件相关目录的工具类
}


IPackageManagerImpl -> PluginPackageParser
PluginPackageParser -> PackageParser
```

1. 从`IPackageManagerImpl.mPluginCache`的定义可以看出来，插件的构建过程依赖于关键类`PluginPackageParser`

# 关键过程

## install 流程

```{puml}
title PluginInstall 相关关键类图

class IPluginManagerImpl{

}

class PluginPackageParser{
  - Map<ComponentName, ActivityInfo> mActivityInfoCache
  - Map<ComponentName, ServiceInfo> mServiceInfoCache
  - Map<ComponentName, ProviderInfo> mProviderInfoCache
  ...
  ..
  在这个类里面缓存了通过parser解析出来的
  各种当前插件的数据缓存  
  其本身可以抽象为插件安装之后在整个插件系统
  中的逻辑代表
  其中具体的逻辑都是通过特定ApiLevel的PackageParser
  来实现
}

class PackageParser{

}

class PackageParserApiXXX{

}

IPluginManagerImpl -> PluginPackageParser
PluginPackageParser --> PackageParser
PackageParserApiXXX <- PackageParser

```

<br/>

```{puml}
title PluginInstall 流程图
participant PluginManager as pm
participant IPluginManagerImpl as ipmi
participant PluginPackageParser
participant PackageParser
participant PackageParserApi21

[-> pm: installPackage
activate pm

pm -> ipmi: installPackage
activate ipmi

ipmi -> ipmi: get plugin PackageInfo
ipmi -> ipmi: force stop the same package
ipmi -> Utils: copyFile copy apk to storage path

ipmi -> PluginPackageParser: new instance
activate PluginPackageParser
PluginPackageParser -> PackageParser: newPluginParser

PackageParser -> PackageParserApi21: new instance
activate PackageParserApi21
PackageParserApi21 -> PackageParserApi21: initClass
note right
加载几个关键的类
parser会解析manifest文件中的信息
end note
PackageParserApi21 --> PluginPackageParser: mParser
deactivate PackageParserApi21

PluginPackageParser -> PluginPackageParser: fixApplicationInfo
note right
为插件的Application配置各种文件路径
同时，在这个流程后面还会伴随着抓取
解析出来的四大组件信息内容
end note
deactivate PluginPackageParser

ipmi -> PluginPackageParser: collectCertification
```

## 多线程流程

```{puml}
participant PluginManagerService as pms
participant IPluginManagerImpl as ipmi
participant MyActivityManagerService as mams
participant StaticProcessList as spl

[-> pms: onCreate
pms -> ipmi: onCreate
activate ipmi
ipmi -> ipmi: onCreateInner
activate ipmi
ipmi -> ipmi: loadAllPlugin
ipmi -> mams: onCreate
deactivate ipmi
deactivate ipmi
mams -> spl: onCreate
activate spl
spl --> spl: addActivitiesInfo.. to add process item
deactivate spl
```

## Activity 启动流程

```{puml}
participant IActivityManagerHook as iamh
participant ProxyHook as ph
participant HookMethodHandler as hmh
participant IActivityManagerHookHandle as iamhh
participant RunningActivities as ra

[-> iamh: invoke for startActivity
activate iamh
iamh -> ph: invoke
activate ph
ph -> hmh: doHookInner
activate hmh
hmh -> iamhh: beforeInvoke
activate iamhh
iamhh -> ra: beforeStartActivitys
note right
在启动一个新的Activity之前会对其LaunchMode进行检查
从而找到合适的Activity
end note

iamhh -> iamhh: doReplaceIntentForStartActivityAPIHigh
note right
intent 马甲
end note
```

## 留白
