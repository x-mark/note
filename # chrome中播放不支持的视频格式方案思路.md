# chrome中播放非html5支持的视频格式方案思路
chrome html5 Video Audio支持格式有限,[支持列表](https://developer.mozilla.org/en-US/docs/Web/HTML/Supported_media_formats)。

新的WebAssembly技术可以用来将特定的解码器移植进浏览器中，解决js解码性能不够的问题。但是同样有html5 Video Audio标签限制问题。

考虑通过本地host提供性能有要求的编解码等服务，通过chrome Native Massage方式和chrome Extension通信，extension再将处理返回的html5支持内容嵌入。
参考方案：
1. 类似视频直播；先在本地转码成html5支持的格式，然后分段加载播放。  
源数据-->host-->mp4List-->chromeExtension-->html_Video;  
* 转码MP4,性能


2. 直接绘制；在本地解码视频得到每一帧的图像信息，然后传入html里显示图片。  
源数据-->host-->video&audio;  
video-->images-->chromeExtension-->html canvas/img;  
audio-->?
* 图片base64编码传入html，使用img标签插入来显示;
```html
<img src=“data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAAkCAYAAABIdFAMAAAAGXRFWHRTb2Z0d2FyZQBBZG9iZSBJbWFnZVJlYWR5ccllPAAAAHhJREFUeNo8zjsOxCAMBFB/KEAUFFR0Cbng3nQPw68ArZdAlOZppPFIBhH5EAB8b+Tlt9MYQ6i1BuqFaq1CKSVcxZ2Acs6406KUgpt5/LCKuVgz5BDCSb13ZO99ZOdcZGvt4mJjzMVKqcha68iIePB86GAiOv8CDADlIUQBs7MD3wAAAABJRU5ErkJggg%3D%3D”/>
```
* 图片RGB信息传入canvas绘制;  
* audio怎么处理？本地host播放?处理成mp3等片段传入之后再用audio标签嵌入html播放?

* 使用[Stream from canvas to video element](https://webrtc.github.io/samples/src/content/capture/canvas-video/)技术取代替换img的方式，本地host解码RGP数据，通过webSocket发送给js，js实现在canvas上绘制，将stream通过video标签绘制。

## 方案测试
1. host服务将视频转成png图片并进行base64编码，通过chromeExtension的Native Message机制按照帧率将图片发送给Extension，Extension再替换页面中的img标签src属性。NativeMessage机制限制(Extension发送给host单次最大2G,host发送给Extension单次最大1M,这里可以考虑不用NativeMessage而用webSocket方式通信。)  
  测试机配置i76700HQ 4核心8线程 2.6GHZ, 图片分辨率1920x1080  
  100张/s显示Chrome CPU占用率30%+  
  25张/s img设置显示960x540 chrome界面显示时CPU占用10%左右 chrome界面隐藏时CPU占用5%左右