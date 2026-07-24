---
layout: default
title: 结合ImageFont 聊聊字体定位的一些知识
---
笔者近期参与一个课程制作的项目，在其中负责动画合成相关的模块。其中涉及到对文本的处理，包括下划线、行间距、字间距等。为了比较满意地实现需求，笔者接触到字体定位的一些知识。笔者之前一直从事服务功能和性能方面的开发维护，这些知识对于笔者比较新奇，因此记录一下。如下图所示，是一张使用思源宋体(SourceHanSerifCN)，fontSize为80，行间距1.6，带下划线和斜体效果图：
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/drawText1.png" height = "570" width="100%"/>

其中下划线、行间距(行高)的实现，都与字体的定位知识有关。笔者首先介绍字体定位相关的一些知识，然后简单介绍ImageFont模块，最后给出实现方法。在开始介绍之前，笔者先抛出问题，这些问题也是笔者在开始实现之前想不通的。  
1. 如何确定一行文本的高度。因为不同的字符，高度不一样，比如A和a，如果统一取fontSize作为高度，类似字符a将因占据较高高度而不优雅
2. 如何确定文本下划线的位置。因为不同的字符，底线不一样，比如g和a，如果统一以字符g底部为下划线的位置，字符a就会距离下划线较远。反之则字符g有一部分在下划线下边。无论哪种实现，都不太优雅

最终为了更好地呈现文本效果，笔者实现每行高度根据文本内容自适应，恰好“装下”这行。下划线的位置也和文本内容自适应，刚刚“挨着”“底最长”的字符。

字体定位知识
====
每种字体都有一个或多个对应的字体文件，其中定义了字体样式、字体定位以及支持哪些字符等信息。下图是一张有关字体定位的参考图，几乎所有字体文件都包括这些定位信息，有些字体文件会在此基础上添加更多的定位信息。
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/drawText2.svg" height = "350" width="100%"/>  

由图可见，定位分为水平和垂直两个方向。水平方向包括左边界线(left)、中线(middle)和右边界线(right)。垂直方向定位线稍多一些，分为上升线(Ascenter)、顶线(top)、中线(middle)、基线(baseline)、底线(bottom)和下降线(Descenter)。其中上升线、基线和下降线只和字体本身有关，而其它定位位置不仅与字体本身，还有文本内容有关。这些信息都在字体文件中定义，一般的文本绘制框架都可以读取，笔者后面结合ImageFont具体讲解。水平方向三条定位线，和垂直方向六条定位线，共有3*6个交点，专业术语叫锚点，任何一个锚点都可用于文本定位，叫为文本锚点。线和文本锚点一起决定文本的定位。一旦确定区域大小，选定文本锚点并指定坐标，文本在区域的绘制效果随之确认。  

图中的三条蓝色实线分别是上升线、基线和下降线。字体中字符数量成千上万，有些字符“腿”长，需要占用一定的底部空间，如字符g、y。有些字符“头”比较长，需要占用一定顶部空间，相对于a、c这些小写字母，大多数汉字都属于“头”长类型。字体设计者为了不同的字符在一起显示的效果，便规定了这三条线。绝大多数字符都是挨着基线，一些“腿长”的字符可以到基线下面，但最大不能超过descender。同理，有些字符“头部”比较长，但是最大也只能到ascender。上升线和下降线都是相对基线。基线、上升线和下降线的具体含义如下：  
* 基 线——字符集中的绝大多数字符都以基线为往上
* 上升线—— 字符集中最高的字符，可以达到的最大高度
* 下降线—— 字符集中最低的字符，可以达到的最大底部

笔者之前误以为，一个fontSize为80的文本，所占宽高小于等于80px。其实这是错误的。文本所占宽高和fontSize没有直接关系，文本的宽高超过fontSize的情形并不少见。实际上，要想完整地绘制某种字体下fontSize大小的任意文本，区域高度应为对应字体的Ascenter+Descenter。那么Ascenter、Descenter如何获取、文本锚点如何选择、锚点坐标如何指定就需要结合具体框架来介绍。笔者使用的是ImageFont。  

ImageFont模块
====
ImageFont是开源库Pillow下的一个模块，用于处理字体和文本渲染。ImageFont允许用户加载指定的字体，并绘制文本最终生成图像。笔者正是使用ImageFont来实现文本绘制需求。ImageFont加载一个字体和绘制文本的代码如下：  
```
    from PIL import ImageFont, Image, ImageDraw
    
    # 加载指定字体文件，并指定fontSize
    font = ImageFont.truetype(“./SourceHanSerifCN-400.otf", 80)
    ascent, descent = font.getmetrics()
    
    # 计算文本实际占用的宽、高
    text = “abcsd"
    box = font.getbbox(text)
    width =box[2] -box[0]
    height =box[3] -box[1]
    
    # 指定框大小，然后在框中绘制文本
    image = Image.new('RGBA', (width, ascent + descent), "red")
    draw = ImageDraw.Draw(image)
    
    draw.text((0, 0), text, font=font, fill='black', anchor=“la")
    
    image.show()
```
注意其中 ImageFont.truetype、font.getmetrics()以及font.getbbox(text)方法，它们返回的就是具体定位信息。ImageFont.truetype用于加载字体文件，并指定fontSize大小。font.getmetrics()获取在特定fontSize下字体的上升线(Ascenter)和下降线(Descenter)的值。getbbox(text)返回一个四个元素的数组，在笔者看过的绝大多数资料中，都被用于计算文本实际占用宽高，而忽略了数组中每个元素的真实含义。实际上getbbox(text)返回结果是以上升线和左边界线为横纵坐标轴的两个点坐标(x1,y1), (x2,y2), 分别是文本左边界线(left)和顶线(top)的交点，以及文本右边界线(right)和底线(bottom)的交点。  

draw.text作用之一选择文本锚点坐标，相关参数是xy和anchor。xy定义锚点坐标，anchor选择上述锚点中哪一个作文文本锚点由两个字符组成，第一个字符定义水平方向，第二个字符用于定义垂直方向，取上图括号中字符即可，默认是la，表示以左边界和上升线的交点作为文本锚点。  

到此为止，笔者已经介绍获取文本真实宽高。如何在真实高度而不是Ascenter+Descenter中完整绘制文本，不发生“越界”，关键就是理解文本锚点坐标在区域之外。笔者选择la作为文本锚点，坐标为(0, -y1)(y1是getbox结果的第二个元素)，如下所示：  
```
    from PIL import ImageFont, Image, ImageDraw
    
    font = ImageFont.truetype("./SourceHanSerifCN-400.otf", 80)
    ascent, descent = font.getmetrics()
    
    # 计算文本实际占用的宽、高
    text = "AaBbCcDdGg"
    box = font.getbbox(text)
    width =box[2] -box[0]
    height =box[3] -box[1]
    
    # 指定框大小，然后在框中绘制文本, 预留5px的buff
    image = Image.new('RGBA', (width, height + 5), "red")
    draw = ImageDraw.Draw(image)
    
    #选择la作为文本锚点,锚点坐标(0, -box[1])
    draw.text((0, -box[1]), text, font=font, fill='black', anchor='la')
    draw.line(((0, height), (width, height)), fill='black', width=3)
    
    image.show()
```
效果如下图所示:
<img src="http://glsp-img.unipus.cn/glsp/prod/public/avatar/image/20240204/drawText4.png" height = "500" width="100%"/>  

笔者写了一个绘制文本的方法，支持外描边、背景色、字间距以及行间距等特性，并且实现自动换行。已部署服务，亲测可用，代码见[github](https://github.com/fsxtiger/code/blob/master/script/textDraw.py)。