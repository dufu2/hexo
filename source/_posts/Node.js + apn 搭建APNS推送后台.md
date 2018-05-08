---
title: Node.js + apn 搭建APNS推送后台
date: 2017-04-03
updated: 2017-04-03
---

本文重点关注的是APNS推送后台的搭建，不涉及推送证书Certificate 、Profiles 文件，iOS客户端推送代码的介绍。

#### 需求背景
iOS开发，在接收到APNS推送消息后打开App，然后跳转到App内任意模块、或者打开webView页面、或者执行升级操作、或者跳转打开第三方应用；对于这样的需求，开发测试的时候需要频繁发送APNS推送，如果每次都让后台服务器的兄弟来配合测试，那肯定是心好累，最好的方案就是自己搭建服务，想怎么推就怎么推！！

<!-- more -->

#### 写在开头
 网上关于APNS推送的文章有很多，但对于推送后台介绍的文章却不多（有很多都是介绍java后台的，本人曾经参照教程用java实现过，那是在Android Studio上运行的，为了推送消息这个小小的功能，却要安装Android Studio那么大一个IDE，同时还要新建工程来运行项目，想想都头大😓）

在探寻的过程中发现了Node.js + apn是一个不错方案，只是网上关于这方面的介绍不多，有的也是[同一篇文章](http://blog.csdn.net/freedom2028/article/details/12243187)在相互转载。
文章最后对于后台的介绍只一句带过`最后，安装apn，把key.pem和cert.pem拷贝到项目目录。执行以下代码可以推送通知到指定设备`，这对于我这个js门外汉来说完全就是懵逼。。。

无奈只能实践出真理，并将摸索过程中的一些经验做了总结。

#### 干货
##### 1. 生成 pem文件
*请先确保已安装对应的Apple Push Services证书，如果没有请网上搜索Apple Push Notification service SSL关键字*

打开`钥匙串访问`- 找到对应的推送证书，分别导出`apns-dev-cert.p12`以及`apns-dev-key.p12`文件，这里我导出的是development环境下的p12，这个可以在Xcode连真机build的时候收到推送。

![导出p12.png](http://upload-images.jianshu.io/upload_images/1429982-6a7d458bb426e048.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

导出的时候会提示设置密码，这里我设置的是：123456，后面生成pem和js文件中会用到。

然后通过终端命令将两个文件转换为PEM格式：

    openssl pkcs12 -clcerts -nokeys -out apns-dev-cert.pem -in apns-dev-cert.p12

	openssl pkcs12 -nocerts -out apns-dev-key.pem -in apns-dev-key.p12

##### 2. 编辑js文件（APNS.js）

    "use strict";

	const apn = require("apn");

	// token 数组
	let tokens = ["45c0bc9952a930e96cd50df871dfc0f6a8382bb0ded0821c1111495507f8f5c3", "4a633149803b1721590802eac88003ed024be210a3053ad51fa7a657d3b2a531"];

	let service = new apn.Provider({
	  cert: "/你电脑上的绝对路径/apns-dev-cert.pem",
	  key: "/你电脑上的绝对路径/apns-dev-key.pem",
	  gateway: "gateway.sandbox.push.apple.com",
	  // gateway: "gateway.push.apple.com"; //线上地址
	  // port: 2195, //端口
	  passphrase: "123456" //pem证书密码
	});

	let note = new apn.Notification({
	    alert:  "Breaking News: I just sent my first Push Notification",
	});

	// 主题 一般取应用标识符（bundle identifier）
	note.topic = "com.xxx.xxx

	console.log(`Sending: ${note.compile()} to ${tokens}`);
	service.send(note, tokens).then( result => {
	    console.log("sent:", result.sent.length);
	    console.log("failed:", result.failed.length);
	    console.log(result.failed);
	});

	service.shutdown();

##### 3. 配置
安装Node.js地址 https://nodejs.org

安装apn，终端执行：
>$  sudo npm install apn

配置node_modules环境变量 ，终端执行：
>// 进入当前用户目录，创建.bash_profile文件（如果没有会新建）<br>
>$  touch ~/.bash_profile

打开.bash_profile文件，添加以下内容并保存

>export NODE_PATH=/usr/local/lib/node_modules/

完成后在终端运行：
> $ node APNS.js

不出意外，等待若干秒后可以收到推送消息，如果有问题可以根据终端提示修改。
更多关于apn的介绍看这里：https://github.com/node-apn/node-apn