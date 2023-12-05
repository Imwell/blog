---
title: Erroneous data format for unserializing 'Symfony Component Routing CompiledRoute'
date: 2022-06-24 14:32:17
tags:
    - PHP
    - Laravel
    - Bug
categories:
    - PHP
cover: https://cdn.jsdelivr.net/gh/Imwell/image/blog/php.png
---
## 起因
服务器上切换php版本，然后跑Laravel项目，报如下错误：
```php
ErrorException (E_WARNING)
Erroneous data format for unserializing 'Symfony\Component\Routing\CompiledRoute'
```
![img.png](/img/laravel-bug-unserializing.png)

无论是使用php artisan的命令还是其他调用，都会报上述错误

## 解决
从上图可以看到，这个错误是cache文件产生的， 尝试删除 `bootstap/cache/route.php` 文件之后，恢复正常
