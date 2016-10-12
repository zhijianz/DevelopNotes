
> 介绍插件项目DroidPlugin版本的使用和相关代码分析

<!-- toc orderedList:0 -->

- [Abstract](#abstract)
- [源代码分析](#源代码分析)
	- [源代码框架](#源代码框架)
	- [关键流程](#关键流程)
		- [从服务器拉取插件配置和更新本地插件数据流程](#从服务器拉取插件配置和更新本地插件数据流程)
		- [本地插件管理流程](#本地插件管理流程)
		- [插件安装/删除/卸载流程](#插件安装删除卸载流程)
		- [插件运行原理简介](#插件运行原理简介)

<!-- tocstop -->

# Abstract

1. 用一个示例项目介绍SDK的使用
2. 介绍SDK代码结果和关键代码分析

# 源代码分析

## 源代码框架

```{puml}
package "Plugin Sever Controll" #dddddd{
  class PluginDownloadManager
  class PluginConfigManager
  class Plugin
}

package "Open API"{
  class DroidPluginManager
  class PlugInfo
}

package "DroidPlugin logic"{
  class PluginManager
  class IPluginManagerImpl
  class PluginPackageParser  
}
```
上面的类图大体表述了插件系统的代码组织结构。

`Plugin Server Controll`部分的代码负责和服务器进行交互从服务器上拉取当前应用对应的插件配置，根据这些配置对系统当前使用的配置进行校验（插件更新以及删除无用的插件，不包含本地插件）。而对于本地插件，这里只负责对本地插件信息进行保存和创建对应的`Plugin`对象。

`DroidPlugin Logic`是原本`DroidPlugin`开源库的逻辑内容。其中包含插件安装的实际操作，插件运行的原理内容。

`Open API`部分对整个SDK的功能进行抽象并开放对应的逻辑接口给使用SDK的应用程序。

## 关键流程

### 从服务器拉取插件配置和更新本地插件数据流程

```{puml}
start

: 向服务器请求当前应用插件配置，参数：appID, appVer, market, uid;

if( 请求成功？ ) then (success)
  : 解析 json 获取最新远端插件列表;
  : 获取原有的插件列表;
  : 将新插件列表保存到本地
  并创建每个插件对应的PlugInfo;
  : 获取本地插件列表并对每个
  插件创建对应的PlugInfo;
  while( 对旧列表项进行遍历 )
    if( 旧列表项可以删除？ ) then (yeas)
      : 删除旧列表项;
    elseif( 旧列表项需要更新？ ) them (yeas)
      : 更新旧列表项;
    else
      : 下载对应插件;
    endif
  endwhile
  end
else (fail)
  end
```

```{puml}
participant PluginDownloadManager
participant PluginConfigManager
participant DroidPluginManager
participant PluginManager
participant PlugInfo
participant Plugin

[-> PluginConfigManager: readFromFile()

[-> PluginDownloadManager: queryPlugin(appID, appVer, market, uid)
activate PluginDownloadManager

PluginDownloadManager -> PluginConfigManager: getRemotePluginList()
activate PluginConfigManager
PluginConfigManager --> PluginDownloadManager: oldPluginList
deactivate PluginConfigManager

PluginDownloadManager -> PluginConfigManager: updatePluginFromServer
activate PluginConfigManager
PluginConfigManager -> DroidPluginManager: create PlugInfo (Remote & Local)
PluginConfigManager -> PluginConfigManager: writeToFile
deactivate PluginConfigManager

group oldPluginList.foreach
alt isPluginNeedDelete
PluginDownloadManager -> DroidPluginManager: deletePlugin()
else isPluginNeedUpdate
PluginDownloadManager -> DroidPluginManager: removePlugInfoFromCache
end
end
```

上图表述的是向服务器请求该应用当前插件配置列表，并和当前本地保存的插件列表进行交叉对比，更新插件的流程。

### 本地插件管理流程

### 插件安装/删除/卸载流程

### 插件运行原理简介
