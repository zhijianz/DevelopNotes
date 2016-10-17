<!-- slide -->
# DroidPlugin Manager

- 代码结构
- 服务器交互逻辑
- DroidPlugin 关键逻辑
- 公开接口
- 使用示例

<!-- slide -->
# 代码结构
```{puml}
package "Plugin Sever Controll" {
  class PluginDownloadManager
  class PluginConfigManager
  class Plugin
}

package "Open API"{
  class DroidPluginManager
  class PlugInfo
}
```
```{puml}
package "DroidPlugin logic"{
  class PluginManager
  class IPluginManagerImpl
  class PluginPackageParser  
}
```

<!-- slide -->
# 服务器交互逻辑

- Plugin
- 拉取插件信息

<!-- slide -->
## Plugin

- 插件基本配置信息的抽象

```javascript
Plugin {
  pluginID, pluginVer, url,
  shal, size, desc,
  checkSign, packageName, enable
}
```

- 插件信息流向

```{viz}
digraph InformationDerection {
  edge[penwidth = .8, color = saltegray]
  node[shape = box, style = dashed, color = blue, with = 1.3];
  local[label = LocalApk];
  server[label = ServerPlugin]
  node[shape = cds, style = dashed, color = red, height = .6];
  Plugin[label = Plugin]
  PlugInfo[label = PlugInfo]
  PluginPackageParser[label = PluginPackageParser]
  SharePreference[label = SharePreference, shape = folder, color = green];

  rankdir = LR;
  local -> Plugin;
  server -> Plugin;
  Plugin -> PlugInfo;
  PlugInfo -> PluginPackageParser;
  Plugin -> SharePreference;
  {rankdir = TB; rank = same; Plugin, SharePreference};
}
```

<!-- slide -->
## 拉取插件信息

- 请求的参数

`appID`, `appVersion`, `market`, `uid`

- 调用接口

`PluginDownloadManager.queryPlugin(appID, appVersion, market, uid)`

- 执行流程

<!-- slide -->
```{puml}
if( 请求插件配置列表成功 ) then (yeas)
  : 获取现有插件列表;
  if ( 顺序执行 ) then ( step:1 )
    partition 利于服务器返回结果更新现有配置 {
      while( 对服务器返回结果进行遍历 )
        : 创建对应Plugin & PluginInfo;
      endwhile
      while( 对本地插件数据列表进行遍历 )
        : 创建对应PlugInfo;
      endwhile
    }
  elseif( step:2 )
    partition 处理旧插件列表 {
      while ( 遍历之前获取的旧列表 )
        if( 需要更新？ )
          : 进行更新操作;
        elseif ( 无用插件？ )
          : 进行删除操作;
        endif
      endwhile
    }
  elseif( step:3 )
   partition 下载 {
     while( 遍历新的插件列表)
       if ( 需要下载 )
         : 进行下载操作;
       endif
     endwhile
   }
  endif
else (no)
  stop
```

<!-- slide -->
# DroidPlugin 关键逻辑

- PluginPackageParser
- 插件安装过程
- 插件启动替换过程

<!-- slide -->
## PluginPackageParser

插件安装内容的抽象和安装的实际执行者

```{puml}
class PackageInfo{

}

class PluginPackageParser{
	- final File mPluginFile
	- final PackageParser mParser
	- final String mPackageName
	- final Context mHostContext
	- final PackageInfo mHostPackageInfo
	..
	1. 在这个类的构造函数过程中，会从创建的
	PackageParser中获取到关于apk的各种组件
	的信息
}

class PackageParser{

}

PluginPackageParser -> PackageParser
PackageInfo <- PluginPackageParser
```

<!-- slide -->
## 插件安装流程
```{puml}
hide footbox
participant IPluginManagerImpl
participant Context
participant PackageManager
participant PluginPackageParser
participant BaseActivityManagerService

[-> IPluginManagerImpl: installPlugin(pluginFile)
activate IPluginManagerImpl
IPluginManagerImpl -> Context: getPackageManager()
Context --> IPluginManagerImpl: pm
IPluginManagerImpl -> PackageManager: getPackageArchiveInfo(pluginFile, 0)
PackageManager --> IPluginManagerImpl: pi
IPluginManagerImpl -> IPluginManagerImpl: forceStopPackage(packageName)
IPluginManagerImpl -> PluginPackageParser: new PluginPackageParser()
PluginPackageParser --> IPluginManagerImpl: parser
IPluginManagerImpl -> PluginPackageParser: collectCertification()
IPluginManagerImpl -> PluginPackageParser: 检查权限是否允许运行
IPluginManagerImpl -> PluginPackageParser: 获取签名文件
IPluginManagerImpl -> IPluginManagerImpl: copyNativeLibs(parser)
IPluginManagerImpl -> IPluginManagerImpl: dexOpt(context, parser)
IPluginManagerImpl -> IPluginManagerImpl: save parser to cache
IPluginManagerImpl -> BaseActivityManagerService: onPkgInstalled(parserCache, parser, pkg)
IPluginManagerImpl -> IPluginManagerImpl: sendInstalledBroadcast(packageName)
deactivate IPluginManagerImpl
```

<!-- slide -->

```{puml}
hide footbox

participant PluginPackageParser
participant PackageParser
participant ComponentName

[-> PluginPackageParser: new PluginPackageParser(context, pluginFile)
activate PluginPackageParser
PluginPackageParser -> PackageParser: newPluginParser(context)
activate PackageParser
PackageParser --> PluginPackageParser: mParser
deactivate PackageParser

== 获取Activity相关信息 ==
PluginPackageParser -> PackageParser: getActivities()
activate PackageParser
activate PluginPackageParser
PackageParser --> PluginPackageParser: datas
deactivate PackageParser

group datas.loop
PluginPackageParser -> ComponentName: new ComponentName(packageName, mParser.readNameFromComponent(data))
activate ComponentName
ComponentName --> PluginPackageParser: componentName
deactivate ComponentName

PluginPackageParser -> PluginPackageParser: mActivityObjCache.put(componentName, data)
PluginPackageParser -> PackageParser: generateActivityInfo(data, 0)
activate PackageParser
PackageParser --> PluginPackageParser: value
deactivate PackageParser
PluginPackageParser -> PluginPackageParser: fixApplicationInfo(value.application)
PluginPackageParser -> PluginPackageParser: mActivityInfoCache.put(componentName, value)

PluginPackageParser -> PackageParser: readIntentFilterFromComponent(data)
activate PackageParser
PackageParser --> PluginPackageParser: filters
deactivate PackageParser
PluginPackageParser -> PluginPackageParser: mActivityIntentFilterCache.remove(componentName)
PluginPackageParser -> PluginPackageParser: mactivityintentFilterCache.put(componentName, filters)

end

PluginPackageParser -> PluginPackageParser: decode info from datas
deactivate PluginPackageParser

== 获取Service相关信息 ==
|||
== 获取Provider相关信息 ==
|||
== 获取Receivers相关信息 ==
|||
== 获取Insturmentation相关信息 ==
|||
== 获取Permission相关信息 ==

```

<!-- slide -->
# 开放接口和部分关键逻辑

- PlugInfo
- 开放接口
  - 安装本地插件
  - 删除插件
  - 卸载插件
