# 使用openssl实现FTP客户端对FTPS的支持

## FTP主动模式和被动模式

* PASV 被动模式
  被动模式是ftp server开启监听，等待client发起tcp连接到server的指定端口  
  + client 发送PASV命令
  + server 响应消息携带ip和端口信息，形如：(192,168,60,1,24,56)前个数是ip，后两个数m,n通过m*256+n计算出端口，这里是24*256+56=6200
  + client tcp连接到server的指定端口
  + dataChannel建立
* PORT 主动模式
  主动模式是ftp client开启监听，等待server发起tcp连接到client的指定端口
  + client 发送PORT (192,168,60,1,24,56) 消息通知server client监听的ip和端口
  + server 通过20端口tcp connect到client的制定端口
  + dataChannel建立完成

## TLS注意SSL_connect调用

dataChannel的tcp连接建立成功之后，需要controlChannel发送需要使用dataChannel命令，比如RETR，然后dataChannel进行SSL_connect，开始传输数据。
数据传输完成后，SSL_shutdown SSL_free 再关闭tcp连接

## PROT命令

client需要通知server dataChannel的保护级别，在FTPS中是PROT P  
详细信息见[RFC2228](https://tools.ietf.org/html/rfc2228)