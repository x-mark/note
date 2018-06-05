# NPAPI·ActiveX·NaCl·PPAPI·Emscripten·WebAssembly

以下内容整理自wikipedia和相关项目主页

## NPAPI(淘汰)
网景插件应用程序接口（英语：Netscape Plugin Application Programming Interface，缩写：NPAPI）是一个跨平台的通用浏览器插件应用程序接口（API）。1995年由网景公司发布，应用于网景导航者2.0版本，但其他浏览器很快也跟进支持，成为一个共通的插件标准，与微软的ActiveX形成竞争关系。

2014年11月，Google宣布Chrome将于2015年1月默认屏蔽NPAPI插件，9月份会完全移除支持，以鼓励开发者和用户转用HTML5、Chrome API或Google Native Client等新技术取代NPAPI。
2015年10月，Mozilla也宣布Firefox将于2016年年底移除支持NPAPI插件，但Flash Player除外。

## ActiveX(淘汰)
ActiveX在广义上是指微软公司的整个COM架构，但是现在通常用来称呼基于标准COM接口来实现对象链接与嵌入（OLE）的ActiveX控件。后者是指从VBX发展而来的，面向微软的Internet Explorer技术而设计的以OCX为扩展名的OLE控件。通过定义容器和组件之间的接口规范，如果编写了一个遵循规范的控件，那么可以很方便地在多种容器中使用而不用修改控件的代码。同样，通过实现标准接口调用，一个遵循规范的容器可以很容易地嵌入任何遵循规范的控件。

在2015年7月29日发行的Windows 10，以不支持ActiveX的Microsoft Edge浏览器，取代使用多年的Internet Explorer做为Windows默认浏览器。

## NACl(native-client)
google开发的基于沙箱技术在浏览器中安全高效地运行编译过的C/C++代码，不依赖用户操作系统。

现已宣布移除除chromeOS之外的所有平台。参见[NaCl主页](https://developer.chrome.com/native-client)
Google给出了迁移至WebAssembly的建议。[迁移建议](https://developer.chrome.com/native-client/migration)
## PPAPI
Google出于NPAPI安全性问题开发的一套跨平台的用于浏览器插件开发的API。用于替代NPAPI。是NaCl技术使用的一套API集合。
## WebAssembly
WebAssembly是一种新的编码方式，可以在现代的网络浏览器中运行 － 它是一种低级的类汇编语言，具有紧凑的二进制格式，可以接近原生的性能运行，并为诸如C / C ++等语言提供一个编译目标，以便它们可以在Web上运行。它也被设计为可以与JavaScript共存，允许两者一起工作。
[WebAssembly相关](https://developer.mozilla.org/zh-CN/docs/WebAssembly)
WebAssembly被认为是未来， Chrome, Edge, Firefox,  WebKi四大厂商共同开发。
## Emscripten
一个源代码到源代码翻译器(Source-to-source compiler),可以将C/C++编写的模块编译至WebAssembly
## 其他相关
以下内容引用自  [stackoverflow](https://stackoverflow.com/questions/41083606/porting-c-code-native-client-to-browser-web-app)

```
NaCl will not be supported by browsers other than Chrome. We (Google's NaCl/WebAssembly team) are more focused on WebAssembly for the future.
PPAPI is the set of APIs used by NaCl.
NPAPI: on the way out, and needs a plugin installation anyway.
Emscripten. It does compile C++ to JS, but I would not say that it reveals your source code to the user. It has been significantly transformed through the usual compilation and optimization process, and I would say it's closer to machine code than source code. In addition we are adding WebAssembly support to emscripten, so that with the same source you can produce WebAssembly for those browsers that support it, and asm.js for those that don't yet.
WebAssembly is the future, and will meet all of your criteria, with the caveat that the browser APIs and capabilities available to wasm are the same as those available to JavaScript. So you won't be able to get unlimited access to the user's filesystem, for example. All 4 major browser vendors are implementing webassembly, but it should appear in Chrome and Firefox first, because the release schedules of those browsers are uncoupled from their respective operating systems.
```