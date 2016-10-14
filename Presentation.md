
> 介绍插件项目DroidPlugin版本的使用和相关代码分析

<!-- toc orderedList:0 -->

- [Abstract](#abstract)
- [源代码分析](#源代码分析)
	- [源代码框架](#源代码框架)
	- [关键流程](#关键流程)
		- [从服务器拉取插件配置和更新本地插件数据流程](#从服务器拉取插件配置和更新本地插件数据流程)
		- [已安装插件信息恢复流程](#已安装插件信息恢复流程)
		- [插件预装载流程](#插件预装载流程)
		- [插件安装流程](#插件安装流程)
		- [插件实体对应关系](#插件实体对应关系)
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

### 已安装插件信息恢复流程

```{puml}
title  已安装插件信息恢复流程

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

上图显示的从插件存储系统中恢复已安装插件信息的流程。在流程图中可以看到，这个恢复流程是从`PluginManagerService`启动的时候开始的，然后一只辗转到调用`IPluginManagerImpl.loadAllPlugin()`来执行恢复的具体操作。首先会从插件存储系统的保存目录中进行遍历，从其中找到所有的可能是安装完成的插件。在这里的判断条件有两个，第一是apk目录下必须存在对应的apk文件，第二个点是该插件的目录下除了apk目录之外应该还有其他的文件目录存在。额外增加第二点判断是因为在插件卸载之后仍然会把apk文件保留在原来的目录下面，如果这个时候不增加第二点判断条件，那么就会错误地将已经卸载的插件当作安装完成的插件进行加载。

这里还存在一个没有弄清楚地方就是关于签名文件，包括整个签名文件地获取，保存，使用地意义等等。

### 插件预装载流程

```{puml}
title 插件预装载流程

start
if (预装载插件类型) then (Local)
	: 使用对应APK文件创建对应的Plugin;
	if (Plugin == null) then (yeas)
		: 本地预加载错误;
		: return null;
	else (no)
		: 根据获取的Plugin从缓存中获取PlugInfo;
		if (PlugInfo == null) then (yeas)
			: 创建新的PlugInfo;
			: 保存到本地缓存中;
			: 执行PlugInfo的预加载操作;
		else (no)
			: 使用缓存中获取的PlugInfo执行预加载操作;
		endif
	endif
else (Remote)
	: 尝试从缓存中获取PlugInfo;
	if (PlugInfo == null)	then (yeas)
		: 从缓存中获取对应地Plugin;
		if (Plugin == null) then (yeas)
			: 预加载错误;
		else (no)
			: 使用Plugin创建对应地PlugInfo;
			: 保存PlugInfo到缓存中;
			: 执行PlugInfo的预加载操作;
		endif
	else (no)
		: 执行PlugInfo的预加载操作;
	endif
endif
```

关于插件预装载，首要说明的是执行这个过程的意义。在当前系统中存在多个插件，或者是应用希望同时展现出一个可选择使用的插件列表的时候，就需要从插件中获取到用于展示的基本信息。按照和服务器定义的`Plugin`实体的配置信息本来是可以完成这个工作，但是最开始没有注意到这一部分的内容，而且也要兼容到本地插件的预装载信息。所以在这里采用的方案是直接对插件的APK文件进行解析，目的是为了获取到插件的`应用名称\应用图标`用于插件列表的展示。这部分的实在操作封装在`PlugInfo`中，关键代码如下:

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

代码的核心部分是尝试获取当前APK文件的`PackageInfo`，如果这个获取操作失败就直接判定当前的APK文件是不可用的，成功获取之后就可以直接从其中获取需要的内容。

### 插件安装流程

插件的安装虽然会有本地插件安装和远端服务器配置插件安装两种流程，但其实两种方式不同点仅仅在于本地插件安装的时候，最开始本地插件是不存在与之对应的`Plugin`信息的，在这个方面的缺陷就需要手动的从该插件构建出对应的`Plugin`并切保存到本地配置文件中。在Plugin创建完成之后，两种不同类型的插件安装过程实际上就合并到了同样的一条操作路径中。所以在介绍关于插件安装的过程时分成两个部分的内容，第一个是为本地插件创建对应`Plugin`的过程；第二个则是`PlugInfo`作为起点的插件安装路径。

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

在这个流程中，最基本的条件是可以从`originPath`指定目录下获取到该文件`PackageInfo`。在成功获取到`PackageInfo`之后可以使用对应的包名尝试在缓存中获取到对应`Plugin`对象，当该对象不存在的时候就会尝试使用现有的信息来构建`Plugin`对象。这些现有的填充出来的本地`Plugin`对象应该是下面的这种数据结构:

```javascript
Plugin {
	pluginID: LOCAL_ID;
	url: originPath;
	enable: true;
	packageName: packageName
}
```

因为该插件信息是从本地生成的，所以配置的是一个统一的LOCAL_ID；然后会在url中保存APK文件原始放置路径；最后packageName则是直接从PackageInfo中解析出来；当然，对于所有的本地插件来说，enable字段都是true值。

只要配置完对应的`Plugin`对象之后，这个流程基本上算是完成了，后面只需要把新生成的对象保存到合适的位置。但是上面的流程图中对于`replaceFlag`参数的时候并没有表述出来，在这里需要做一个补充说明。因为考虑到本地插件不能够向远端服务器下载的插件那样可以通过插件的ID和版本号来进行更新操作，所以在开这个接口的时候留下了一个`replaceFlag`参数来配置在缓存中已经存在对应包名插件的时候采取怎样的操作，可以是直接返回不替代也可以删除旧的插件采用替代的方式导入新的插件。

在完成本地插件信息获取过程分析之后，现在来介绍插件的具体安装过程。

```{puml}
title 插件安装流程

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

整个插件的安装流程基本上采用的还是`DroidPlugin`原来的那一套，在最开始的部分仍然需要获取到APK文件的包名，然后通过这个包名尝试关闭当前在运行的同包名的插件。关闭同名插件的关键代码如下：

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

上面的代码遍历了正在运行的进程并通过匹配插件的包名来确定需要关掉的进程来达到停止正在运行的同名插件的目的。

在准备上面的准备工作完成之后，会去创建在插件安装工程中的一个关键类`PluginPackageParser`，整个插件的的安装工作基本上在这个类的构造函数中就执行的差不多了，而且后续插件使用的过程中也无法避免的要和这个类进行交互。

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

```{puml}
title PluginPackageParser 创建流程

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

`PluginPackageParser`类主要依赖`PackageParser`类来获取插件的各种相关信息。这些信息的获取操作如上图所示，在`PluginPackageParser`的构造函数中创建完`PackageParser`对象之后就开始执行插件信息获取的操作，其实整个插件的安装工作大部分都是从插件插件中解析出这些信息内容，所以在整个构造函数执行完毕的时候，插件的安装过程已经完成了一大半。因为获取各种信息的操作基本差不多所以在流程图中就采用了比较简略的写法来表述，具体的信息的可以查看相关的代码。在这个流程中还有一个`fixApplicationInfo`函数需要详细看一下:

```java
private ApplicationInfo fixApplicationInfo(ApplicationInfo applicationInfo) {
		if (applicationInfo.sourceDir == null) {
				applicationInfo.sourceDir = mPluginFile.getPath();
		}
		if (applicationInfo.publicSourceDir == null) {
				applicationInfo.publicSourceDir = mPluginFile.getPath();
		}

		if (applicationInfo.dataDir == null) {
				applicationInfo.dataDir = PluginDirHelper.getPluginDataDir(getPluginBaseDirName(), mPackageName);
		}

		try {
				if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
						if (FieldUtils.readField(applicationInfo, "scanSourceDir", true) == null) {
								FieldUtils.writeField(applicationInfo, "scanSourceDir", applicationInfo.dataDir, true);
						}
				}
		} catch (Throwable e) {
				//Do nothing
		}

		try {
				if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
						if (FieldUtils.readField(applicationInfo, "scanPublicSourceDir", true) == null) {
								FieldUtils.writeField(applicationInfo, "scanPublicSourceDir", applicationInfo.dataDir, true);
						}
				}
		} catch (Throwable e) {
				//Do nothing
		}

		applicationInfo.uid = mHostPackageInfo.applicationInfo.uid;

		if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.GINGERBREAD) {
				if (applicationInfo.nativeLibraryDir == null) {
						applicationInfo.nativeLibraryDir = PluginDirHelper.getPluginNativeLibraryDir(getPluginBaseDirName());
				}
		}

		if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
				if (applicationInfo.splitSourceDirs == null) {
						applicationInfo.splitSourceDirs = new String[]{mPluginFile.getPath()};
				}
		}

		if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
				if (applicationInfo.splitPublicSourceDirs == null) {
						applicationInfo.splitPublicSourceDirs = new String[]{mPluginFile.getPath()};
				}
		}

		if (TextUtils.isEmpty(applicationInfo.processName)) {
				applicationInfo.processName = applicationInfo.packageName;
		}
		return applicationInfo;
}
```

在构造函数获取插件信息的过程中，会调用该函数对所有组件相对应的`Application`进行设置，在这个设置的过程中，会将`Application`中各种路径参数对应到我们自己插件系统中具体路径。

### 插件实体对应关系

### 插件运行原理简介

### 插件状态

### 插件保存方式
