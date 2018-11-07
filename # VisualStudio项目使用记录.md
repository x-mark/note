# VisualStudio 使用记录

一直习惯使用Qt qmake的一套构建系统，使用vs的工程管理的一些问题记录一下：

## win32 control程序使用MFC库

报错信息：

```java
1>C:\Program Files (x86)\Microsoft Visual Studio 9.0\VC\atlmfc\include\afx.h(24) : fatal error C1189: #error :  Building MFC application with /MD[d] (CRT dll version) requires MFC shared dll version. Please #define _AFXDLL or do not use /MD[d]

```
解决办法：
项目->属性->常规->项目默认->MFC的使用->在共享DLL中使用MFC

## 一个库导出符号匹配问题

错误信息：

```java
1>NMXFtpClientTest.obj : error LNK2001: 无法解析的外部符号 "__declspec(dllimport) public: int __thiscall CNMXFtpFileFinder::ConnectionFtp(class ATL::CStringT<wchar_t,class StrTraitMFC_DLL<wchar_t,class ATL::ChTraitsCRT<wchar_t> > >,class ATL::CStringT<wchar_t,class StrTraitMFC_DLL<wchar_t,class ATL::ChTraitsCRT<wchar_t> > >,class ATL::CStringT<wchar_t,class StrTraitMFC_DLL<wchar_t,class ATL::ChTraitsCRT<wchar_t> > >,unsigned short)" (__imp_?ConnectionFtp@CNMXFtpFileFinder@@QAEHV?$CStringT@_WV?$StrTraitMFC_DLL@_WV?$ChTraitsCRT@_W@ATL@@@@@ATL@@00G@Z)
```

常见的无法解析外部符号问题：

1. 检查lib依赖的设置。 
    * 项目-属性-链接器-常规-附加库目录 添加lib所在文件夹路径
    * 项目-属性-链接器-输入-附加依赖项 添加依赖的lib文件名
2. 库导出符号是否匹配
    * 使用VS的dumpbin 命令查看lib的导出表 dumpbin /LINKERMEMBER xxx.lib > 1.txt
    * 搜索导出表中是否有所需要的函数
  在lib导出表中搜索：

  ```java
  __imp_?ConnectionFtp@CNMXFtpFileFinder@@QAEHV?$CStringT@GV?$StrTraitMFC_DLL@GV?$ChTraitsCRT@G@ATL@@@@@ATL@@00G@Z
  ```

  发现lib导出的函数签名中是@GV而程序中无法找到的函数签名是@WV因此怀疑是lib项目和应用项目设置不匹配
  最终找到：项目-属性-C++-语言-将wchar_t视为内置类型 lib中设置的为否 应用中默认为是 将应用中改为否

## windbg调试

在win下可以使用windbg来调试编译好的exe

1. windbg文件菜单中设置符号文件(.pdb)路径,比如C:\\symbolslocal文件夹，同时配置微软符号服务器如果没有系统符号会自动下载

```shell
C:\symbolslocal; SRV*C:\symbolslocal*http://msdl.microsoft.com/download/symbols
```

2. 可以使用::OutputDebugString(CString) win API进行调试信息输出，这些调试信息会windbg中打印出来