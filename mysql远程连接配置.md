
# Mysql配置

## 创建用户

```sql
#创建用户
create user 'username'@'%' IDENTIFIED BY 'password'; # %表示允许任意ip
#创建数据库
create database ZKRMQG03 DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
#权限
grant all privileges on `ZKRMQG03`.* to 'ZKRMQG03'@'%';
#刷新
flush privileges;
```