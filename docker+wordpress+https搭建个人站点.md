# Docker Wordpress Https搭建个人站点

## 搭建环境

CentOS7

## 部署工具
+ Docker
+ Docker Compose
+ WordPress MySql5.7 nginx （运行在docker中）

### 安装Docker、Dcoker-Compose

待续

## 获取SSL证书

这里我使用的是[Let's encrypt](https://certbot.eff.org/)提供的免费证书。  
没有使用[certbot](https://certbot.eff.org/docs/install.html)工具而是使用了轻量级的[ACME.sh](https://github.com/Neilpang/acme.sh)来代替

### 安装ACME

按照[wiki](https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E)说明一条语句安装  
```bash
curl  https://get.acme.sh | sh
```

### 生成证书文件

按照[DNS manual mode](https://github.com/Neilpang/acme.sh/wiki/dns-manual-mode)说明我选用了dns的方式来进行验证生成证书  
```bash
acme.sh --issue -d example.com --dns \
 --yes-I-know-dns-manual-mode-enough-go-ahead-please
```
这会生成一个解析记录出来
```bash
TXT_Record: _acme-challenge.example.com
Record_Value: xxxxxxxxxxx
```
然后登陆域名解析服务商账户（我用的是[阿里云解析](https://www.aliyun.com)）添加一条记录，记录类型为TXT，记录和值为上面生成的。  
再通过以下命令生成证书文件
```bash
acme.sh --renew -d example.com \
  --yes-I-know-dns-manual-mode-enough-go-ahead-please
```

## 配置

### 新建docker-compose.yml

新建一个docker-compose.yml文件
```yml
version: '3.3'
services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql #HOST:container挂载，保存数据库数据
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: xxxxx #密码
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: xxxxxx #密码

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     expose:
       - "80"
     restart: always
     environment:
       VIRTUAL_HOST: domain.com  #域名
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: xxxxxx #上面的密码
     volumes:
       - wp_plugins:/var/www/html  #HOST:container挂载，插件会保存在本机中，当容器被停止或删除后再次启动插件不会丢失

   #nginx反向代理服务设置
   nginx-proxy:
     image: jwilder/nginx-proxy
     container_name: nginx-proxy
     restart: always
     ports:
       - "80:80"
       - "443:443" # ssl 默认是443端口，因此需要将443端口映射到宿主机上
     volumes:
       - /var/run/docker.sock:/tmp/docker.sock:ro # 将宿主机的docker.sock绑定到nginx，这样，今后添加新的站点时，nginx将会自动发现站点并重启服务
       - wp_certs:/etc/nginx/certs:ro # 将nginx中的证书目录，映射到宿主机中
       - wp_nginx_conf

volumes:
    db_data:
    wp_plugins:
    wp_certs: #nginx证书命名卷

# 配置一个公共外部网络来联通所有容器
networks:
  default:
    external:
      name: nginx-proxy

```

### 添加一个Docker Network
创建一个Docker Network用来将yml中的容器连接到这个网络上。
```bash
docker network create nginx-proxy
```

### 启动容器
```bash
 docker-compose up -d
```
### 查看本地挂载位置
执行以下命令
```bash
docker volume ls
```
这个命令将在宿主机中查看Docker中所有的卷信息，你会看到一个VOLUME NAME为xxx_wp_certs（前缀是Docker自动添加的，后面的wp_certs是yml配置中指定的）的卷。
```bash
docker volume inspect –format ‘{{ .Mountpoint }}’ xxx_wp_certs
```
执行这个命令，将打印出xxx_wp_certs（这个卷名应该替换成上一步中获得的真实卷名）这个卷在宿主机中的实际路径，一般可能是：/var/lib/docker/volumes/xxx_wp_certs/_data
```bash
cp ~/.acme.sh/example.com/example.com.cer  /var/lib/docker/volumes/xxx_wp_certs/_data/example.com.crt
cp ~/.acme.sh/example.com/example.com.key  /var/lib/docker/volumes/xxx_wp_certs/_data/example.com.key
```
 执行这个命令将生成的SSL证书拷贝到nginx-proxy容器的证书目录下。  

 镜像nginx-proxy中的脚本包含了如下功能：
如果在certs文件夹下找到当前域名的.crt和.key文件，则将自动将访问转到HTTPS协议。

### 重启容器
```bash
docker-compose down
docker-compose up -d
```
 至此访问example.com就会是https的了


 ## 参考
 [https://www.fujiabin.com/2017/11/09/deploy-wordpress-with-docker-compose-with-nginx-and-add-ssl/](https://www.fujiabin.com/2017/11/09/deploy-wordpress-with-docker-compose-with-nginx-and-add-ssl/)
 [https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E](https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E)
 [https://github.com/Neilpang/acme.sh/wiki/dns-manual-mode](https://github.com/Neilpang/acme.sh/wiki/dns-manual-mode)