```{puml}
class DexClassLoader{

}

class BaseDexClassLoader{

}

class ClassLoader{

}

DexClassLoader -|> BaseDexClassLoader
BaseDexClassLoader -|> ClassLoader
```
