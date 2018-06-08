# GoProxy使用测试配置

为了实现TCP/Http通过UDP协议转发，解决TCP长连接海外Server干扰问题。  
KCP是一个基于UDP的快速可靠传输协议。[GitHub](https://github.com/skywind3000/kcp)  
GoProxy是一个功能完备的网络代理工具,支持KCP协议传输。[GitHub](https://github.com/snail007/goproxy)  
利用GoProxy可以实现http/udp的代理。实现以下传输链路：   
**user**<---*tcp/http*--->**Proxy_1**<---*kcp*--->**Proxy_2**<----*tcp/http*--->**Server**  
## http代理
Proxy_1启动参数:
```bash
proxy http --always -t tcp  -p :80 -T kcp -P [Proxy_2_IPAddr:Port]
#--always: 不进行DNS解析，全部转发.(GoProxy默认会进行DNS解析，如果主机可达就不再转发)
#-t: user到Proxy1的连接类型
#-p: proxy_1的监听端口，应为代理http所以是80
#-T: 和上级代理连接方式，这里选KCP
#-P: 上级代理地址:端口
``` 
Proxy_2启动参数:
```bash
proxy http -t kcp -p :PORT 
```
配置完成后user访问http时将代理设置为Proxy_1:80,这样user通过http请求server时，先请求proxy_1,proxy_1通过kcp转发请求至proxy_2,proxy_2请求server得到结果再原路返回。

通过两台虚拟机做Proxy_1,Proxy_2，物理机通过代理访问http网站，wireshark抓包显示数据链路符合预期。

## TCP代理
Proxy_1启动参数:

```bash
proxy tcp  -t tcp  -p :PORT -T kcp -P Proxy_2_IPAddr:Port
#-t: user到Proxy1的连接类型
#-p: proxy_1的监听端口
#-T: 和上级代理连接方式，这里选kcp
#-P: 上级代理地址:端口
``` 
Proxy_2启动参数:
```bash
proxy tcp -t kcp -p :PORT -T tcp -P Server_IP:Port
```

配置完成后user,tcp连接至Proxy_1的指定端口,proxy_1通过kcp连接至proxy_2,proxy_2再通过tcp连接至server。

通过两台虚拟机做Proxy_1,Proxy_2，物理机部署echo_server和echo_client，echo_client向Proxy_1发起连接发送数据，wireshark抓包显示数据链路符合预期。