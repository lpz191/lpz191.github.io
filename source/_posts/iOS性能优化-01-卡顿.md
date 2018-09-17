title: iOS性能优化（一）—— 卡顿优化
tags:
  - iOS高级
  - 性能优化
categories:
  - iOS开发
date: 2017-04-01 01:50:56
---

<Excerpt in index | 首页摘要>
性能优化从哪些方面着手？
什么是CPU，什么是GPU？
列表卡顿的原因可能有哪些？你平时是怎么优化的？
遇到tableView卡顿嘛？会造成卡顿的原因大致有哪些？
<!-- more -->
<The rest of contents | 余下全文>

# CPU和GPU
在屏幕成像的过程中，CPU和GPU起着至关重要的作用。
* `CPU`——（`Central Processing Unit`，中央处理器）主要负责对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制（`Core Graphics`）。

* `GPU`——（`Graphics Processing Unit`，图形处理器）主要负责纹理的渲染。
APP的显示一般是由`CPU`和`GPU`协同处理的。
`CPU`负责数据的计算，然后将计算好的数据提交给`GPU`，`GPU`会将这些计算好的数据进行`渲染`，`GPU`将渲染好的数据存放在`帧缓存`里面；接着，`视频控制器`会去读取`帧缓存`里面的数据，然后将读取出来的数据完整地显示在`屏幕`上。这样就完成了数据的显示。

*注意：在iOS中是双缓冲机制，有前帧缓存、后帧缓存。* 

# 卡顿产生的原因及优化思路

* 我们看到的手机APP能够完成一些列的动态显示，其实是由`垂直同步信号（VSync）`和`水平同步信号（HSync）`协作完成。`VSync`先发出，随后`HSync`发出，完成整屏画面的显示；紧接着，再次发出一个`VSync`，显示下一帧的画面。

* 一旦发生了一个`VSync`，`GPU`会立马将缓存里面的数据渲染到屏幕上，完成画面显示。如果`GPU`处理的时间过长，导致`VSync`已经发出，而`GPU`还没有处理完，这时只能显示上一帧的`VSync`，而当前帧的`VSync`就丢失了，俗称`掉帧`。那么原先`GPU`处理好的数据只能由下一帧`VSync`来显示，造成界面的卡顿。

* 卡顿优化的思路：**尽可能减少CPU、GPU的资源消耗。**按照60FPS的刷帧率，每隔16ms就会有一次`VSync`信号，我们只要保证在这个时间段内完成数据的计算，那么卡顿就不会发生。

# 卡顿优化 —— CPU

* 尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用`CALayer`取代`UIView`

* 不要频繁地调用`UIView`的相关属性，比如`frame`、`bounds`、`transform`等属性，尽量减少不必要的修改

* 尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性

* `Autolayout`会比直接设置`frame`消耗更多的`CPU`资源

* 图片的`size`最好刚好跟`UIImageView`的`size`保持一致

* 控制一下线程的最大`并发数量`

* 尽量把耗时的操作放到`子线程`，例如：
文本尺寸计算
```objc
[@"text" boundingRectWithSize:CGSizeMake(100, MAXFLOAT) 
options:NSStringDrawingUsesLineFragmentOrigin 
attributes:nil context:nil];
```
文本绘制：
```objc
    [@"text" drawWithRect:CGRectMake(0, 0, 100, 100) 
    options:NSStringDrawingUsesLineFragmentOrigin 
    attributes:nil context:nil];
```
图片处理（解码、绘制）:
```objc
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        // 获取CGImage
        CGImageRef cgImage = [UIImage imageNamed:@"timg"].CGImage;

        // alphaInfo
        CGImageAlphaInfo alphaInfo = CGImageGetAlphaInfo(cgImage) & kCGBitmapAlphaInfoMask;
        BOOL hasAlpha = NO;
        if (alphaInfo == kCGImageAlphaPremultipliedLast ||
            alphaInfo == kCGImageAlphaPremultipliedFirst ||
            alphaInfo == kCGImageAlphaLast ||
            alphaInfo == kCGImageAlphaFirst) {
            hasAlpha = YES;
        }

        // bitmapInfo
        CGBitmapInfo bitmapInfo = kCGBitmapByteOrder32Host;
        bitmapInfo |= hasAlpha ? kCGImageAlphaPremultipliedFirst : kCGImageAlphaNoneSkipFirst;

        // size
        size_t width = CGImageGetWidth(cgImage);
        size_t height = CGImageGetHeight(cgImage);

        // context
        CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, 0, CGColorSpaceCreateDeviceRGB(), bitmapInfo);

        // draw
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), cgImage);

        // get CGImage
        cgImage = CGBitmapContextCreateImage(context);

        // into UIImage
        UIImage *newImage = [UIImage imageWithCGImage:cgImage];

        // release
        CGContextRelease(context);
        CGImageRelease(cgImage);

        // back to the main thread
        dispatch_async(dispatch_get_main_queue(), ^{
            self.imageView.image = newImage;
        });
    });
```

# 卡顿优化 —— GPU

* 尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示

* GPU能处理的最大纹理尺寸是4096x4096，一旦超过这个尺寸，就会占用CPU资源进行处理，所以纹理尽量不要超过这个尺寸

* 尽量减少视图数量和层次

* 减少透明的视图（alpha\<1），不透明的就设置opaque为YES

* 尽量避免出现离屏渲染
那么，什么是离屏渲染呢？

# 离屏渲染
### 概念
在OpenGL中，GPU有2种渲染方式
* `On-Screen Rendering`：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
* `Off-Screen Rendering`：离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作

### 离屏渲染消耗性能的原因
* 需要创建新的缓冲区
* 离屏渲染的整个过程，需要多次切换上下文环境，先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕

### 哪些操作会触发离屏渲染？
* 光栅化：
```objc
layer.shouldRasterize = YES
```

* 遮罩：
```objc
layer.mask
```

* 圆角，同时设置以下两个变量的值：
```objc
layer.masksToBounds = YES;
layer.cornerRadius > 0;
```
**考虑通过`CoreGraphics`绘制裁剪圆角，或者叫美工提供圆角图片**

* 阴影：
```objc
layer.shadowXXX
```

**如果设置了`layer.shadowPath`就不会产生离屏渲染**


#  卡顿的检测
我们平时所说的“卡顿”主要是因为在主线程执行了比较耗时的操作，可以添加Observer到主线程RunLoop中，通过监听RunLoop状态切换的耗时，以达到监控卡顿的目的：
（监测从结束休眠处理source1，到开始休眠处理source0所用的时间）
