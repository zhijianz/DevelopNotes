> Maven 依赖本地内容的各种姿势

# 安装到本地仓库

# 使用 systemPath

```xml
<dependency>  
    <groupId>org.ibm</groupId>  
    <artifactId>jms</artifactId>  
    <version>1.0.0</version>  
    <scope>system</scope>  
    <systemPath>${project.basedir}/lib/jms.jar</systemPath>  
</dependency>  
```
上面的代码和声明普通的`dependency`差不多，只是多了`scope/systemPath`两个标签。`scope`标签指定为`system`和`provide`类似是都由系统来提供依赖，不同的地方在于需要使用`systemPath`标签来指定依赖存放的位置，就如同上面的例子展示的那样。

# 使用 Repository 标签
