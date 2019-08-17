# proxychains终端socks5代理

```shell
#安装proxychains
sudo apt-get install proxychains
#修改配置文件
vi /etc/proxychains.conf
#按照说明设置 socks5 127.0.0.1 1080
```
此时终端中执行
```shell
proxychains curl ip.gs
```
会显示代理ip

# 命令别名设置

```shell
vi ~/.brashrc
#新增别名
#proxychain代理设置 USEAGE:  ss $echoip
alias ss="proxychains"
echoip="curl ip.gs"
```
