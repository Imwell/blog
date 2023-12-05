---
title: Linux常用指令
date: 2022-01-14 14:10:55
tags:
    - Linux
    - Shell
categories:    
    - Linux
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/linux.png
---

## 常用命令

ps命令：
```shell
ps -aux | grep java
ps -ef | grep java
```

查找对应应用PID：
```shell
# 新写法
pgrep java
# 老写法
ps -aux|grep java| grep -v grep | awk '{print $2}'
```

清空文件：
```shell
cat /dev/null > text.txt 
```

将用户从某个组删除
```shell
gpasswd --delete user group
```

杀死、暂停、继续一个进程
ps: 正在进行的进程
kill: 发信号给一个进程，一般用于杀死改进程
jobs: 列出当前shell环境中已启动的任务。挂起任务时，会出现`[1]+ stop`类似的提示
bg: 将一个后台进行的任务，搬到前台；bg %jobnumber
fg: 将一个前台任务搬到后台；fg %jobnumber

前台进程的挂起和终止：
挂起：ctrl + z
终止：ctrl + c

screen命令: 用于多重视窗管理程序，以下为一些简单命令
```shell
#创建
screen -S <名字>
# 查看所有 
screen ls
# 退出：先ctrl + a，后按d
# 恢复某个会话
screen -r <会话编号>
# 关闭
screen -X -S <会话编号> quit
```
