---
title: Laravel上传文件或图片失败
date: 2022-05-06 19:45:18
tags:
    - PHP
categories:
    - PHP
    - Bug
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/php.png
---
# Laravel上传图片或文件失败

### 起因
今天运营突然提出问题说是线上环境上传图片出现问题，我寻思，上传图片不光一个地方再用，怎么就正好是手机webview上出问题了，于是开始加入日志来排除问题。

### 错误
the "" file does not exist or readable

### 环境
服务器：nginx + php.7.4
上传：webview

### 分析
图片资源是通过ios外部传入webview内的，所以，推测是不是ios传入的数据出现问题，又因为这个功能老早就有了，ios又经过多版本更新，不确定是不是因为ios代码有改动出现的问题，
所以，我更换了老版本的ios包进行了测试。测试后发现，并非ios包的问题。所以，问题定位在图片服务器上。图片服因为根据手机类型分服的，单单只有ios出问题，但是代码又是相同的，
所以，问题应该会出现web服务器和PHP相关配置上。nginx环境如果上传的文件过大，会直接抛出错误，但是错误是从Laravel程序里抛出，那么，和nginx关联就不大了，问题只会在php相关配置上了

### 解决
修改**php.ini**文件配置，并重启php-fpm
```text
upload_max_filesize = 20M
post_max_size = 20M
```
