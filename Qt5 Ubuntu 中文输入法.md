# Ubuntu Qt5中文输入法

## 安装iBus-Rime输入法
 [RIME | 中州韻輸入法引擎](https://rime.im/)
 ```shell
 sudo apt install ibus-rime
 #简体拼音输入法
 sudo apt install librime-data-pinyin-simp

 #配置iBus输入法
 ibus-setup
 #弹出iBus首选项
 #  常规中可以设置切换快捷键
 #  输入法中添加可添加Rime等输入法
 ```

 ## Qt中文输入配置

 ```shell
 sudo vim /etc/profile 

 # 在文件最后新增
export GTK_IM_MODULE=ibus 
export QT_IM_MODULE=ibus 
export XMODIFIERS=@im=ibus 
ibus-daemon -d -x
#保存后重启
 ```