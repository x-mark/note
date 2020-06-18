# vscode gopls配置

## 版本信息

+ vscode version 1.46.0
+ go extension 0.14.4

## 配置

1. ctrl+shift+P 选择 Go:Install/Update Tools 选择gopls和其他需要升级的go tool  
2. 打开Settings勾选Go:Use Language Server  
3. 其他配置参考 go extension Feature列表
4. 重启vscode生效配置

## 使用注意

1. 日志信息可以在vscode控制台OUTPUT中右侧下拉选择gopls查看  
2. **一个workspace下多个包含go.mod子目录项目，选择File->Add Folder To Workspace加入工作区，否则gopls无法进行分析补全**