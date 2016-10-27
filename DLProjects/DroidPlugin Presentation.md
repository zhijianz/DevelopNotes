
<!-- toc orderedList:0 -->

- [DroidPlugin Manager](#droidplugin-manager)
- [使用示例](#使用示例)
	- [关键接口调用](#关键接口调用)
- [代码结构](#代码结构)
- [服务器交互逻辑](#服务器交互逻辑)
	- [Plugin](#plugin)
	- [拉取插件信息](#拉取插件信息)
- [DroidPlugin 关键逻辑](#droidplugin-关键逻辑)
	- [PluginPackageParser](#pluginpackageparser)
	- [插件安装流程](#插件安装流程)
	- [killBackgroundProcesses](#killbackgroundprocesses)
	- [fixApplicationInfo](#fixapplicationinfo)
- [开放接口部分关键逻辑](#开放接口部分关键逻辑)
	- [PlugInfo](#pluginfo)
	- [预装载插件](#预装载插件)
	- [安装本地插件](#安装本地插件)
	- [卸载和删除插件](#卸载和删除插件)
- [END](#end)

<!-- tocstop -->


<!-- slide -->
# DroidPlugin Manager

- 示例
- 代码结构
- 服务器交互逻辑
- DroidPlugin 关键逻辑
- 公开接口

<!-- slide -->
# 使用示例

- 依赖
- 服务器插件配置
- 本地插件
- 关键接口调用

<!-- slide -->
## 关键接口调用

1. 使用`PluginApplication`或者调用下面两个函数初始化DroidPlugin
```java
1. PluginHelper.getInstance().applicationOnCreate(this);
2. PluginHelper.getInstance().applicationAttachBaseContext(base);
```

2. 调用`DroidPlugin.init(context, appId, appVer, market, uid)`执行插件管理系统的出事操作

3. 调用对应`API`完成响应插件操作

<!-- slide -->
# 代码结构
```{puml}
package "Plugin Server Logic" {
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
package "DroidPlugin Logic"{
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
<!-- - 插件启动替换过程 -->

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
## killBackgroundProcesses

```java
public boolean killBackgroundProcesses(String pluginPackageName) throws RemoteException {
	ActivityManager am = (ActivityManager) mContext.getSystemService(Context.ACTIVITY_SERVICE);
	List<RunningAppProcessInfo> infos = am.getRunningAppProcesses();
	boolean success = false;
	for (RunningAppProcessInfo info : infos) {
			if (info.pkgList != null) {
					String[] pkgListCopy = Arrays.copyOf(info.pkgList, info.pkgList.length);
					Arrays.sort(pkgListCopy);
					if (Arrays.binarySearch(pkgListCopy, pluginPackageName) >= 0 && info.pid != android.os.Process.myPid()) {
							Log.i(TAG, "killBackgroundProcesses(%s),pkgList=%s,pid=%s", pluginPackageName, Arrays.toString(info.pkgList), info.pid);
							android.os.Process.killProcess(info.pid);
							success = true;
					}
			}
	}
	return success;
}
```

<!-- slide -->
## fixApplicationInfo

针对于获取到的每个组件，配置其ApplicationInfo的对应路径

```javascript
applicationInfo.sourceDir = mPluginFile.getPath();
applicationInfo.publicSourceDir = mPluginFile.getPath();
applicationInfo.dataDir = PluginDirHelper.getPluginDataDir(getPluginBaseDirName(), mPackageName);
applicationInfo.nativeLibraryDir = PluginDirHelper.getPluginNativeLibraryDir(getPluginBaseDirName());
applicationInfo.splitSourceDirs = new String[]{mPluginFile.getPath()};
applicationInfo.splitPublicSourceDirs = new String[]{mPluginFile.getPath()};
applicationInfo.processName = applicationInfo.packageName;
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
# 开放接口部分关键逻辑

- PlugInfo
- 开放接口
  - 预装载插件
  - 安装本地插件
  - 卸载和删除插件

<!-- slide -->
## PlugInfo

对于插件在系统中运行状态的管理

```{puml}
class PlugInfo{

}

class Plugin{

}

enum STATUS {
  NONE
  PRELOADED
  INSTALLED
  RUNNING
}

PlugInfo -> STATUS
PlugInfo --> Plugin
```

<!-- slide -->

## 预装载插件

- 为了在创建为安装之前快速的获取插件信息用于显示

- 预装载的过程对于本地插件和服务器配置插件来说差别仅仅在于本地插件需要自行创建对应的`Plugin`而服务器配置插件不需要。所以将预装载流程分成`Plugin`获取和预装载信息获取两个部分

<!-- slide -->

```{puml}
title 本地插件创建Plugin流程

participant PluginConfigManager
participant Context
participant PackageManager
participant Plugin

[-> PluginConfigManager: getPlugin(originPath, context, replaceFlag)
activate PluginConfigManager
PluginConfigManager -> Context: getPackageManager()
Context --> PluginConfigManager: pm
PluginConfigManager -> PackageManager: getPackageArchiveInfo(originPath, 0)
PackageManager --> PluginConfigManager: pi
note left: can get some information about apk
PluginConfigManager -> PluginConfigManager: try to get plugin from cache
alt plugin not in cache
PluginConfigManager -> Plugin: new Plugin()
Plugin --> PluginConfigManager: plugin
PluginConfigManager -> PluginConfigManager: save to cache
PluginConfigManager -> PluginConfigManager: writeToFile
note left: save the new plugin to SharePreference
else plugin in cache
PluginConfigManager -> PluginConfigManager: return cache plugin
end
```

<!-- slide -->
预装载操作的关键代码

```java
PackageManager pm = context.getPackageManager();
PackageInfo pi = pm.getPackageArchiveInfo(apkFile.getAbsolutePath(), 0);
if (pi == null){
    Log.e(TAG, "doPreLoad can not get PackageInfo for apk: " + apkFile.getAbsolutePath());
    return false;
}
pi.applicationInfo.sourceDir = apkFile.getAbsolutePath();
pi.applicationInfo.publicSourceDir = apkFile.getAbsolutePath();
label = (String) pi.applicationInfo.loadLabel(pm);
icon = pi.applicationInfo.loadIcon(pm);
pluginStatus = STATUS.PRELOADED;
```

<!-- slide -->
## 安装本地插件

插件安装操作和预加载的类似，同样存在本地插件和服务器配置插件的差别，并且这些差别都是体现在`Plugin`的获取方式上。在获取完全本地插件对应的`Plugin`对象之后采用于上面介绍的安装流程相同。

<!-- slide -->
## 卸载和删除插件

卸载和删除插件的过程本质上是对插件相关的删除操作。两者之间的区别在于卸载插件仅仅是删除插件安装过程解析到本地存储的内容同时将插件的运行状态恢复到正确的位置；而对于插件的删除操作来说，插件在当前应用程序中所有的数据都已经是无用的，所以都不会得到保留。

```{viz}
digraph deleteAnduninstall {
  // 定义节点
  node[shape = box, style = dashed, color = blue];
  unisntall[label = unisntall];
  delete[label = delete];
  node[shape = folder, color = red];
  SharePreference[label = SharePreference];
  PluginDir[label = PluginDir];
  node[shape = cds, color = green];
  Plugin[label = Plugin];
  PlugInfo[label = PlugInfo];
  PluginPackageParser[label = PluginPackageParser];

  // 关系
  unisntall -> {PluginDir PluginPackageParser}[color = slategrey, style = dashed];
  delete -> {SharePreference PluginDir Plugin PlugInfo PluginPackageParser}[color = purple, style = dashed];
}
```

<!-- slide -->

# END

Thanks for watching :)
