title: HBuilder 云打包苹果 安卓应用
date: 2014-12-15 20:40:51
tags: [cloud]
---
HBuilder 是一个国人开发的IDE,这是[HBuilder]()的介绍 "HBuilder是当前最快的HTML开发工具",快不快我不知道,反正用起来都没有WebStorm 顺手,如果真要快直接VIM吧.
不过正是这个HBuild中集成了一个App云打包的特性吸引了我,也就是说,通过这个IDE编写的代码可以通过云端直接打包出IOS和Android两种应用程序,着实让人想入非非啊~~.
编写的代码需要符合[HTML5+](http://www.html5plus.org/#specification#/specification/Accelerometer.html)的规范,云编译后即可调用各种系统接口了...(点点爽不!?).
另外在此基础上还有一个MUI,自称 "最接近原生APP体验的高性能前端框架".
反正都这么说了,我就来试一试呗.

<!-- more -->

## HBuilder
本着用之前先看文档的原则,就看看[文档](http://ask.dcloud.net.cn/docs)吧

>HTML5过去被称为有“性工能”障碍，即性能不如原生，工具不如原生、功能不如原生。

![对比](http://ask.dcloud.net.cn/uploads/article/20141107/4af865dea6c5babe56af7c095f0791be.png)
这吐槽够可以的,看来这一套件就是专为会JS的人写写java或者object-C移动应用准备的,桌面有node-webkit,后端有node.js,这是要发啊~~

![屎黄色界面](http://daysv.qiniudn.com/o_19csr3nmd1iv84qu18bo1gpu1873e.jpg)
屎黄色界面也有一定的亲切感,据说能保护视力,HBuilder 主体基于`eclipse`,也就是说很多eclipse插件都能运用啦
在新建项目的过程中可以先把自带的几个项目先看看,该有的组件都有了
不过在写应用的时候**不建议使用各种库,只建议使用vanilla javascript这个库**这样才能有效的保证程序的运行速度.
最后看看最主要的打包功能
![App云打包](http://daysv.qiniudn.com/o_19csq9q2d13bdnkj1gh25q01hb19.jpg)

看到cnode社区有人通过ionic写了个app,我也尝试写了个用来学习 **[a piece of shit](https://github.com/daysv/MUI-CNode)**,写的过程中发现有些文档写的还是不太清楚

## HTML5plus
[http://www.html5plus.org/#specification#/specification/Accelerometer.html](http://www.html5plus.org/#specification#/specification/Accelerometer.html)

## MUI
MUI是个轻量级框架,其中不少直接基于HTML5+
[http://dcloudio.github.io/mui/](http://dcloudio.github.io/mui/)