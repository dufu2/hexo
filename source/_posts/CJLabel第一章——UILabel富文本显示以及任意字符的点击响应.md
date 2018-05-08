---
title: CJLabel第一章——富文本显示及任意链点点击
date: 2016-04-20
updated: 2016-04-20
---


[CJLabel已更新](https://github.com/lele8446/CJLabelTest)<br>相关文章介绍：<br>
[CJLabel第二章——图文混排及精确点击区域](http://lele8446.top/2017/05/03/CJLabel第二章——图文混排及精确点击区域/)<br>
[CJLabel第三章——支持任意区域点击响应和可选择复制原理](http://lele8446.top/2017/11/14/CJLabel第三章——支持任意区域点击响应和可选择复制原理/)
***
在NSAttributedString出现之前，UILabel只能做简单的文本显示，如果要显示富文本我们只能通过Core Text来实现，同时UILabel的userInteractionEnabled属性默认是NO的，即不能响应用户的交互操作。在本文我通过UILabel的attributedText来显示富文本（关于对NSAttributedString的封装以及size信息的计算，可参照上一篇文章[动态计算NSAttributedString的size大小](http://lele8446.top/2016/04/19/动态计算NSAttributedString的size大小/)），同时判断每次点击label时对应字符的NSRange位置信息，实现了任意字符的点击响应。

<!-- more -->

***
* 先上效果图

	![image](http://upload-images.jianshu.io/upload_images/1429982-a44d3e804e769dc0.gif?imageMogr2/auto-orient/strip)
* 创建UILabel的子类CJLabel

	      @interface CJLabel : UILabel
	      /**
	        *  是否加大点击响应范围，类似于UIWebView的链点点击效果，默认NO
	        */
	      @property (nonatomic, assign) BOOL extendsLinkTouchArea;
	
	      /**
	       *  文本中内容相同的链点是否都能够响应点击，
	       *  必须在设置self.attributedText前赋值，
	       *  如果值为NO，则取文本中首次出现的链点，
	       *  默认YES
	       */
	      @property (nonatomic, assign) IBInspectable BOOL sameLinkEnable;
	
	      /**
	        *  增加点击链点
	        *
	        *  @param linkString 响应点击的字符串
	        *  @param linkDic    响应点击的字符串的Attribute值
	        *  @param parameter  响应点击的字符串的相关参数：id，色值，字体大小等
	        *  @param linkBlock  点击回调
	        */
	        - (void)addLinkString:(NSString *)linkString linkAddAttribute:(NSDictionary *)linkDic linkParameter:(id)parameter block:(CJLinkLabelModelBlock)linkBlock;
	
	      /**
	        *  取消点击链点
	        *
	        *  @param linkString 取消点击的字符串
	        */
	      - (void)removeLinkString:(NSAttributedString *)linkString;
	      @end
	      
方法 `- addLinkString: linkAddAttribute: linkParameter: block:`添加要响应点击事件的字符串，block为点击该字符串对应的回调事件；`- removeLinkString:`移除点击事件。
	
	      - (void)addLinkString:(NSAttributedString *)linkString block:(CJLinkLabelModelBlock)linkBlock {
	          CJLinkLabelModel *linkModel = [[CJLinkLabelModel alloc]initLinkLabelModelWithString:linkString range:[self getRangeWithLinkString:linkString] block:linkBlock];
	          if (nil != linkModel) {
	              [self.linkArray addObject:linkModel];
	          }
	      }
`CJLinkLabelModel`是自定义的点击链点model类，包含三个属性linkBlock（该链点的点击回调事件）、linkString（链点对应的字符串）、range（链点对应的NSRange信息）、parameter（可传递点击链点的相关参数：id，色值，字体大小等）。每添加一段能够响应点击事件的字符串，都新创建一个`CJLinkLabelModel`，并add到`self.linkArray`中。

 设置CJLabel的属性`self.userInteractionEnabled = YES`，同时重写`- touchesBegan: withEvent:`方法

      - (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event {
         UITouch * touch = touches.anyObject;
         //获取触摸点击当前view的坐标位置
         CGPoint location = [touch locationInView:self];
         //判断是否响应了点击事件
         if(![self needResponseTouchLabel:location]) {
            [self.nextResponder touchesBegan:touches withEvent:event];
         }
      }
`- needResponseTouchLabel`方法将点击坐标转化为对应的第index个字符，然后遍历`self.linkArray`数组，判断该字符是否在需响应点击事件的字符串数组内，是的话则响应点击回调，否则向`self.nextResponder`继续传递点击交互。

 另外`self.extendsLinkTouchArea`可以设置是否加大点击响应范围，类似于UIWebView的链点点击效果，默认NO。

 `- removeLinkString:`方法

      - (void)removeLinkString:(NSAttributedString *)linkString {
          __block NSUInteger index = 0;
          __block BOOL needRemove = NO;
         [self.linkArray enumerateObjectsUsingBlock:^(id num, NSUInteger idx, BOOL *stop){
              CJLinkLabelModel *linkModel = (CJLinkLabelModel *)num;
              if ([linkModel.linkString isEqualToAttributedString:linkString]) {
                  index = idx;
                  needRemove = YES;
                  *stop = YES;
               }
          }];
          [self.linkArray removeObjectAtIndex:index];
      }
#####  CJLabel的使用
* 使用cocoapods安装

		platform :ios, '7.0'
		pod 'CJLabel', '~> 1.0.3'
* 直接引用：
从GitHub下载[CJLabel](https://github.com/lele8446/CJLabelTest)文件夹，直接引用到工程中。

***
最后附上[demo地址](https://codeload.github.com/lele8446/CJLabelTest/zip/1.0.3)