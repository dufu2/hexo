---
title: 动态计算NSAttributedString的size大小
date: 2016-04-19
updated: 2016-04-19
---

NSAttributedString（富文本），作为NSString的子类，是一种带有属性的字符串，通过它可以轻松的在一个字符串中表现出多种字体、字号、背景色、下划线等各不相同的风格，还可以对段落进行格式化。下面就来探讨一下动态计算NSAttributedString的size大小实现：

<!-- more -->

****
* 首先提供一个对NSAttributedString进行封装的函数<br>
该方法会为NSAttributedString添加默认段落属性以及字体属性(如果不存在的话)

	      /**
	       *  return 返回封装后的NSMutableAttributedString,添加了默认NSParagraphStyleAttributeName与NSFontAttributeName属性
	       *
	       *  @param labelStr  NSString
	       *  @param labelDic  属性字典
	          @{
	             NSFontAttributeName://(字体)
	             NSBackgroundColorAttributeName://(字体背景色)
	             NSForegroundColorAttributeName://(字体颜色)
	             NSParagraphStyleAttributeName://(段落)
	             NSLigatureAttributeName://(连字符)
	             NSKernAttributeName://(字间距)
	             NSStrikethroughStyleAttributeName://NSUnderlinePatternSolid(实线) | NSUnderlineStyleSingle(删除线)
	             NSUnderlineStyleAttributeName://(下划线)
	             NSStrokeColorAttributeName://(边线颜色)
	             NSStrokeWidthAttributeName://(边线宽度)
	             NSShadowAttributeName://(阴影)
	             NSVerticalGlyphFormAttributeName://(横竖排版)
	           }
	        *
	        *  @return NSMutableAttributedString
	        */
	      + (NSMutableAttributedString *)getNSAttributedString:(NSString *)labelStr labelDict:(NSDictionary *)labelDic
	      {
	         NSMutableAttributedString *atrString = [[NSMutableAttributedString alloc] initWithString:labelStr];
	         NSRange range = NSMakeRange(0, atrString.length);
	         if (labelDic && labelDic.count > 0) {
	             NSEnumerator *enumerator = [labelDic keyEnumerator];
	             id key;
	             while ((key = [enumerator nextObject])) {
	                 [atrString addAttribute:key value:labelDic[key] range:range];
	             }
	         }
	         //段落属性
	         NSMutableParagraphStyle *paragraphStyle = labelDic[NSParagraphStyleAttributeName];
	         if (!paragraphStyle || nil == paragraphStyle) {
	             paragraphStyle = [[NSParagraphStyle defaultParagraphStyle] mutableCopy];
	             paragraphStyle.lineSpacing = 0.0;//增加行高
	             paragraphStyle.headIndent = 0;//头部缩进，相当于左padding
	             paragraphStyle.tailIndent = 0;//相当于右padding
	             paragraphStyle.lineHeightMultiple = 0;//行间距是多少倍
	             paragraphStyle.alignment = NSTextAlignmentLeft;//对齐方式
	             paragraphStyle.firstLineHeadIndent = 0;//首行头缩进
	             paragraphStyle.paragraphSpacing = 0;//段落后面的间距
	             paragraphStyle.paragraphSpacingBefore = 0;//段落之前的间距
	             [atrString addAttribute:NSParagraphStyleAttributeName value:paragraphStyle range:range];
	         }
	         //字体
	         UIFont *font = labelDic[NSFontAttributeName];
	         if (!font || nil == font) {
	             font = [UIFont fontWithName:@"HelveticaNeue" size:12.0];
	             [atrString addAttribute:NSFontAttributeName value:font range:range];
	         }
	         return atrString;
	      }

##### 使用boundingRectWithSize:options:attributes:context计算
系统提供了`- boundingRectWithSize:options:attributes:context:`方法来计算NSAttributedString的size大小,`- sizeWithFont:constrainedToSize:lineBreakMode:`已经被废弃了。

      /**
       *  return 动态返回字符串size大小
       *
       *  @param aString 字符串
       *  @param width   指定宽度
       *  @param height  指定宽度
       *
       *  @return CGSize
       */
      + (CGSize)getStringRect:(NSAttributedString *)aString width:(CGFloat)width height:(CGFloat)height
      {
         CGSize size = CGSizeZero;
         NSMutableAttributedString *atrString = [[NSMutableAttributedString alloc] initWithAttributedString:aString];
         NSRange range = NSMakeRange(0, atrString.length);
    
         //获取指定位置上的属性信息，并返回与指定位置属性相同并且连续的字符串的范围信息。
         NSDictionary* dic = [atrString attributesAtIndex:0 effectiveRange:&range];
         //不存在段落属性，则存入默认值
         NSMutableParagraphStyle *paragraphStyle = dic[NSParagraphStyleAttributeName];
         if (!paragraphStyle || nil == paragraphStyle) {
              paragraphStyle = [[NSParagraphStyle defaultParagraphStyle] mutableCopy];
              paragraphStyle.lineSpacing = 0.0;//增加行高
              paragraphStyle.headIndent = 0;//头部缩进，相当于左padding
              paragraphStyle.tailIndent = 0;//相当于右padding
              paragraphStyle.lineHeightMultiple = 0;//行间距是多少倍
              paragraphStyle.alignment = NSTextAlignmentLeft;//对齐方式
              paragraphStyle.firstLineHeadIndent = 0;//首行头缩进
              paragraphStyle.paragraphSpacing = 0;//段落后面的间距
              paragraphStyle.paragraphSpacingBefore = 0;//段落之前的间距
              [atrString addAttribute:NSParagraphStyleAttributeName value:paragraphStyle range:range];
          }
    
         //设置默认字体属性
         UIFont *font = dic[NSFontAttributeName];
         if (!font || nil == font) {
              font = [UIFont fontWithName:@"HelveticaNeue" size:12.0];
              [atrString addAttribute:NSFontAttributeName value:font range:range];
          }
    
         NSMutableDictionary *attDic = [NSMutableDictionary dictionaryWithDictionary:dic];
         [attDic setObject:font forKey:NSFontAttributeName];
         [attDic setObject:paragraphStyle forKey:NSParagraphStyleAttributeName];
    
         CGSize strSize = [[aString string] boundingRectWithSize:CGSizeMake(width, height)
                                                    options:NSStringDrawingUsesLineFragmentOrigin | NSStringDrawingUsesFontLeading
                                                 attributes:attDic
                                                    context:nil].size;
    
         size = CGSizeMake(CGFloat_ceil(strSize.width), CGFloat_ceil(strSize.height));
         return size;
      }
需要注意的是调用时，要选择NSStringDrawingUsesLineFragmentOrigin | NSStringDrawingUsesFontLeading选项，不然计算出来的高度不准确

##### 通过sizeToFit计算

      /**
       *  返回UILabel自适应后的size
       *
       *  @param aString 字符串
       *  @param width   指定宽度
       *  @param height  指定高度
       *
       *  @return CGSize
       */
      + (CGSize)sizeLabelToFit:(NSAttributedString *)aString width:(CGFloat)width height:(CGFloat)height {
         UILabel *tempLabel = [[UILabel alloc]initWithFrame:CGRectMake(0, 0, width, height)];
         tempLabel.attributedText = aString;
         tempLabel.numberOfLines = 0;
         [tempLabel sizeToFit];
         CGSize size = tempLabel.frame.size;
         size = CGSizeMake(CGFloat_ceil(size.width), CGFloat_ceil(size.height));
         return size;
      }
其实就是通过新建一个临时的UILabel，然后通过sizeToFit方法计算出合适的CGSize。

##### 通过CTFramesetter进行计算
* CTFramesetter
首先来了解一下CTFramesetter与NSAttributedString的关系。CTFramesetter是CTFrame的创建工厂，NSAttributedString需要通过CTFrame绘制到界面上，得到CTFramesetter后，创建path（绘制路径），然后得到CTFrame，最后通过CTFrameDraw方法绘制到界面上。如图：
  ![image](http://upload-images.jianshu.io/upload_images/1429982-4e090f1b0e3e7cef.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  CTFramesetter关联NSAttributedString，此时CTTypesetter实例将自动创建，它管理了字体。然后使用CTFramesetter 创建您要用于渲染文本的一个或多个帧。当创建帧时，指定一个用于此帧矩形内的子文本范围。Core Text 为每行文本自动创建一个CTLine ，并在CTLine内创建多个 CTRun文本分段，每个CTRun内的文本有着同样的格式。同时每个 CTRun 对象可以采用不同的属性，所以你可以精确的控制字距，连字，宽度，高度等更多属性。
  
* 字符（Character）和字形（Glyphs）<br>
看一下字形图：
![image](http://upload-images.jianshu.io/upload_images/1429982-cac872112562a19d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 1. Bounding Box（边界框 bbox），这是一个假想的框子，它尽可能紧密的装入字形。
 2. Baseline（基线），一条假想的线,一行上的字形都以此线作为上下位置的参考，在这条线的左侧存在一个点叫做基线的原点。
 3. Ascent（上行高度）从原点到字体中最高（这里的高深都是以基线为参照线的）的字形的顶部的距离，Ascent是一个正值。
 4. Descent（下行高度）从原点到字体中最深的字形底部的距离，Descent是一个负值（比如一个字体原点到最深的字形的底部的距离为2，那么Descent就为-2）。
 5. Linegap（行距），Linegap也可以称作leading（其实准确点讲应该叫做External leading）,行高LineHeight则可以通过 Ascent + |Descent| + Linegap 来计算。
 6. Origin（每一行的原点），Origin是在图中的baseLine处的。
* 计算行高
了解了以上知识点我们就来看一下通过CTFramesetter进行计算行高的实现<br>

	方法一，将每一行CTLine的行高相加得到最终高度：

	      CGFloat heightValue = 0;
	      //string 为要计算高的NSAttributedString
	      CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)string);
	        
	      //这里的高要设置足够大
	      CGFloat height = 10000;
	      CGRect drawingRect = CGRectMake(0, 0, width, height);
	      CGMutablePathRef path = CGPathCreateMutable();
	      CGPathAddRect(path, NULL, drawingRect);
	      CTFrameRef textFrame = CTFramesetterCreateFrame(framesetter,CFRangeMake(0,0), path, NULL);
	      CGPathRelease(path);
	      CFRelease(framesetter);
	      CFArrayRef lines = CTFrameGetLines(textFrame);
	      CGPoint lineOrigins[CFArrayGetCount(lines)];
	      CTFrameGetLineOrigins(textFrame, CFRangeMake(0, 0), lineOrigins);
	    
	      /******************
	       * 逐行lineHeight累加
	       ******************/
	      heightValue = 0;
	      for (int i = 0; i < CFArrayGetCount(lines); i++) {
	         CTLineRef line = CFArrayGetValueAtIndex(lines, i);
	         CGFloat lineAscent;//上行行高
	         CGFloat lineDescent;//下行行高
	         CGFloat lineLeading;//行距
	         CGFloat lineHeight;//行高
	         //获取每行的高度
	         CTLineGetTypographicBounds(line, &lineAscent, &lineDescent, &lineLeading);
	         lineHeight = lineAscent +  fabs(lineDescent) + lineLeading;
	         heightValue = heightValue + lineHeight;
	      }
	      heightValue = CGFloat_ceil(heightValue);

  方法二，最后一行原点y坐标加最后一行高度：

	      CGFloat heightValue = 0;
	      //string 为要计算高的NSAttributedString
	      CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)string);
	        
	      //这里的高要设置足够大
	      CGFloat height = 10000;
	      CGRect drawingRect = CGRectMake(0, 0, width, height);
	      CGMutablePathRef path = CGPathCreateMutable();
	      CGPathAddRect(path, NULL, drawingRect);
	      CTFrameRef textFrame = CTFramesetterCreateFrame(framesetter,CFRangeMake(0,0), path, NULL);
	      CGPathRelease(path);
	      CFRelease(framesetter);
	      CFArrayRef lines = CTFrameGetLines(textFrame);
	      CGPoint lineOrigins[CFArrayGetCount(lines)];
	      CTFrameGetLineOrigins(textFrame, CFRangeMake(0, 0), lineOrigins);
	      /******************
	       * 最后一行原点y坐标加最后一行下行行高跟行距
	       ******************/
	      heightValue = 0;
	      CGFloat line_y = (CGFloat)lineOrigins[CFArrayGetCount(lines)-1].y;  //最后一行line的原点y坐标
	      CGFloat lastAscent = 0;//上行行高
	      CGFloat lastDescent = 0;//下行行高
	      CGFloat lastLeading = 0;//行距
	      CTLineRef lastLine = CFArrayGetValueAtIndex(lines, CFArrayGetCount(lines)-1);
	      CTLineGetTypographicBounds(lastLine, &lastAscent, &lastDescent, &lastLeading);
	      //height - line_y为除去最后一行的字符原点以下的高度，descent + leading为最后一行不包括上行行高的字符高度
	      heightValue = height - line_y + (CGFloat)(fabs(lastDescent) + lastLeading);
	      heightValue = CGFloat_ceil(heightValue);

  方法三，使用CTFramesetterSuggestFrameSizeWithConstraints计算：

	      static inline CGSize CTFramesetterSuggestFrameSizeForAttributedStringWithConstraints(CTFramesetterRef framesetter, NSAttributedString *attributedString, CGSize size, NSUInteger numberOfLines) {
	         CFRange rangeToSize = CFRangeMake(0, (CFIndex)[attributedString length]);
	         CGSize constraints = CGSizeMake(size.width, 10000);
	    
         if (numberOfLines == 1) {
             // If there is one line, the size that fits is the full width of the line
             constraints = CGSizeMake(10000, 10000);
         } else if (numberOfLines > 0) {
             // If the line count of the label more than 1, limit the range to size to the number of lines that have been set
             CGMutablePathRef path = CGPathCreateMutable();
             CGPathAddRect(path, NULL, CGRectMake(0.0f, 0.0f, constraints.width, 10000));
             CTFrameRef frame = CTFramesetterCreateFrame(framesetter, CFRangeMake(0, 0), path, NULL);
             CFArrayRef lines = CTFrameGetLines(frame);
        
             if (CFArrayGetCount(lines) > 0) {
                 NSInteger lastVisibleLineIndex = MIN((CFIndex)numberOfLines, CFArrayGetCount(lines)) - 1;
                 CTLineRef lastVisibleLine = CFArrayGetValueAtIndex(lines, lastVisibleLineIndex);
            
                 CFRange rangeToLayout = CTLineGetStringRange(lastVisibleLine);
                 rangeToSize = CFRangeMake(0, rangeToLayout.location + rangeToLayout.length);
              }
        
              CFRelease(frame);
              CGPathRelease(path);
         }
    
         CGSize suggestedSize = CTFramesetterSuggestFrameSizeWithConstraints(framesetter, rangeToSize, NULL, constraints, NULL);
         return CGSizeMake(CGFloat_ceil(suggestedSize.width), CGFloat_ceil(suggestedSize.height));
      }

  调用方法：

	      //string 为要计算高的NSAttributedString
	      CTFramesetterRef framesetter = CTFramesetterCreateWithAttributedString((__bridge CFAttributedStringRef)string);
	      //预设size
	      CGSize size = CGSizeMake(width, 10000);
	      CGSize suggestedSize= CTFramesetterSuggestFrameSizeForAttributedStringWithConstraints(framesetter,string,size,1000);

* 写在最后
最后说一下，经测试发现，以上说的三种通过CTFramesetter来计算高度的方法，都会存在误差，表现为`UILabel显示时上下会有空白行，且留白范围与所显示内容呈递增关系`，具体原因未知，如果有理解的欢迎指正！