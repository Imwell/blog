---
title: 完全卸载nginx
date: 2021-12-08 11:04:38
tags:
    - nginx
categories:    
    - Web
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/nginx.jpg
---
## nginx卸载

### ubuntu
一般情况下，我们都是会使用nginx作为web服务器的，但是，如果希望通过openresty来替换掉原来的nginx，那么我们就需要将原来的nginx进行卸载。
步骤如下：
1. 删除nginx
```shell
sudo apt-get --purge remove nginx
```
2. 自动删除不需要的包
```shell
sudo apt-get autoremove
```
3. 罗列出所有与nginx相关的包
```shell
dpkg --get-selections|grep nginx
```
（例）结果如下：
```shell
root@ip:~# dpkg --get-selections|grep nginx
libnginx-mod-http-image-filter                  deinstall
libnginx-mod-http-xslt-filter                   deinstall
libnginx-mod-mail                               deinstall
libnginx-mod-stream                             deinstall
nginx-common                                    deinstall
```
4. 卸载相关软件
```shell
sudo apt-get --purge remove nginx-common
```
这样就可以卸载nginx和配置了

5. 全局查询nginx相关文件
```shell
sudo  find  /  -name  nginx*
```
（例）结果如下：
```shell
root@ip:~# sudo  find  /  -name  nginx*
/usr/local/openresty/nginx
/usr/local/openresty/nginx/logs/nginx.pid
/usr/local/openresty/nginx/sbin/nginx
/usr/local/openresty/nginx/conf/nginx.conf.default
/usr/local/openresty/nginx/conf/nginx.conf
```
之后就可以通过`rm`指令去删除残留的相关文件或者文件夹了
