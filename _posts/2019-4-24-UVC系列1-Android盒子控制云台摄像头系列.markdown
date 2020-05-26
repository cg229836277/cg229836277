---
layout: post
title:  "UVC系列1-Android盒子控制云台摄像头系列"
date:   2019-4-24 18:19:25 +0800
categories: Android
---


> 微信公众号：Android部落格
>
> 文章最初发布在CSDN


## 1、知识点
Android作为host端控制云台摄像头整个实现过程中涉及了Android kernel底层UVC部分，Android kernel代码的编译，USB协议，Android JNI方面的知识。

## 2、背景
刚开始项目提出这个需求的时候，想到的是通过Android原生的USB API 去控制外接的USB PTZ摄像头，因为大多数的云台摄像头支持pelco-d或是pelco-p协议，而Android提供的接口可以传递byte[]类型的参数过去，设想通过这种方式实现控制。现在回想起来这个方法真是too young to naive。因为Android kernel层是通过UVC（usb video class）协议取控制摄像头PTZ，需要走这一套逻辑才能实现。

## 3、思路
由此思路开始转到通过Android UVC来控制摄像头转动，但是google了大大小小的网站没有人做过这个东西，侧面也在一定程度上说明了需求足够操蛋。UVC在Android的kernel层，但是怎么去验证这个东西呢，就想到了有Ubuntu，而Ubuntu上面有一个工具uvcdyctrl 可以输入对应的参数控制摄像头，于是通过这个工具验证了在linux下控制云台转动的可能性。

紧接着要解决的是如何把这个可能性移植到Android上，从前期的实践看，是需要查看kernel层的实现，kernel层的参数是否实现了相对和绝对的控制；如果kernel层实现了，怎么把这个实现传递到app层面，让app可以输入参数控制转动。

现在思路很清晰了，就是打通从kernel到app的通道，实现控制，将app层的控制指令传递到kernel，由kernel将控制字节传递到硬件。

## 4、探索系列
接下来的文章是：

- 1、[探索Android UVC协议](https://chengang.plus/blog/UVC%E7%B3%BB%E5%88%972-%E6%8E%A2%E7%B4%A2Android-UVC%E5%8D%8F%E8%AE%AE.html)；

- 2、[研究UVC控制协议](https://chengang.plus/blog/UVC%E7%B3%BB%E5%88%973-%E7%A0%94%E7%A9%B6UVC%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE.html)；

- 3、[定制Android kernel UVC部分支持相对和绝对参数](https://chengang.plus/blog/UVC%E7%B3%BB%E5%88%974-%E5%AE%9A%E5%88%B6Android-kernel-UVC%E9%83%A8%E5%88%86%E6%94%AF%E6%8C%81%E7%9B%B8%E5%AF%B9%E5%92%8C%E7%BB%9D%E5%AF%B9%E5%8F%82%E6%95%B0.html)；

- 4、[编写Android jni代码实现控制PTZ](https://chengang.plus/blog/UVC%E7%B3%BB%E5%88%975-%E7%BC%96%E5%86%99Android-jni%E4%BB%A3%E7%A0%81%E5%AE%9E%E7%8E%B0%E6%8E%A7%E5%88%B6PTZ.html)。