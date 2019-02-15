# CentOS7添加开机启动脚本

## 通过chkconfig方式

新建启动脚本

```sh
cd /etc/rc.d/init.d/
nano autostart.sh
```

编辑需要执行的命令

```sh
#!/bin/sh
#
# chkconfig: - 85 15
# description: xmark.xyz docker-compose 开机启动脚本
#

service docker restart
docker-compose -f /root/xmark/wordpress/docker-compose.yml up -d

```

添加可执行权限

```sh
chmod +x autostart.sh
```

添加chkconfig

```sh
chkconfig --add autosatrt.sh
chkconfig autostart.sh on
```

### 注意

autostart.sh中如果没有chkconfig 和 description两行，在执行`chkconfig --add autosatrt.sh`会报错`service autostart.sh does not support chkconfig`
