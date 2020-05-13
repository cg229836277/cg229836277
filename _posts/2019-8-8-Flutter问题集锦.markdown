---
layout: post
title:  "Flutter问题集锦"
date:   2019-8-8 11:09:22 +0800
categories: flutter
---

#### 1、执行flutter命令时阻塞，Waiting for another flutter command to release the startup lock

A：rm ./flutter/bin/cache/lockfile

#### 2、连接了设备，却报no connected devices found , or see the flutter.io

A：到任务管理器中，找到dart.exe，逐个结束任务

#### 3、VM snapshot invalid and could not be inferred from settings.

A:问题比较棘手，有好几种情况：

对比着新建一个flutter工程，看看能不能在release模式下跑起来，如果可以跑起来就将出问题的工程对比新建的工程，看看差异在哪里，参照github上面flutter的issues，可以知道大体有几个原因，参考
https://github.com/flutter/flutter/issues/19818 
可以知道：
- 1、flutter_assets下面文件拷贝不正确
- 2、Flavor没有配置，if you config Flavor in app, the flutter lib must config the same Flavor.
- 3、flutter clean,然后clean一下项目，再部署
- 4、https://github.com/flutter/flutter/issues/37841，
app目录下的build.gradle里面的debug配置签名可能存在问题，注释掉再部署也ok了。
- 5、Android工程下的目录名称不是app，是其他名称导致flutter_asset下的文件复制不完全，需要修改flutter sdk下的flutter.gradle.(https://github.com/flutter/flutter/issues/26948)

