
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
		- [插件状态](#插件状态)
		- [插件保存方式](#插件保存方式)

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
title 从服务器拉取插件配置进行插件更新流程

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
    elseif( 旧列表项需要更新？ ) then (yeas)
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
title 从服务器拉取插件配置进行插件更新流程

participant PluginDownloadManager
participant PluginConfigManager
participant DroidPluginManager
participant Utils

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
PluginDownloadManager -> Utils: emptyDir()
end
end

PluginDownloadManager -> PluginConfigManager: getRemotePluginList()
PluginConfigManager --> PluginDownloadManager: newPluginList
group newPluginList.foreach
PluginDownloadManager -> PluginDownloadManager: downloadPlugin()
end
```

上图表述的是向服务器请求该应用当前插件配置列表，并和当前本地保存的插件列表进行交叉对比，更新插件的流程。

在向服务器发送请求获取当前应用的插件配置列表的时候，需要携带下面的几个参数：
`appID`: 应用程序ID
`appVersion`: 对应的应用程序的版本
`market`：针对特定的市场渠道
`uid`：针对特定的uid

在成功查询到最新的配置列表之后，会将该配置列表保存到本地配置文件中然后对当前系统中的每个插件（Remote & Local）创建对应的`PlugInfo`对象。在完成这些工作之后，继续对最开始获取到的旧插件列表（Remote）进行校验，校验的内容包括该插件是否已经启用，该插件是否需要进行升级。完成旧版本插件列表校验之后将会执行最后的下载操作，针对新插件列表（Remote）中需要下载的进行下载操作。

在整个流程中，旧版本插件列表(Remote)的校验完成后主要执行的对于插件存储系统中保存的插件文件信息，因为`Plugin/PlugInfo`在最开始的时候就已经被更新掉。此外，整个下载更新的流程对于`Disenable`的插件不会做任何的限制，这个标识只会在插件安装，尝试使用插件的时候产生影响。

### 本地插件管理流程

```{puml}
title 本地插件管理流程

start
if( 启动插件的方式 ) then ( preLoad )
	: 加载指定的APK文件;

	partition 创建apk对应的Plugin {
		if ( apk文件存在 ) then (yeas)
			: 创建apk文件对应的packageinfo;
			if ( PackageInfo成功获取 ) then ( yeas )
				: 获取该文件的包名;
				: 根据包名尝试从本地缓存中获取Plugin;
				if ( Plugin 成功获取到) then ( yeas )
					: 返回获取到的缓存对象;
				else ( no )
					: 根据当前信息创建Plugin;
					: 将创建出来的对象保存到缓存中;
					: 返回新创建的对象;
				endif
			else ( no )
				: return null;
			endif
		else ( no )
			: return null;
		endif
	}

	partition 创建对应的PlugInfo {
		if ( Plugin == null ) then ( true )
			: 预加载发生错误;
			: return null;
		else ( false )
			: 尝试从缓存中获取对应的PlugInfo;
			if ( PlugInfo == null ) then ( yeas )
				: 为Plugin创建对应的PlguInfo;
				: 执行预加载操作;
				: 将创建的PlugInfo保存到缓存中;
				: 返回新建的PlugInfo;
			else ( no )
				: 返回获取到的缓存;
			endif
		endif
	}
else ( install )
	: 加载指定的apk文件;
	: 使用和Preload相同的方式尝试获取PlugInfo;
	if ( PlugInfo == null ) then ( yeas )
		: 安装本地插件发生错误;
		: return null;
	else ( no )
		: 执行install操作;
		: 保存到缓存中;
		: 返回获取到的PlugInfo;
	endif
endif
```

上图表述的是本地类型插件从无到有进入整个系统的过程。入口的途径有两条，按照具体的需求可以选择使用预加载或者是直接安装的两种方式，在这两条路径上，尝试创建`Plugin`的过程是共同的，而也就是在这个过程中将本地插件的相关信息保存到了本地的配置文件，同时也将对应的`APK文件`拷贝到了插件系统对应的存储路径下。所以在完成了上述操作之后，系统每次启动的时候都可以从配置文件中获取到本地插件的相关信息，而至此本地插件也就真正的进入到了当前应用的插件系统中。

之前在介绍从服务器获取插件配置列表并且更新本地插件的时候有提及到，在利用服务器获取到的新数据进行本地配置更新的时候会为每个插件创建对用的`PlugInfo`对象，这里的操作同样也包括了本地插件。这样做的目的是为了在插件系统启动的时候就完成了`Plugin`和`PlugInfo`的对应关系，方便后续的逻辑操作。

在本地插件真正进入到当前应用的插件系统之后，每次插件系统启动的时候除了从配置文件中获取本地插件的相关信息外，同时也会在插件的存储系统中进行扫描恢复所有已经安装完成的插件信息保存在一个`PluginPackageParser`列表中。虽然这个过程同样涉及到远端的插件，但是也在这里一并讲了。

```{puml}
title 已安装插件信息恢复

participant PluginManagerService
participant IPluginManagerImpl

[-> PluginManagerService: onCreate
PluginManagerService -> IPluginManagerImpl: onCreate()
activate IPluginManagerImpl
IPluginManagerImpl -> IPluginManagerImpl: onCreateInner()
activate IPluginManagerImpl
IPluginManagerImpl -> IPluginManagerImpl: laodAll()
activate IPluginManagerImpl
deactivate IPluginManagerImpl
deactivate IPluginManagerImpl
deactivate IPluginManagerImpl
```

```{puml}
title loadAllPlugin流程

start
: PluginDirHelper.getBaseDir;
while (对BaseDir进行遍历)
	if (dir.isDirectory && dir.length > 1)
		: 从插件目录中获取apk文件;
		: 保存到文件列表中;
	else (false)
		: continue;
	endif
endwhile
while(文件列表进行遍历)
	: 使用该文件创建对应PluginPackageParser;
	: 获取签名文件;
	: 保存PluginPackageParser到缓存中;
endwhile
stop
```

上图显示的从插件存储系统中恢复已安装插件信息的流程。在流程图中可以看到，这个恢复流程是从`PluginManagerService`启动的时候开始的，然后一只辗转到调用`IPluginManagerImpl.loadAllPlugin()`来执行恢复的具体操作。首先会从插件存储系统的保存目录中进行遍历，从其中找到所有的可能是安装完成的插件。在这里的判断条件有两个，第一是apk目录下必须存在对应的apk文件，第二个点是该插件的目录下除了apk目录之外应该还有其他的文件目录存在。额外增加第二点判断是因为在插件卸载之后仍然会把apk文件保留在原来的目录下面，如果这个时候不增加第二点判断条件，那么就会错误将已经卸载的插件当作安装完成的插件进行加载。

### 插件安装/删除/卸载流程

### 插件运行原理简介

### 插件状态

### 插件保存方式
