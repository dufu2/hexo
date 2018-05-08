---
title: CJLabel第二章——图文混排及精确点击区域
date: 2017-05-03
updated: 2017-05-03
---

相关文章介绍：<br>
[CJLabel第一章——富文本显示及任意链点点击](http://lele8446.top/2016/04/20/CJLabel第一章——UILabel富文本显示以及任意字符的点击响应/)<br>
[CJLabel第三章——支持任意区域点击响应和可选择复制原理](http://lele8446.top/2017/11/14/CJLabel第三章——支持任意区域点击响应和可选择复制原理/)

[CJLabel](https://github.com/lele8446/CJLabelTest)在`V2.0.0`版本之前，其图文显示是基于NSAttributedString来实现的，但有若干不足：

1. 图片点击响应只支持emoji表情，对于NSAttributedString中通过NSTextAttachment添加的图片，无法响应点击。
2. 点击链点的判断：核心方法是调用`CTLineGetStringIndexForPosition()`获取当前触摸点CGPoint在NSString中对应的index，再判断该index是否属于点击链点的NSRange。然而当文本中包含多个emoji表情时，`CTLineGetStringIndexForPosition()`的获取并不准确，特别是如果当前点击链点处在文本的最后一行时，存在较大的误差。
3. 无法响应长按点击，另外点击链点不能设置点击高亮属性。

<!-- more -->

*针对以上问题，我对CJLabel 进行了重构优化，下面就来看看[CJLabel](https://github.com/lele8446/CJLabelTest) 新的实现。*


![CJLabel1.gif](http://upload-images.jianshu.io/upload_images/1429982-ad29e6db37fc95ea.gif?imageMogr2/auto-orient/strip)

![CJLabel2.gif](http://upload-images.jianshu.io/upload_images/1429982-279e01b2aceba923.gif?imageMogr2/auto-orient/strip)
***
开始之前先来看几张图（某些图片来自网络）
#####  字形与字符
![字形描述属性.jpg](http://upload-images.jianshu.io/upload_images/1429982-692b028cf512d627.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. Bounding Box（边界框 bbox），这是一个假想的框子，它尽可能紧密的将字形包括。
2. Baseline（基线），一条假想的线,一行上的字形都以此线作为上下位置的参考。
3. Origin（每一行的原点），Origin是在图中的Baseline的左侧。
4. Ascent（上行高度）从原点到字体中最高（这里的高深都是以基线为参照线的）的字形的顶部的距离，Ascent是一个正值。
5. Descent（下行高度）从原点到字体中最深的字形底部的距离，Descent是一个负值。
6. Linegap（行距），Linegap也可以称作leading（其实准确点讲应该叫做External leading）。
</br>
![字符描述属性.jpg](http://upload-images.jianshu.io/upload_images/1429982-fcb560ae7ef3d7fb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中绿色线条表示基线，黄色线条表示下行高度，绿色线条到红框最顶部的距离为上行高度，而黄色线条到红框底部的距离为行间距。因此行高的计算公式是 lineHeight = Ascent + |Descent| + Leading

##### CoreText绘制NSAttributedString

![CTFrame.png](http://upload-images.jianshu.io/upload_images/1429982-2de2c08aad8cfd0a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
CoreText是iOS/OSX里的文字渲染引擎，它会把一行里连在一起相同属性的文字合在一起作为一个CTRun，每一行是一个CTLine，多行合在一起组成CTFrame。如上图，第一行的文字有两种样式，第一部分是加粗，第二部分是斜体，因为样式不同所以分成了两个CTRun，CTLine包含了这两个CTRun，CTFrame包含了所有CTLine。

![](http://upload-images.jianshu.io/upload_images/1429982-c88c144075d470a9.png!web?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
NSAttributeString可以通过CoreText提供的方法生成CTFramesetter，CTFramesetter是用于创建CTFrame的工厂，给CTFramesetter一个CGPath，或者简单理解为给他一个框框，它就会通过它持有的CTTypesetter生成CTFrame，CTFrame生成时里面包含的CTLine和CTRun就全部生成好了，可以直接绘制到画布上。CTFrame/CTLine/CTRun都提供了渲染接口，但前两者是封装，最后实际都是调用到CTRun的渲染接口去绘制。

新建UILabel子类，重写`- (void)drawTextInRect:(CGRect)rect`绘制NSAttributeString

![测试.png](http://upload-images.jianshu.io/upload_images/1429982-a0cdb30ffb464c11.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

    - (void)drawTextInRect:(CGRect)rect {
	    NSMutableAttributedString *attributedText = [[NSMutableAttributedString alloc]initWithString:@"这是一段测试数据"];
	    [attributedText addAttribute:NSForegroundColorAttributeName value:[UIColor blueColor] range:NSMakeRange(2, 2)];
	    [attributedText addAttribute:NSFontAttributeName value:[UIFont boldSystemFontOfSize:15] range:NSMakeRange(2, 2)];
	    
	    CGContextRef c = UIGraphicsGetCurrentContext();
	    // 先将当前图形状态推入堆栈
	    CGContextSaveGState(c);
	    // 设置字形变换矩阵为CGAffineTransformIdentity，也就是说每一个字形都不做图形变换
	    CGContextSetTextMatrix(c, CGAffineTransformIdentity);
	    // 坐标转换，iOS 坐标原点在左上角，Mac OS 坐标原点在左下角
	    CGContextTranslateCTM(c, 0.0f, rect.size.height);
	    // CTM 坐标移翻转
	    CGContextScaleCTM(c, 1.0f, -1.0f);
	    
	    //生成 CTFramesetterRef
	    CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)attributedText);
	    CFRange textRange = CFRangeMake(0, (CFIndex)[attributedText length]);
	    // 获取path
	    CGMutablePathRef path = CGPathCreateMutable();
	    CGPathAddRect(path, NULL, rect);
	    //生成 CTFrameRef
	    CTFrameRef frame = CTFramesetterCreateFrame(framesetter, textRange, path, NULL);
	    //获取所有行
	    CFArrayRef lines = CTFrameGetLines(frame);
	    //原点数组
	    CGPoint lineOrigins[CFArrayGetCount(lines)];
	    //获取所有CTLineRef的原点
	    CTFrameGetLineOrigins(frame, CFRangeMake(0, CFArrayGetCount(lines)), lineOrigins);
	    for (CFIndex lineIndex = 0; lineIndex < CFArrayGetCount(lines); lineIndex++) {
	        CGPoint lineOrigin = lineOrigins[lineIndex];
	        CGContextSetTextPosition(c, lineOrigin.x, lineOrigin.y);
	        CTLineRef line = CFArrayGetValueAtIndex(lines, lineIndex);
	        //绘制当前行
	        CTLineDraw(line, c);
	    }
	    CFRelease(frame);
	    CGPathRelease(path);
	    CFRelease(framesetter);
	    CGContextRestoreGState(c);
	}
##### NSAttributedString图文混排

![插入图片.jpg](http://upload-images.jianshu.io/upload_images/1429982-5b685d4bdf7a18fb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
NSAttributedString插入图片可以通过`CTRunDelegateCallbacks`来实现，如上图所示，蓝色CTRun设置了`CTRunDelegateRef`，并在需要插入图片的位置以空白字符串代替。CoreText逐行绘制文字，当绘制到蓝色CTRun时，会触发`CTRunDelegateCallbacks`的相关回调：

    void RunDelegateDeallocCallback(void * refCon) {
	}
	//获取图片高度
	CGFloat RunDelegateGetAscentCallback(void * refCon) {
	    return [(NSNumber *)[(__bridge NSDictionary *)refCon objectForKey:kCJImageHeight] floatValue];
	}
	CGFloat RunDelegateGetDescentCallback(void * refCon) {
	    return 0;
	}
	//获取图片宽度
	CGFloat RunDelegateGetWidthCallback(void * refCon) {
	    return [(NSNumber *)[(__bridge NSDictionary *)refCon objectForKey:kCJImageWidth] floatValue];
	}
在指定位置插入图片

    /**
	 指定位置插入图片

	 @param line 当前绘制行
	 @param c CGContextRef
	 @param lineOrigins CTLine坐标原点
	 @param lineIndex 第几行
	 */
	- (void)drawImageLine:(CTLineRef)line
	              context:(CGContextRef)c
	          lineOrigins:(CGPoint[])lineOrigins
	            lineIndex:(CFIndex)lineIndex
	{
	    CFArrayRef runs = CTLineGetGlyphRuns(line);
	    for (int j = 0; j < CFArrayGetCount(runs); j++) {
	        CGFloat runAscent;
	        CGFloat runDescent;
	        CGPoint lineOrigin = lineOrigins[lineIndex];
	        //获取每个CTRun
	        CTRunRef run = CFArrayGetValueAtIndex(runs, j);
	        NSDictionary* attributes = (NSDictionary*)CTRunGetAttributes(run);
	        CGRect runRect;
	        //调整CTRun的rect
	        runRect.size.width = CTRunGetTypographicBounds(run, CFRangeMake(0,0), &runAscent, &runDescent, NULL);
	        
	        runRect = CGRectMake(lineOrigin.x + CTLineGetOffsetForStringIndex(line, CTRunGetStringRange(run).location, NULL), lineOrigin.y - runDescent, runRect.size.width, runAscent + runDescent);
	        
	        NSDictionary *imgInfoDic = attributes[kCJImageAttributeName];
	        if (imgInfoDic[kCJImageName]) {
	            UIImage *image = [UIImage imageNamed:imgInfoDic[kCJImageName]];
	            if (image) {
	                CGRect imageDrawRect;
	                CGFloat imageSizeWidth = ceil(runRect.size.width);
	                CGFloat imageSizeHeight = ceil(runRect.size.height);
	                imageDrawRect.size = CGSizeMake(imageSizeWidth, imageSizeHeight);
	                imageDrawRect.origin.x = runRect.origin.x + lineOrigin.x;
	                imageDrawRect.origin.y = lineOrigin.y;
	                CGContextDrawImage(c, imageDrawRect, image.CGImage);
	            }
	        }
	    }
	}
	
***

####  CJLabel点击链点的判断

前面我们了解了CoreText在UILabel上面的图文混排绘制的过程，而且在绘制过程中细化到了每一行`CTLine`的`CTRun`，那何不将点击链点对应的`CTRun`也做上标记；在绘制的过程中判断到当前`CTRun `是点击链点则将其对应的`CGRect`保存到NSArray中。
再在`- touchesBegan: withEvent:`事件中判断触摸点是否在NSArray里的`CGRect `里面，核心调用方法`CGRectContainsPoint(bounds, point)`。
这样就彻底告别了`CTLineGetStringIndexForPosition()`的判断逻辑，而且`CGRectContainsPoint `的判断准确度更高。

对于如何标记点击链点的`CTRun `，这就轮到`NSMutableAttributedString`上场了，调用它的实例方法`- (void)addAttribute:(NSString *)name value:(id)value range:(NSRange)range`，可以在指定位置添加自定义属性（前面说到的标记插入图片，也就是这样做的）。

***另外CJLabel新增了以下属性***

新增属性能够对指定位置的字符串添加背景色、描边、点击高亮的效果。

    /**
	 背景填充颜色。值为UIColor。默认 `nil`。
	 该属性优先级低于NSBackgroundColorAttributeName，如果设置NSBackgroundColorAttributeName会覆盖kCJBackgroundFillColorAttributeName
	 */
	extern NSString * const kCJBackgroundFillColorAttributeName;

	/**
	 背景边框线颜色。值为UIColor。默认 `nil`
	 */
	extern NSString * const kCJBackgroundStrokeColorAttributeName;

	/**
	 背景边框线宽度。值为NSNumber。默认 `1.0f`
	 */
	extern NSString * const kCJBackgroundLineWidthAttributeName;

	/**
	 背景边框线圆角角度。值为NSNumber。默认 `5.0f`
	 */
	extern NSString * const kCJBackgroundLineCornerRadiusAttributeName;

	/**
	 点击时候的背景填充颜色。值为UIColor。默认 `nil`。
	 该属性优先级低于NSBackgroundColorAttributeName，如果设置NSBackgroundColorAttributeName会覆盖kCJActiveBackgroundFillColorAttributeName
	 */
	extern NSString * const kCJActiveBackgroundFillColorAttributeName;

	/**
	 点击时候的背景边框线颜色。值为UIColor。默认 `nil`
	 */
	extern NSString * const kCJActiveBackgroundStrokeColorAttributeName;
	
***

####  CJLabel实现逻辑
实现逻辑总结如下：

1. 在NSAttributedString中对点击链点、插入图片、添加圆角背景或者描边的字符进行标记；
2. 根据NSAttributedString中的标记，获取对应字符在label中的CGRect、自定义参数、图片名称等信息，并保存在NSArray中；
3. 逐行绘制CTLine。CoreText先在指定位置填充背景色，然后绘制文字，接着在插入图片的位置绘制图片，最后在指定位置添加描边（CoreText绘制是叠加覆盖的，要注意层级关系）；
4. 点击label，在`- touchesBegan:withEvent:`以及`UILongPressGestureRecognizer`长按回调事件中，将触摸点CGPoint与CGRect数组做遍历判断`CGRectContainsPoint(bounds, point)`，找到对应字符并响应回调事件。

当然实现细节中还包括了不少其他的复杂逻辑处理，这里就不一一说明了，有兴趣的可以下载源码看看。
***
####  CJLabel类
CJLabel响应点击以及长按点击事件，回调事件可选择`CJLabelLinkModelBlock `或`CJLabelLinkDelegate`来实现。

    @protocol CJLabelLinkDelegate <NSObject>
    @optional
    /**
     点击链点回调

     @param label 点击label
     @param linkModel 链点model
     */
    - (void)CJLable:(CJLabel *)label didClickLink:(CJLabelLinkModel *)linkModel;

    /**
     长按点击链点回调

     @param label 点击label
     @param linkModel 链点model
     */
    - (void)CJLable:(CJLabel *)label didLongPressLink:(CJLabelLinkModel *)linkModel;
    @end


    IB_DESIGNABLE
    /**
     * CJLabel 继承自 UILabel，其文本绘制基于NSAttributedString实现，同时增加了图文混排、富文本展示以及添加自定义点击链点并设置点击链点文本属性的功能。
     *
     *
     * CJLabel 与 UILabel 不同点：
     *
       1. `- init` 不可直接调用init初始化，请使用`initWithFrame:` 或 `initWithCoder:`，以便完成相关初始属性设置
     
       2. `attributedText` 与 `text` 均可设置文本，注意 [self setText:text]中 text类型只能是NSAttributedString或NSString
     
       3. `NSAttributedString`不再通过`NSTextAttachment`显示图片（使用`NSTextAttachment`不会起效），请调用
          `- configureAttributedString: addImageName: imageSize: atIndex: attributes:`或者
          `- configureLinkAttributedString: addImageName: imageSize: atIndex: linkAttributes: activeLinkAttributes: parameter: clickLinkBlock: longPressBlock:`方法添加图片
     
       4. 新增`extendsLinkTouchArea`， 设置是否加大点击响应范围，类似于UIWebView的链点点击效果
     
       5. 新增`shadowRadius`， 设置文本阴影模糊半径，可与 `shadowColor`、`shadowOffset` 配合设置，注意改设置将对全局文本起效
     
       6. 新增`textInsets` 设置文本内边距
     
       7. 新增`verticalAlignment` 设置垂直方向的文本对齐方式
     
       8. 新增`delegate` 点击链点代理
     *
     *
     * CJLabel 已知bug：
     *
       `numberOfLines`大于0且小于实际`label.numberOfLines`，同时`verticalAlignment`不等于`CJContentVerticalAlignmentTop`时，文本显示位置有偏差
     *
     */
    @interface CJLabel : UILabel

    /**
     * 指定初始化函数为 initWithFrame: 或 initWithCoder:
     * 直接调用 init 会忽略相关属性的设置，所以不能直接调用 init 初始化.
     */
    - (instancetype)init NS_UNAVAILABLE;

    /**
     * 对应UILabel的attributedText属性
     */
    @property (readwrite, nonatomic, copy) NSAttributedString *attributedText;
    /**
     * 对应UILabel的text属性
     */
    @property (readwrite, nonatomic, copy) id text;
    /**
     * 是否加大点击响应范围，类似于UIWebView的链点点击效果，默认NO
     */
    @property (readwrite, nonatomic, assign) IBInspectable BOOL extendsLinkTouchArea;
    /**
     * 阴影模糊半径，值为0表示没有模糊，值越大越模糊，该值不能为负， 默认值为0。
     * 可与 `shadowColor`、`shadowOffset` 配合设置
     */
    @property (readwrite, nonatomic, assign) CGFloat shadowRadius;
    /**
     * 绘制文本的内边距，默认UIEdgeInsetsZero
     */
    @property (readwrite, nonatomic, assign) UIEdgeInsets textInsets;
    /**
     * 当text rect 小于 label frame 时文本在垂直方向的对齐方式，默认 CJVerticalAlignmentCenter
     */
    @property (readwrite, nonatomic, assign) CJLabelVerticalAlignment verticalAlignment;
    /**
     点击链点代理对象
     */
    @property (readwrite, nonatomic, weak) id<CJLabelLinkDelegate> delegate;

    /**
     *  return 计算NSAttributedString字符串的size大小
     *
     *  @param attributedString NSAttributedString字符串
     *  @param size   预计大小（比如：CGSizeMake(320, CGFLOAT_MAX)）
     *  @param numberOfLines  指定行数（0表示不限制）
     *
     *  @return 结果size
     */
    + (CGSize)sizeWithAttributedString:(NSAttributedString *)attributedString
                       withConstraints:(CGSize)size
                limitedToNumberOfLines:(NSUInteger)numberOfLines;

    // 在指定位置插入图片，图片所在行，图文在垂直方向的对齐方式默认：底部对齐
    + (NSMutableAttributedString *)configureAttributedString:(NSAttributedString *)attrStr
                                                addImageName:(NSString *)imageName
                                                   imageSize:(CGSize)size
                                                     atIndex:(NSUInteger)loc
                                                  attributes:(NSDictionary *)attributes;

    // 在指定位置插入可点击链点图片，图片所在行，图文在垂直方向的对齐方式默认：底部对齐
    + (NSMutableAttributedString *)configureLinkAttributedString:(NSAttributedString *)attrStr
                                                    addImageName:(NSString *)imageName
                                                       imageSize:(CGSize)size
                                                         atIndex:(NSUInteger)loc
                                                  linkAttributes:(NSDictionary *)linkAttributes
                                            activeLinkAttributes:(NSDictionary *)activeLinkAttributes
                                                       parameter:(id)parameter
                                                  clickLinkBlock:(CJLabelLinkModelBlock)clickLinkBlock
                                                  longPressBlock:(CJLabelLinkModelBlock)longPressBlock;

    /**
     在指定位置插入图片，并返回插入图片后的NSMutableAttributedString（图片占位符所占的NSRange={loc,1}）
     
     注意！！！插入图片， 如果设置 NSParagraphStyleAttributeName 属性，例如:
     NSMutableParagraphStyle *paragraph = [[NSMutableParagraphStyle alloc] init];
     paragraph.lineBreakMode = NSLineBreakByCharWrapping;
     [attrStr addAttribute:NSParagraphStyleAttributeName value:paragraph range:range];
     请保证 paragraph.lineBreakMode = NSLineBreakByCharWrapping，不然当Label的宽度不够显示内容或图片时，不会自动换行, 部分图片将会看不见
        
     默认 paragraph.lineBreakMode = NSLineBreakByCharWrapping
     
     @param attrStr 需要插入图片的NSAttributedString
     @param imageName 图片名称
     @param size 图片大小
     @param loc 图片插入位置
     @param verticalAlignment 图片所在行，图片与文字在垂直方向的对齐方式（只针对当前行）
     @param attributes 图片文本属性
     
     @return 插入图片后的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureAttributedString:(NSAttributedString *)attrStr
                                                addImageName:(NSString *)imageName
                                                   imageSize:(CGSize)size
                                                     atIndex:(NSUInteger)loc
                                           verticalAlignment:(CJLabelVerticalAlignment)verticalAlignment
                                                  attributes:(NSDictionary *)attributes;

    /**
     在指定位置插入图片，插入图片为可点击的链点！！！
     返回插入图片后的NSMutableAttributedString（图片占位符所占的NSRange={loc,1}）
     
     @param attrStr 需要插入图片的NSAttributedString
     @param imageName 图片名称
     @param size 图片大小
     @param loc 图片插入位置
     @param verticalAlignment 图片所在行，图片与文字在垂直方向的对齐方式（只针对当前行）
     @param linkAttributes 图片链点属性
     @param activeLinkAttributes 点击状态下的图片链点属性
     @param parameter 链点自定义参数
     @param clickLinkBlock 链点点击回调
     @param longPressBlock 长按点击链点回调
     
     @return 插入图片后的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureLinkAttributedString:(NSAttributedString *)attrStr
                                                    addImageName:(NSString *)imageName
                                                       imageSize:(CGSize)size
                                                         atIndex:(NSUInteger)loc
                                               verticalAlignment:(CJLabelVerticalAlignment)verticalAlignment
                                                  linkAttributes:(NSDictionary *)linkAttributes
                                            activeLinkAttributes:(NSDictionary *)activeLinkAttributes
                                                       parameter:(id)parameter
                                                  clickLinkBlock:(CJLabelLinkModelBlock)clickLinkBlock
                                                  longPressBlock:(CJLabelLinkModelBlock)longPressBlock;

    /**
     根据指定NSRange配置富文本
     
     @param attrStr NSAttributedString源
     @param range 指定NSRange
     @param attributes 文本属性
     
     @return 返回新的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureAttributedString:(NSAttributedString *)attrStr
                                                     atRange:(NSRange)range
                                                  attributes:(NSDictionary *)attributes;

    /**
     根据指定NSRange配置富文本，指定NSRange文本为可点击链点！！！
     
     @param attrStr NSAttributedString源
     @param range 指定NSRange
     @param linkAttributes 链点文本属性
     @param activeLinkAttributes 点击状态下的链点文本属性
     @param parameter 链点自定义参数
     @param clickLinkBlock 链点点击回调
     @param longPressBlock 长按点击链点回调
     
     @return 返回新的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureLinkAttributedString:(NSAttributedString *)attrStr
                                                         atRange:(NSRange)range
                                                  linkAttributes:(NSDictionary *)linkAttributes
                                            activeLinkAttributes:(NSDictionary *)activeLinkAttributes
                                                       parameter:(id)parameter
                                                  clickLinkBlock:(CJLabelLinkModelBlock)clickLinkBlock
                                                  longPressBlock:(CJLabelLinkModelBlock)longPressBlock;

    /**
     对文本中跟withString相同的文字配置富文本
     
     @param attrStr NSAttributedString源
     @param withString 需要设置的文本
     @param sameStringEnable 文本中所有与withAttString相同的文字是否同步设置属性，sameStringEnable=NO 时取文本中首次匹配的NSAttributedString
     @param attributes 文本属性
     
     @return 返回新的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureAttributedString:(NSAttributedString *)attrStr
                                                  withString:(NSString *)withString
                                            sameStringEnable:(BOOL)sameStringEnable
                                                  attributes:(NSDictionary *)attributes;

    /**
     对文本中跟withString相同的文字配置富文本，指定的文字为可点击链点！！！
     
     @param attrStr NSAttributedString源
     @param withString 需要设置的文本
     @param sameStringEnable 文本中所有与withAttString相同的文字是否同步设置属性，sameStringEnable=NO 时取文本中首次匹配的NSAttributedString
     @param linkAttributes 链点文本属性
     @param activeLinkAttributes 点击状态下的链点文本属性
     @param parameter 链点自定义参数
     @param clickLinkBlock 链点点击回调
     @param longPressBlock 长按点击链点回调
     
     @return 返回新的NSMutableAttributedString
     */
    + (NSMutableAttributedString *)configureLinkAttributedString:(NSAttributedString *)attrStr
                                                      withString:(NSString *)withString
                                                sameStringEnable:(BOOL)sameStringEnable
                                                  linkAttributes:(NSDictionary *)linkAttributes
                                            activeLinkAttributes:(NSDictionary *)activeLinkAttributes
                                                       parameter:(id)parameter
                                                  clickLinkBlock:(CJLabelLinkModelBlock)clickLinkBlock
                                                  longPressBlock:(CJLabelLinkModelBlock)longPressBlock;

    /**
     *  移除制定range的点击链点
     *
     *  @param range 移除链点位置
     *
     *  @return 返回新的NSAttributedString
     */
    - (NSAttributedString *)removeLinkAtRange:(NSRange)range;

    /**
     *  移除所有点击链点
     *
     *  @return 返回新的NSAttributedString
     */
    - (NSAttributedString *)removeAllLink;

    @end


#### CJLabel支持CocoaPods安装

    platform :ios, '7.0'
    target 'CJLabelDemo' do
       pod 'CJLabel', '~> 2.1.3'
    end

更多实现细节请看[Demo](https://github.com/lele8446/CJLabelTest)

附：
字形与字符的相关描述引用自 [http://blog.cnbang.net/tech/2729/](http://blog.cnbang.net/tech/2729/?utm_source=tuicool&utm_medium=referral)