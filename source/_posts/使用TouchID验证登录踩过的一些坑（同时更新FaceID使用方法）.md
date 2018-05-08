---
title: 使用TouchID验证登录踩过的一些坑（同时更新FaceID使用方法）
date: 2017-08-03
updated: 2017-11-26
---


### FaceID
iPhoneX 提供的刷脸功能与之前的设备的TouchID类似，都是属于生物验证的范畴。苹果爸爸也是为了照顾开发者，这两个功能对应的API基本相同，对于之前支持TouchID的APP其实可以在不做任何修改的基础上兼容FaceID，只是在一些UI样式上需要修改。

<!-- more -->

1. biometryType<br>
iOS11之后`LAContext`新增`biometryType`属性，调用时候可以根据这个属性来判断当前设备是使用FaceID还是TouchID，并据此做UI样式上的调整

		typedef NS_ENUM(NSInteger, LABiometryType)
		{
		    /// The device does not support biometry.
		    LABiometryTypeNone API_AVAILABLE(macos(10.13.2), ios(11.2)),
		    LABiometryNone API_DEPRECATED_WITH_REPLACEMENT("LABiometryTypeNone", macos(10.13, 10.13.2), ios(11.0, 11.2)) = LABiometryTypeNone,
		    
		    /// The device supports Touch ID.
		    LABiometryTypeTouchID,
		    
		    /// The device supports Face ID.
		    LABiometryTypeFaceID API_UNAVAILABLE(macos),
		} API_AVAILABLE(macos(10.13.2), ios(11.0)) API_UNAVAILABLE(watchos, tvos);


		/// Indicates the type of the biometry supported by the device.
		///
		/// @discussion  This property is set only when canEvaluatePolicy succeeds for a biometric policy.
		///              The default value is LABiometryTypeNone.
		@property (nonatomic, readonly) LABiometryType biometryType API_AVAILABLE(macos(10.13.2), ios(11.0)) API_UNAVAILABLE(watchos, tvos);

2. NSFaceIDUsageDescription<br>
使用FaceID需要在info.plist中增加`NSFaceIDUsageDescription `权限申请说明，这个跟定位、拍照等一样，如果不增加默认提示如下，虽然不会崩溃，但最好还是加上。

	![FaceID权限.jpg](http://upload-images.jianshu.io/upload_images/1429982-9cfc808834baadad.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. 其他<br>
FaceID的调用方法跟TouchID一样，都是先判断再调用，具体流程参照分割线后的TouchID部分。

4. 注意<br>
FaceID如果 ***不间断连续尝试*** 次数超过5次之后，会弹窗提示如下，同时不再执行`reply:`对应的block，这个需要注意

	![超出次数.jpg](http://upload-images.jianshu.io/upload_images/1429982-bb8849f47db4099c.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

        [myContext evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics
                  localizedReason:myLocalizedReasonString
                            reply:^(BOOL success, NSError *error) {
            //！！！超出次数，提示弹窗后，这里的block不会执行！！！
        }];


***
### TouchID
iPhone 5s之后苹果推出的TouchID功能绝对是登录验证的一大神器，自此之后各种APP在涉及到登录时如果不把这一方式加上，估计都不好意思说是做APP的。这就苦了我们众程序猿，在开发中免不了要遇上各种坑。

在次我将自己曾经趟过的一些坑罗列了下
#### 调用前的判断
在调用TouchID验证弹窗前最好先判断一下设备是否支持TouchID

		//创建安全验证对象
		LAContext * con = [[LAContext alloc]init];
		NSError * error;
		//判断是否支持密码验证
		/**
		 * LAPolicyDeviceOwnerAuthentication 可输入手机密码的验证方式
		 * LAPolicyDeviceOwnerAuthenticationWithBiometrics 只有指纹的验证方式
		 */
		BOOL can = [con canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error];
这里有两种验证方式可选：
. `LAPolicyDeviceOwnerAuthenticationWithBiometrics ` iOS8.0以上支持，只有指纹验证功能
. `LAPolicyDeviceOwnerAuthentication` iOS 9.0以上支持，包含指纹验证与输入密码的验证方式
#### 调用TouchID

    //初始化上下文对象
    LAContext *context = [[LAContext alloc] init];
    //localizedFallbackTitle＝@“”,不会出现“输入密码”按钮
    context.localizedFallbackTitle = @"";
    //错误对象
    NSError *error = nil;
    NSString *result = @"验证信息";
    
    //判断是否支持密码验证
    /**
     *LAPolicyDeviceOwnerAuthentication 手机密码的验证方式
     *LAPolicyDeviceOwnerAuthenticationWithBiometrics 指纹的验证方式
     */
    if ([context canEvaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics error:&error]) {
        [context evaluatePolicy:LAPolicyDeviceOwnerAuthenticationWithBiometrics localizedReason:result reply:^(BOOL success, NSError *error) {
            if(error.code == LAErrorTouchIDLockout) {
                
                BOOL can = [context canEvaluatePolicy:LAPolicyDeviceOwnerAuthentication error:&error];
                if (can) {
                    [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication localizedReason:result reply:^(BOOL success, NSError * _Nullable error) {
                        
                    }];
                    
                }
                else{
                    NSLog(@"调起账号密码页面失败!!!");
                }
            }
        }];
    }

注意`context.localizedFallbackTitle = @"";`如果不设置空值，则AlertView弹窗默认会有“输入密码”的选项，但是在`LAPolicyDeviceOwnerAuthenticationWithBiometrics `模式下点击“输入密码”不会有反应；`LAPolicyDeviceOwnerAuthentication`模式下点击可以唤起输入手机密码页面，页面如下，其中除了“指纹”两字是你的app名称，其他都不可定制

![iPhone.jpg](http://upload-images.jianshu.io/upload_images/1429982-a89dd0e5d803266d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
	
#### 验证错误码的判断
系统提供的错误码

        // Error codes
        #define kLAErrorAuthenticationFailed                       -1
        #define kLAErrorUserCancel                                 -2
        #define kLAErrorUserFallback                               -3
        #define kLAErrorSystemCancel                               -4
        #define kLAErrorPasscodeNotSet                             -5
        #define kLAErrorTouchIDNotAvailable                        -6
        #define kLAErrorTouchIDNotEnrolled                         -7
        #define kLAErrorTouchIDLockout                             -8
        #define kLAErrorAppCancel                                  -9
        #define kLAErrorInvalidContext                            -10
        #define kLAErrorNotInteractive                          -1004

        #define kLAErrorBiometryNotAvailable              kLAErrorTouchIDNotAvailable
        #define kLAErrorBiometryNotEnrolled               kLAErrorTouchIDNotEnrolled
        #define kLAErrorBiometryLockout                   kLAErrorTouchIDLockout
验证失败，你可以根据实际情况将错误原因反馈给用户，比如在上面的调用TouchID代码中，当判断到TouchID被锁定，使用`LAPolicyDeviceOwnerAuthentication `模式再次验证，并弹出输入密码页面解锁。
   
    if(error.code == LAErrorTouchIDLockout) {
        BOOL can = [context canEvaluatePolicy:LAPolicyDeviceOwnerAuthentication error:&error];
        if (can) {
            [context evaluatePolicy:LAPolicyDeviceOwnerAuthentication localizedReason:result reply:^(BOOL success, NSError * _Nullable error) {
            }];
        }
        else{
            NSLog(@"调起账号密码页面失败!!!");
        }
    }
#### 敲黑板 看重点
前面说的都是TouchID使用时候的常规场景，下面说一下可能会忽视的重点！！

1.  使用TouchID，必须确保app已经是活动状态！！<br>
使用TouchID，必须确保app已经是活动状态！！<br>
使用TouchID，必须确保app已经是活动状态！！<br>
也即是说当你调用TouchID时，必须确保程序已经收到了UIApplicationDidBecomeActiveNotification的消息，不然的话会调用失败，返回一个错误
   
      > Code: -1004 NSLocalizedDescription: User interaction is required

	<del> **-1004**这个错误码并不包含在官方SDK提供的文档中，但根据提示应该能够明白这是由于APP并没完全启动，未能提供用户交互导致。 </del> 
	
	**-1004 错误在iOS11 SDK中已经更新，对应的错误描述如下：**
     
        /// Authentication failed, because it would require showing UI which has been forbidden
        /// by using interactionNotAllowed property.
        LAErrorNotInteractive API_AVAILABLE(macos(10.10), ios(8.0), watchos(3.0), tvos(10.0)) = kLAErrorNotInteractive,
我曾经因为在`applicationWillEnterForeground:`以及`application: didReceiveRemoteNotification:`两个方法中进行登录判断，过早的调用了TouchID导致-1004报错，着实被坑了一把。特别是在`application: didReceiveRemoteNotification:`点击消息推送启动的时，部分设备会报-1004，而有些又不会，最后费了好大劲才找到原因。
2. 同一个文件路径下的同一份代码，打包编译成多个ipa安装包，就算各个包的`Bundle Identifier`不同，安装到同一个设备后，也只有最后生成的那个ipa包可以启用TouchID，其他包会报**Code: -1004 NSLocalizedDescription: User interaction is required**的错误。
神奇吧！！这个也是我经过了多次踩坑才发现的。开发中一份代码打多个包测试时注意下，避免再次入坑。