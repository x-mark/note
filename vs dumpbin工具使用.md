# VS dumpbin 工具使用
vs提供的dumpbin工具可以很方便的帮助我们分析程序发布时会遇到的各种奇葩问题
## 常用参数
```java
dumpbin /headers xxx.dll //查看dll header 常用区分32/64

dumpbin /dependents xxx.dll //查看dll依赖关系

dumpbin /exports xxx.dll //查看dll导出表 常用于解决LINK 200 无法找到特定符号 错误

```


## 扩展
基于dumpbin可以实现一个类似dependency的程序依赖分析工具，来检测依赖树，以及32/64库错乱问题