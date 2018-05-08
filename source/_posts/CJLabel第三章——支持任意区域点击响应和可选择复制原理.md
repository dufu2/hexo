---
title: CJLabel第三章——支持任意区域点击响应和可选择复制原理
date: 2017-11-14
updated: 2017-11-14
---

### 导读
**[CJLabel](https://github.com/lele8446/CJLabel)** 是一个继承自UILabel的自定义控件，它在支持UILabel所有属性的基础上，还提供富文本展示、图文混排、自定义点击链点设置、长按（双击）唤起UIMenuController选择复制文本等功能。

相关文章介绍：<br>
[CJLabel第一章——富文本显示及任意链点点击](http://lele8446.top/2016/04/20/CJLabel第一章——UILabel富文本显示以及任意字符的点击响应/)<br>
[CJLabel第二章——图文混排及精确点击区域](http://lele8446.top/2017/05/03/CJLabel第二章——图文混排及精确点击区域/)
 
<!-- more -->

**`CJLabel`** 经过若干版本迭代，各个功能已经日趋完善，并且不断精细，特别是在`V4.0.0`版本迎来了重头戏：**新增enableCopy属性，支持选择、全选、复制功能，类似UITextView的选择复制效果。**
老规矩，上效果图：

![CJLabel](https://user-gold-cdn.xitu.io/2018/3/24/162584e258dacc65?w=295&h=430&f=gif&s=232382)

![CJLabel1](https://user-gold-cdn.xitu.io/2018/3/27/1626507cc251a22b?w=290&h=438&f=gif&s=1589611)

### CJLabel链点点击实现细节
先来回顾一下CJLabel在显示文本以及响应链点点击的过程中，底层是怎样实现的。
![CJLabel.png](https://user-gold-cdn.xitu.io/2018/3/24/162584e258dbebe5?w=1240&h=793&f=png&s=257954)


#### 一. 设置Attributes属性
首先设置需要显示的NSAttributedString文本的属性，除了可设置系统提供的`NSFontAttributeName` `NSForegroundColorAttributeName` `NSParagraphStyleAttributeName`等默认属性外，还支持**CJLabel**的若干自定义扩展属性：
`kCJBackgroundFillColorAttributeName`背景填充颜色
`kCJBackgroundStrokeColorAttributeName`背景边框线颜色
`kCJBackgroundLineWidthAttributeName`背景边框线宽度
`kCJStrikethroughColorAttributeName`删除线颜色
……
CJLabel提供配置管理类`CJLabelConfigure`，专门用来方便设置指定字符的副文本属性，同时还提供了对应的API，调用可生成封装好的NSAttributedString副文本（此处只选取若干方法说明，更多可查看[源码](https://github.com/lele8446/CJLabel)

	/**
	 根据图片名初始化NSAttributedString

	 @param image         图片名称，或者UIImage
	 @param size          图片大小（这里是指显示图片等区域大小）
	 @param lineAlignment 图片所在行，图片与文字在垂直方向的对齐方式（只针对当前行）
	 @param configure     链点配置
	 @return              NSAttributedString
	 */
	+ (NSMutableAttributedString *)initWithImage:(id)image
	                                   imageSize:(CGSize)size
	                          imagelineAlignment:(CJLabelVerticalAlignment)lineAlignment
	                                   configure:(CJLabelConfigure *)configure;

	/**
	 根据NSString初始化NSAttributedString
	 */
	+ (NSMutableAttributedString *)initWithString:(NSString *)string configure:(CJLabelConfigure *)configure;

#### 二. 计算label的CGRect大小 

` label.attributedText = @"text"`

UILabel绘制显示文本，首先会触发以下方法
`-textRectForBounds:limitedToNumberOfLines:`
`-sizeThatFits:`
 
我们可以在这两个方法里面根据需要显示的文本内容以及扩展属性`self.textInsets（绘制文本的内边距，默认UIEdgeInsetsZero）`，计算当前label的CGRect大小，计算使用的核心函数是:

	CGSize CTFramesetterSuggestFrameSizeWithConstraints(
		CTFramesetterRef framesetter,
		CFRange stringRange,
		CFDictionaryRef __nullable frameAttributes,
		CGSize constraints,
		CFRange * __nullable fitRange )
		
#### 三. CTFrameRef
`-drawTextInRect:`是真正进行内容绘制的方法，我们将在这里得到所有字符对应的CTFrameRef、CTLineRef以及CTRunRef
![CTFrameRef.png](https://user-gold-cdn.xitu.io/2018/3/24/162584e2433d6999?w=1240&h=400&f=png&s=54772)

如图，UILabel显示的时候，所有内容都由CTFrameRef管理，然后每一行内容是一个CTLineRef，而每一行CTLineRef中包含了若干个CTRunRef。每一个CTRunRef对应的可能只是一个字符，也可能是整一行文字（连续的具有相同Attributes属性的字符会包含在同一个CTRunRef中），比如以下例子，CTLineRef中包含三个CTRunRef，分别对应为：`这是` `一段` `测试数据` 三部分。![  .jpg](https://user-gold-cdn.xitu.io/2018/3/24/162584e28668df7b?w=186&h=60&f=jpeg&s=2327)
获取到各个字符对应的CTRunRef后，我们可以根据CTRunRef进一步判断得到这一部分字符在UIlabel中对应的CGRect大小，以及在第一步中对当前字符设置的其他自定义属性。这一步也是整体流程中最复杂的部分，涉及到各种坐标数值的转换判断，最后将CTRunRef对应的信息转换为model记录保存。

#### 四. CTRunDraw
上一步已经获取得到了每一个CTRunRef的详细信息，此时我们可以执行最后的图文绘制操作了。首先是绘制自定义背景颜色（即`kCJBackgroundFillColorAttributeName`相关属性）；然后是绘制图文，如果是文字执行`CTRunDraw(CTRunRef run, CGContextRef context, CFRange range )`函数，如果是图片执行`CGContextDrawImage(CGContextRef c, CGRect rect, CGImageRef image)`函数；最后填充边框线以及删除线（`kCJBackgroundStrokeColorAttributeName` `kCJStrikethroughColorAttributeName`）。

至此，CJLabel已经完成了显示部分的所有操作。

#### 五. 点击响应
CJLabel默认`userInteractionEnabled = YES`，如此我们可以在touch相关的方法中捕获到CJLabel的点击事件，通过判断点击触摸点`CGPoint`是否在保存记录的CGRect数组内，如果是则执行对应点击字符的点击回调事件，同时触发点击字符的高亮重绘（如果存在高亮状态的话，而且CGContextRef的重绘是全局重绘，无法做到局部刷新）。
![touches.png](https://user-gold-cdn.xitu.io/2018/3/24/162584e258e38389?w=800&h=99&f=png&s=136097)
另外给CJLabel添加长按手势`UILongPressGestureRecognizer`监听，在长按事件中同样执行与touchesBegan:类似的逻辑判断，从而使CJLabel具备长按点击功能。


***
以上便是CJLabel功能的实现原理讲解，下面进入本文的重点——**如何使UILabel具备选择复制的能力**

当然这里说的选择复制不可能是指点击唤起`UIMenuController`菜单，然后出现复制剪切选项，点击只能复制所有文本那样的功能。那样的例子网上已经有很多，没有必要在这里再大费周章地来罗列说明。
CJLabel需要具备的是类似于UITextView或UIWebView那样，双击或长按，可出现`选择、全选、拷贝`选项，同时选中字符左右出现标示大头针，拖动则有放大镜提示当前选中字符，并且尽量做到与系统行为一致。
![CJLabel.gif](https://user-gold-cdn.xitu.io/2018/3/24/162584e2589879ed?w=292&h=152&f=gif&s=773362)
刚开始面对如此需求的时候，感觉有点无从下手。查遍资料也没找到UITextView或UIWebView中有关选择复制功能的资料说明，更不要说相关的API调用了，很明显苹果并没有将此类功能封装成模块化，就算有那相关的方法也是私有API。
因此在初始的时候，由于开发时间紧，本人选择了使用UITextView代替CJLabel作为显示控件（产品业务要求支持图文混排，支持富文本显示，文本能够自动识别@用户，能够自动识别网址链接并替换为规定的图标展示，文本内容还要支持选择复制……类似于微博的列表页面，但却比它更复杂😓😓😓）

但是UITextView存在若干问题：首先是点击链点的设置不够灵活，而且链点的高亮颜色只能全局设置，不能做到不同链点分别自定义；再就是UITextView在不同的iOS的系统版本下UI层级不一致，而且在触发点击、滑动操作时样式会发生偏移重置。
经历了初代版本的各种bug填坑，下一次版本迭代时我果断放弃了UITextView，决定用CJLabel来实现以上的需求。
很明显，CJLabel本身对于图文混排、富文本展示部分已经很好的支持了，那么剩下的就只是怎么支持选择复制。

### 需求细化
选择复制的需求主要包含以下几点
* 选中字符后出现`选择 全选 复制`菜单，这个使用系统的`UIMenuController`功能即可实现，不存在难点问题。
* 对于选中的文字，起始要有大头针标识，中间填充浅蓝色背景，而且这一部分区域会是一块不规则多边形。系统没有提供现成可复用的对应UI控件，但只要我们能够判断到选中区域，想要什么样式都可以自己绘制。所以这一块也不存在问题。
* 拖动选择的过程中，出现放大镜提示选中字符的更改。在能够获取到指定触摸点区域的前提下，只需要将对应区域的`CGContextRef`上下文做`CGContextScaleCTM`缩放，然后再将放大后的`CALayer`层显示出来，所以这个也是可以实现的。
* 最后便是重点了，如何判断每一个字符对应的`CGRect `坐标位置，并在手指移动时准确判断选择区域的变化。

### 实现
回顾前面CJLabel图文显示的过程中，其实已经做过了对特定字符的`CGRect `坐标位置的计算。只不过上面只是对指定链点做了判断记录，那如果我们能够对每一个字符都做转换并保存记录到`_allRunItemArray`数组内，那么后面的所有操作就都可以基于`_allRunItemArray`来实现了。
对应到`CTFrameRef `层则是需要保证`CTLineRef`中的每一个`CTRunRef `都只包含一个字符，还是这个例子：
![  .jpg](https://user-gold-cdn.xitu.io/2018/3/24/162584e28668df7b?w=186&h=60&f=jpeg&s=2327)
此时它在`CTFrameRef `层级应该是`这` `是` `一` `段` `测` `试` `数` `据` 这样实现，而我们知道连续的具有相同Attributes属性的字符会包含在同一个CTRunRef中，那就想办法让每一个字符都具有不同属性。
我一开始的做法是添加一个自定义属性`kCJIndexAttributesName`，然后给每个字符存储不同的index值

	//给每一个字符设置index值，enableCopy=YES时用到
	__block NSInteger index = 0;

	[attText.string enumerateSubstringsInRange:NSMakeRange(0, [attText length]) options:NSStringEnumerationByComposedCharacterSequences usingBlock:
	 ^(NSString *substring, NSRange substringRange, NSRange enclosingRange, BOOL *stop) {
	     
	     [attText addAttribute:kCJIndexAttributesName
	                     value:@(index)
	                     range:substringRange];
	     index++;
	 }];
在`CTFrameRef `的判断中，确实达到了将每个字符拆分为一个对应的`CTRunRef `要求，但其中却存在一个难以发觉的bug！！！按照常规思路，对于添加的自定义属性`kCJIndexAttributesName`，在遍历完成后将其移除，那么之后也就不会再对这个属性进行判断。但实际使用中却是移除并不生效，特别是当页面内的UITableView存在多个CJLabel，每个CJLabel都是长文本，滑动的时候会变的越来越卡。因为滑动UITableViewCell重置时会对每个CJLabel的每个字符的`kCJIndexAttributesName`做不停的遍历计算。

然而如果是用系统提供的Attributes相关的属性设置不同值则不会存在以上问题（好不容易才发现了这个bug，我猜测苹果对于NSAttributedString的Attributes属性的管理应该是有一个类似单例的地方统一存储管理的，而且它对于一些自定义的添加对象不会友好支持）。踩完坑后只好乖乖地从系统方法中寻找解决思路，幸好发现了`NSLinkAttributeName`属性，这是UITextView中用来设置http链接的扩展属性，存储的对象是`NSURL`或`NSString`类型，而UILabel默认是不支持http链点的，使用`NSLinkAttributeName`属性可以最大限度的降低UILabel对默认NSAttributedString展示的影响。同时为了更好的判断计算，我将存储的对象改为NSURL的子类`CJCTRunUrl`，更改后的代码

	//给每一个字符设置index值，enableCopy=YES时用到
	__block NSInteger index = 0;

	[attText.string enumerateSubstringsInRange:NSMakeRange(0, [attText length]) options:NSStringEnumerationByComposedCharacterSequences usingBlock:
	 ^(NSString *substring, NSRange substringRange, NSRange enclosingRange, BOOL *stop) {
	     
	     CJCTRunUrl *runUrl = nil;
	     if (!runUrl) {
	         NSString *urlStr = [NSString stringWithFormat:@"https://www.CJLabel%@",@(index)];
	         runUrl = [CJCTRunUrl URLWithString:urlStr];
	     }
	     runUrl.index = index;
	     runUrl.rangeValue = [NSValue valueWithRange:substringRange];
	     [attText addAttribute:NSLinkAttributeName
	                     value:runUrl
	                     range:substringRange];
	     index++;
	 }];
初始化的时候给CJLabel新增双击手势`UITapGestureRecognizer`。
结合前面已经判断记录的所有字符的CGRect信息，当发生长按或者双击事件的时候，判断到当前触摸的字符不是可点击链点时，那么出现选择复制视图。

#### 选择 复制视图
选择复制视图包含三部分：
* 选择、全选、复制 菜单
* 放大镜
* 大头针包含区域

第一部分直接使用`UIMenuController `，重点重载以下方法就可以了


	- (BOOL)canBecomeFirstResponder {
	    return YES;
	}

	- (BOOL)canPerformAction:(SEL)action withSender:(id)sender {
	    if ( (action == @selector(select:) && self.attributedText) // 需要有文字才能支持选择复制
	        || (action == @selector(selectAll:) && self.attributedText)
	        || (action == @selector(copy:) && self.attributedText))
	    {
	        return YES;
	    }
	    return NO;
	}

第二部分放大镜，自定义UIView子类`CJMagnifierView`，并在`CJMagnifierView`上添加一个处理放大效果的layer层`CJContentLayer`

	@interface CJContentLayer : CALayer
	@property (nonatomic, assign) CGPoint pointToMagnify;//放大点
	@end
	@implementation CJContentLayer

	- (void)drawInContext:(CGContextRef)ctx {
		CGContextTranslateCTM(ctx, self.frame.size.width/2, self.frame.size.height/2);
		CGContextScaleCTM(ctx, 1.40, 1.40);
		CGContextTranslateCTM(ctx, -1 * self.pointToMagnify.x, -1 * self.pointToMagnify.y);
		[CJkeyWindow().layer renderInContext:ctx];
		CJkeyWindow().layer.contents = (id)nil;
	}

	@end

	/**
	 长按时候显示的放大镜视图
	 */
	@interface CJMagnifierView ()
	@property (nonatomic, assign) CGPoint pointToMagnify;//放大点
	@property (strong, nonatomic) CJContentLayer *contentLayer;//处理放大效果的layer层

	- (void)updateMagnifyPoint:(CGPoint)pointToMagnify showMagnifyViewIn:(CGPoint)showPoint;

	@end

在更改放大点的时候主动调用`[self.contentLayer setNeedsDisplay];`那么就会触发`CJContentLayer `的`-drawInContext:`方法，这样也就达到了更改放大镜内容的效果。

第三部分大头针包含区域，同样自定义UIView的子类`CJSelectTextRangeView`，并且在其中设定三部分区域`headRect` `middleRect` `tailRect` 
![CJSelectTextRangeView.png](https://user-gold-cdn.xitu.io/2018/3/24/162584e258cb2af8?w=304&h=266&f=png&s=7418)
这三部分存在任意组合的情况，所以我们对这三部分区分开来，分别进行颜色填充的操作

	/**
	 大头针的显示类型
	 */
	typedef NS_ENUM(NSInteger, CJSelectViewAction) {
	    ShowAllSelectView    = 0,//显示大头针（长按或者双击）
	    MoveLeftSelectView   = 1,//移动左边大头针
	    MoveRightSelectView  = 2 //移动右边大头针
	};
	/**
	 选中复制填充背景色的view
	 */
	@interface CJSelectTextRangeView : UIView
	/**
	 前半部分选中区域
	 */
	@property (nonatomic, assign) CGRect headRect;
	/**
	 中间部分选中区域
	 */
	@property (nonatomic, assign) CGRect middleRect;
	/**
	 后半部分选中区域
	 */
	@property (nonatomic, assign) CGRect tailRect;
	/**
	 选择内容是否包含不同行
	 */
	@property (nonatomic, assign) BOOL differentLine;
	- (void)updateFrame:(CGRect)frame headRect:(CGRect)headRect middleRect:(CGRect)middleRect tailRect:(CGRect)tailRect differentLine:(BOOL)differentLine;
	@end
	@implementation CJSelectTextRangeView

	- (instancetype)init {
	    self = [super init];
	    if (self) {
	        self.backgroundColor = [UIColor clearColor];
	        self.opaque = NO;
	    }
	    return self;
	}

	- (void)updateFrame:(CGRect)frame headRect:(CGRect)headRect middleRect:(CGRect)middleRect tailRect:(CGRect)tailRect differentLine:(BOOL)differentLine {
	    self.differentLine = differentLine;
	    self.frame = frame;
	    self.headRect = headRect;
	    self.middleRect = middleRect;
	    self.tailRect = tailRect;
	    [self setNeedsDisplay];
	}

	- (void)drawRect:(CGRect)rect {

	    CGContextRef ctx = UIGraphicsGetCurrentContext();

	    //背景色
	    UIColor *backColor = CJUIRGBColor(0,84,166,0.2);
	    
	    if (self.differentLine) {
	        [backColor set];
	        CGContextAddRect(ctx, self.headRect);
	        if (!CGRectEqualToRect(self.middleRect,CGRectNull)) {
	            CGContextAddRect(ctx, self.middleRect);
	        }
	        CGContextAddRect(ctx, self.tailRect);
	        CGContextFillPath(ctx);
	        
	        [self updatePinLayer:ctx point:CGPointMake(self.headRect.origin.x, self.headRect.origin.y) height:self.headRect.size.height isLeft:YES];
	        
	        [self updatePinLayer:ctx point:CGPointMake(self.tailRect.origin.x + self.tailRect.size.width, self.tailRect.origin.y) height:self.tailRect.size.height isLeft:NO];
	    }else{
	        
	        [backColor set];
	        CGContextAddRect(ctx, self.middleRect);
	        CGContextFillPath(ctx);
	        
	        [self updatePinLayer:ctx point:CGPointMake(self.middleRect.origin.x, self.middleRect.origin.y) height:self.middleRect.size.height isLeft:YES];
	        
	        [self updatePinLayer:ctx point:CGPointMake(self.middleRect.origin.x + self.middleRect.size.width, self.middleRect.origin.y) height:self.middleRect.size.height isLeft:NO];
	    }
	    
	    CGContextStrokePath(ctx);
	}

	- (void)updatePinLayer:(CGContextRef)ctx point:(CGPoint)point height:(CGFloat)height isLeft:(BOOL)isLeft {
	    UIColor *color = [UIColor colorWithRed:0/255.0 green:128/255.0 blue:255/255.0 alpha:1.0];
	    CGRect roundRect = CGRectMake(point.x - 5,
	                                  isLeft?(point.y - 10):(point.y + height),
	                                  10,
	                                  10);
	    //画圆
	    CGContextAddEllipseInRect(ctx, roundRect);
	    [color set];
	    CGContextFillPath(ctx);
	    
	    CGContextMoveToPoint(ctx, point.x, point.y);
	    CGContextAddLineToPoint(ctx, point.x, point.y + height);
	    CGContextSetLineWidth(ctx, 2.0);
	    CGContextSetStrokeColorWithColor(ctx, color.CGColor);
	    
	    CGContextStrokePath(ctx);
	}

	@end
接下来便是显示这三个选择复制相关的视图了，一开始我只是简单的将它们添加到自定义的`CJSelectBackView`上面，在将`CJSelectBackView` add 到CJLabel上面来统一管理的，但这样会存在一个问题。那就是当页面中存在多个CJLabel，并且对多个CJLabel分别执行选择复制操作时，那么不同的label上都会出现选择复制视图，这是与系统的默认行为不一致的。就算是页面内存在多个不同的UITextView，对不同的UITextView进行选择复制，系统给人的感觉是只会有一个选择控制视图存在。

权衡之后我选择将`CJSelectBackView`作为单例处理，全局只初始化一次，避免了重复初始化的开销。并且引入UIWindow层，在不同的CJLabel之间进行选择复制时，借助UIWindow来进行控制切换。
最终选择复制相关的层级结构如下

![CJLabelSelect.png](https://user-gold-cdn.xitu.io/2018/3/24/162584e288b57eae?w=900&h=603&f=png&s=246756)

最后便是一些优化操作了，比如选择复制拖动的时候处理UIScrollView滑动的手势冲突，左右大头针高度的调整等。

具体的实现可以查看源码[CJLabel](https://github.com/lele8446/CJLabel)，欢迎star以及issue