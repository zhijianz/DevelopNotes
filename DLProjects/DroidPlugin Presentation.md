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

插件基本配置信息的抽象

```javascript
Plugin {
  pluginID, pluginVer, url,
  shal, size, desc,
  checkSign, packageName, enable
}
```

```{viz}
digraph InformationDerection {
  node[shape = box, style = dashed, color = blue, with = 1.3];
  local[label = LocalApk];
  server[label = ServerPlugin]
  node[shape = cds, style = dashed, color = red, height = .6];
  Plugin[label = Plugin]
  PlugInfo[label = PlugInfo]
  PluginPackageParser[label = PluginPackageParser]

  randir = RL;
  local -> Plugin;
  server -> Plugin;

}
```

<!-- slide -->
# DroidPlugin 关键逻辑

- PluginPackageParser
- 插件安装过程
- 插件启动替换过程

<!-- slide -->
# 开放接口和部分关键逻辑

- PlugInfo
- 开放接口
  - 安装本地插件
  - 删除插件
  - 卸载插件
