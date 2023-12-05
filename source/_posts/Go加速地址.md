---
title: Go加速地址
date: 2022-01-13 21:44:53
tags:
  - Go
categories:    
  - Go
description: Go加速地址配置
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/golang.png
---
# Go 国内加速镜像
go的很多包都需要在国外的网站拉取，比如`golang.org/x/...`。而且在国内拉取github上的模块也很慢，一般推荐一下三个镜像站：
- 七牛：https://goproxy.cn
- 阿里：https://mirrors.aliyun.com/goproxy/
- 官方：https://goproxy.io/
## 如何使用

### 配置Goproxy环境变量

#### MacOS or Linux
```shell
export GO111MODULE=on
export GOPROXY=https://goproxy.io,direct
```
#### Window(PowerShell)
```shell
$env:GO111MODULE = "on"
$env:GOPROXY = "https://goproxy.io,direct"
```
### 配置长久生效

#### Mac or Linux
```shell
# 设置你的 bash 环境变量
echo "export GOPROXY=https://goproxy.io,direct" >> ~/.profile && source ~/.profile

# 如果你的终端是 zsh，使用以下命令
echo "export GOPROXY=https://goproxy.io,direct" >> ~/.zshrc && source ~/.zshrc
```

#### Windows
1. 右键 我的电脑 -> 属性 -> 高级系统设置 -> 环境变量
2. 在 “[你的用户名]的用户变量” 中点击 ”新建“ 按钮
3. 在 “变量名” 输入框并新增 “GOPROXY”
4. 在对应的 “变量值” 输入框中新增 “https://goproxy.io,direct”
5. 最后点击 “确定” 按钮保存设置
