---
title: iOS如何部署企业包，以供他人下载
date: 2016-03-18
updated: 2016-03-18
---

要安装一个App到非越狱的手机上，一般有以下几种方式：

1. 通过App Store下载安装；
2. 如果能获取用户设备，直接通过Xcode将包灌入设备；
3. 个人开发者账号，获取用户设备UDID，生成对应的Provisioning Profiles后打包供人安装；
4. 企业开发者账号，则可以将包部署到支持https下载的服务器上随意下载（如果被苹果监测到你通过企业账户来大规模散发，有可能被封号-_-#）

<!-- more -->

今天要说的就是如何部署企业包，让用户可以通过扫码或点击链接的方式下载安装。

***

1. 首先假设你已经打包导出了企业包的.ipa文件（至于企业包的证书配置、打包生成可以自行百度），另外我们还需要.plist文件，这是找到第三方服务器上的.ipa包进行下载的关键，Xcode6之前打包之后会自动生成.plist文件，现在需要自行配制，这里提供一份模版[test.plist](https://o44fado6w.qnssl.com/test.plist.zip)
![test.plist.png](https://upload-images.jianshu.io/upload_images/1429982-9ed128fd3938ad15.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)
需要特别注意的是红框里的几处：
  * ipa的下载地址是指将ipa包上传至第三方服务器后生成的下载地址，后面会详细说明
  * bundle-identifier需要跟app内的一致，千万别搞错了
  * 主标题是下载弹窗提示显示的app名称，其它要点可以按需要来填
  
2. 上传.ipa包和plist文件
上传需要选择能够支持https协议的服务器才行，至于为什么（这是iOS7之后苹果要求的，就是这么任性。***更正下：只要求.plist文件在https服务器上，ipa包不要求***）。我选择的是[七牛云存储](https://portal.qiniu.com/)，算是国内做的比较好的一个第三方服务器，注册免费，免费送空间，不过开通https服务需要¥10大洋，还算ok。
七牛空间设置：
   * 注册七牛，新建一个空间，点击进入该空间
   * 选择空间设置－域名设置，找到HTTPS付费开通，我这里是已经开通了，所以是灰色
![https域名.png](https://upload-images.jianshu.io/upload_images/1429982-ecb65d3267d4bf18.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

   * 开通后拉到下面，选择“默认域名－配置”，在弹框中选择第二项（第一项是http的域名链接，第二项是https的）
![域名配置.png](https://upload-images.jianshu.io/upload_images/1429982-2536f45e48e4354c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

 空间配置完成就可以上传文件了
    * 还是在新建的空间内，点击"内容管理"，上传文件
    * 上传后选中文件，点击向下小三角▼弹出“外链地址”，或者直接copy右边的外链地址
![获取下载地址.png](https://upload-images.jianshu.io/upload_images/1429982-bb4e7baf736c0711.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

    * 将ipa包的外链地址放到第1步提到的plist文件中ipa的下载地址中，保存plist文件
    * 再把plist文件也上传至七牛服务器，copy外链地址
3. 把第二步中得到的plist外链地址拼成（url后面的一串即为plist文件对应的外链地址）
`itms-services:///?action=download-manifest&url=https://xxx/test.plist`
最后把上面一串下载链接输入到Safari中，就会自动弹窗提示下载了

	![下载.png](https://upload-images.jianshu.io/upload_images/1429982-9fb71f58acd45d28.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

4. 每次都这样输入链接安装很是麻烦，你可以把下载链接做成html页面内的点击下载，或生成二维码来扫码下载都可以。这里推荐一个二维码生成链点[http://cli.im](http://cli.im)，操作很简单，将下载链接填入生成二维码，就可以让用户扫码下载安装了（顺便说一下，如果是用微信扫码是不会自动弹出安装页面的，你可以用其它支持扫码的App试试）。