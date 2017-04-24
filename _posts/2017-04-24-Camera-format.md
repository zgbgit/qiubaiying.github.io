---
layout:     post
title:      Camera常见格式及转换
subtitle:   
date:       2017-04-24
author:     ZGB
header-img: assets/camera_format/title.jpg
catalog: true
tags:
    - Linux
---

接触camera有一段时间了，这里总结下camera的各种数据格式以及他们之间的转换关系。

### 常见的数据格式

- YUV：亮度+色差 组合表示方式
- RGB：红绿蓝 组合表示方式
- RAW RGB：sensor直接输出的原始数据格式，一个Byte表示一个像素
- JPEG：压缩格式

### 名词

| 名词          |释义     |
|----           |----|
|颜色编码方法   |	通常说的YUV、RGB都是。|
|颜色空间 	    | 颜色空间也称彩色模型(又称彩色空间或彩色系统）它的用途是在某些标准下用通常可接受的方式对彩色加以说明。|
|色调           |		色调指的是一幅画中画面色彩的总体倾向，是大的色彩效果。|
|饱和度         |			饱和度是指色彩的鲜艳程度，也称色彩的纯度。饱和度取决于该色中含色成分和消色成分（灰色）的比例。含色成分越大，饱和度越大；消色成分越大，饱和度越小。|
|色差   		|	色差简单来说就是颜色的差别，发生在以多色光为光源的情况下，单色光不产生色差。|

### YUV
YUV主要用于优化彩色视频信号的传输，使其向后相容老式黑白电视。与RGB视频信号传输相比，它最大的优点在于只需占用极少的频宽（RGB要求三个独立的视频信号同时传输）。其中“Y”表示明亮度（Luminance或Luma），也就是灰阶值；而“U”和“V” 表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。“亮度”是透过RGB输入信号来建立的，方法是将RGB信号的特定部分叠加到一起。“色度”则定义了颜色的两个方面─色调与饱和度，分别用Cr和Cb来表示。其中，Cr反映了RGB输入信号红色部分与RGB信号亮度值之间的差异。而Cb反映的是RGB输入信号蓝色部分与RGB信号亮度值之间的差异。
采用YUV色彩空间的重要性是它的亮度信号Y和色度信号U、V是分离的。如果只有Y信号分量而没有U、V分量，那么这样表示的图像就是黑白灰度图像。彩色电视采用YUV空间正是为了用亮度信号Y解决彩色电视机与黑白电视机的兼容问题，使黑白电视机也能接收彩色电视信号。

### YCbCr
YUV经过缩放和偏移的翻版。其中Y与YUV 中的Y含义一致,Cb,Cr 同样都指色彩，只是在表示方法上不同而已。在YUV 家族中，YCbCr 是在计算机系统中应用最多的成员，其应用领域很广泛，JPEG、MPEG均采用此格式。一般人们所讲的YUV大多是指YCbCr。YCbCr 有许多取样格式，如4∶4∶4,4∶2∶2,4∶1∶1 和4∶2∶0。

这里有张很直白的图来了解YCbCr 和RGB颜色空间。
![1](/assets/camera_format/CCD.png)

### YUV（YCbCr）采样格式
主要的采样格式有YCbCr 4:2:0、YCbCr 4:2:2、YCbCr 4:1:1和 YCbCr 4:4:4。其中YCbCr 4:1:1 比较常用，其含义为：每个点保存一个 8bit 的亮度值(也就是Y值), 每 2 x 2 个点保存一个 Cr和Cb值, 图像在肉眼中的感觉不会起太大的变化。所以, 原来用 RGB(R,G,B 都是 8bit unsigned) 模型, 每个点需要 8x3=24 bits， 而现在仅需要 8+(8/4)+(8/4)=12bits, 平均每个点占12bits。这样就把图像的数据压缩了一半。
上边仅给出了理论上的示例，在实际数据存储中是有可能是不同的，下面给出几种具体的存储形式：

（1） YUV 4:4:4
　　YUV三个信道的抽样率相同，因此在生成的图像里，每个象素的三个分量信息完整（每个分量通常8比特），经过8比特量化之后，未经压缩的每个像素占用3个字节。
  
　　下面的四个像素为: [Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3]
  
　　存放的码流为: Y0 U0 V0 Y1 U1 V1 Y2 U2 V2 Y3 U3 V3
  
（2） YUV 4:2:2

　　每个色差信道的抽样率是亮度信道的一半，所以水平方向的色度抽样率只是4:4:4的一半。对非压缩的8比特量化的图像来说，每个由两个水平方向相邻的像素组成的宏像素需要占用4字节内存(例如下面映射出的前两个像素点只需要Y0、Y1、U0、V1四个字节)。
  
　　下面的四个像素为: [Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3]
  
　　存放的码流为: Y0 U0 Y1 V1 Y2 U2 Y3 V3
  
　　映射出像素点为：[Y0 U0 V1] [Y1 U0 V1] [Y2 U2 V3] [Y3 U2 V3]
  
（3） YUV 4:1:1

　　4:1:1的色度抽样，是在水平方向上对色度进行4:1抽样。对于低端用户和消费类产品这仍然是可以接受的。对非压缩的8比特量化的视频来说，每个由4个水平方向相邻的像素组成的宏像素需要占用6字节内存
  
　　下面的四个像素为: [Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3]
  
　　存放的码流为: Y0 U0 Y1 Y2 V2 Y3
  
　　映射出像素点为：[Y0 U0 V2] [Y1 U0 V2] [Y2 U0 V2] [Y3 U0 V2]
  
（4）YUV4:2:0

　　4:2:0并不意味着只有Y,Cb而没有Cr分量。它指得是对每行扫描线来说，只有一种色度分量以2:1的抽样率存储。相邻的扫描行存储不同的色度分量，也就是说，如果一行是4:2:0的话，下一行就是4:0:2，再下一行是4:2:0...以此类推。对每个色度分量来说，水平方向和竖直方向的抽样率都是2:1，所以可以说色度的抽样率是4:1。对非压缩的8比特量化的视频来说，每个由2x2个2行2列相邻的像素组成的宏像素需要占用6字节内存。
  
下面八个像素为：

```
    [Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3]
　　[Y5 U5 V5] [Y6 U6 V6] [Y7U7 V7] [Y8 U8 V8]
```

存放的码流为：

```
        Y0 U0 Y1 Y2 U2 Y3
        Y5 V5 Y6 Y7 V7 Y8
```
  
映射出的像素点为：

```
    [Y0 U0 V5] [Y1 U0 V5] [Y2 U2 V7] [Y3 U2 V7]
    [Y5 U0 V5] [Y6 U0 V5] [Y7U2 V7] [Y8 U2 V7]
```

YUV存取格式
YUV格式通常有两大类：打包（packed）格式和平面（planar）格式。前者将YUV分量存放在同一个数组中，通常是几个相邻的像素组成一个宏像素（macro-pixel）；而后者使用三个数组分开存放YUV三个分量，就像是一个三维平面一样。（注意：在介绍各种具体格式时，YUV各分量都会带有下标，如Y0、U0、V0表示第一个像素的YUV分量，Y1、U1、V1表示第二个像素的YUV分量，以此类推。）
- YUY2（和YUYV）格式为每个像素保留Y分量，而UV分量在水平方向上每两个像素采样一次。一个宏像素为4个字节，实际表示2个像素。（4:2:2的意思实际上是一个宏像素中有2个Y分量、1个U分量和1个V分量。）图像数据中YUV分量排列顺序如下：
> Y0 U0 Y1 V0 Y2 U2 Y3 V2 …
- YVYU格式跟YUY2类似，只是图像数据中YUV分量的排列顺序有所不同：
> Y0 V0 Y1 U0 Y2 V2 Y3 U2 …
- UYVY格式跟YUY2类似，只是图像数据中YUV分量的排列顺序有所不同：
> U0 Y0 V0 Y1 U2 Y2 V2 Y3 …
- AYUV格式带有一个Alpha通道，并且为每个像素都提取YUV分量，图像数据格式如下：
> A0 Y0 U0 V0 A1 Y1 U1 V1 …
- Y41P（和Y411）格式为每个像素保留Y分量，而UV分量在水平方向上每4个像素采样一次。一个宏像素为12个字节，实际表示8个像素。图像数据中YUV分量排列顺序如下：
> U0 Y0 V0 Y1 U4 Y2 V4 Y3 Y4 Y5 Y6 Y8 …
- Y211格式在水平方向上Y分量每2个像素采样一次，而UV分量每4个像素采样一次。一个宏像素为4个字节，实际表示4个像素。图像数据中YUV分量排列顺序如下：
> Y0 U0 Y2 V0 Y4 U4 Y6 V4 …
- YVU9格式为每个像素都提取Y分量，而在UV分量的提取时，首先将图像分成若干个4 x 4的宏块，然后每个宏块提取一个U分量和一个V分量。图像数据存储时，首先是整幅图像的Y分量数组，然后就跟着U分量数组，以及V分量数组。IF09格式与YVU9类似。
- IYUV格式为每个像素都提取Y分量，而在UV分量的提取时，首先将图像分成若干个2 x 2的宏块，然后每个宏块提取一个U分量和一个V分量。YV12格式与IYUV类似。
- YUV411、YUV420格式多见于DV数据中，前者用于NTSC制，后者用于PAL制。YUV411为每个像素都提取Y分量，而UV分量在水平方向上每4个像素采样一次。YUV420并非V分量采样为0，而是跟YUV411相比，在水平方向上提高一倍色差采样频率，在垂直方向上以U/V间隔的方式减小一半色差采样，如上图所示。

### Packed YUV Formats

| Label | FOURCC in Hex| Bits per pixel | Description｜
|:----:|:----:|:-----:|:-----:|
|AYUV|0x56555941      |32      |Combined YUV and alpha|
|CLJR|0x524A4C43      |8       |Cirrus Logic format with 4 pixels packed into a u_int32. A form of YUV 4:1:1 wiht less than 8 bits per Y, U and V sample.|
|CYUV|0x76757963|16|Essentially a copy of UYVY except that the sense of the height is reversed - the image is upside down with respect to the UYVY version.|
|GREY(Y800)|0x59455247|8|Apparently a duplicate of Y800 (and also, presumably, "Y8 ")|
|HDYC(UYVY)|0x43594448|16|YUV 4:2:2 (Y sample at every pixel, U and V sampled at every second pixel horizontally on each line). A macropixel contains 2 pixels in 1 u_int32. This is a suplicate of UYVY except that the color components use the BT709 color space (as used in HD video).|
|IRAW|0x57615349|?|Intel uncompressed YUV. I have no information on this format - can you help?|
|IUYV(UYVY)|0x56595549|16|Interlaced version of UYVY (line order 0, 2, 4,....,1, 3, 5....) registered by Silviu Brinzei of LEAD Technologies.|
|IY41|0x31555949|12|Interlaced version of Y41P (line order 0, 2, 4,....,1, 3, 5....) registered by Silviu Brinzei of LEAD Technologies.|
|IYU1|0x31555949|12|12 bit format used in mode 2 of the IEEE 1394 Digital Camera 1.04 spec. This is equivalent to Y411|
|IYU2|0x32555949|24|24 bit format used in mode 0 of the IEEE 1394 Digital Camera 1.04 spec|
|UYNV(UYVY)|0x564E5955|16|A direct copy of UYVY registered by NVidia to work around problems in some old codecs which did not like hardware which offered more than 2 UYVY surfaces.|
|UYVP|0x50565955|24?|YCbCr 4:2:2 extended precision 10-bits per component in U0Y0V0Y1 order. Registered by Rich Ehlers of Evans & Sutherland.|
|UYVY|0x59565955|16|YUV 4:2:2 (Y sample at every pixel, U and V sampled at every second pixel horizontally on each line). A macropixel contains 2 pixels in 1 u_int32.|
|V210|0x30313256|32|10-bit 4:2:2 YCrCb equivalent to the Quicktime format of the same name.|
|V422|0x32323456|16|I am told that this is an upside down version of UYVY.|
|V655|0x35353656|16?|16 bit YUV 4:2:2 format registered by Vitec Multimedia. I have no information on the component ordering or packing.|
|VYUY|0x59555956|?|ATI Packed YUV Data (format unknown)|
|Y16|0x20363159|16|16-bit uncompressed greyscale image.|
|Y211|0x31313259|8|Packed YUV format with Y sampled at every second pixel across each line and U and V sampled at every fourth pixel.|
|Y411|0x31313459|12|YUV 4:1:1 with a packed, 6 byte/4 pixel macroblock structure.|
|Y41P|0x50313459|12|YUV 4:1:1 (Y sample at every pixel, U and V sampled at every fourth pixel horizontally on each line). A macropixel contains 8 pixels in 3 u_int32s.|
|Y41T|0x54313459|12|Format as for Y41P but the lsb of each Y component is used to signal pixel transparency .|
|Y422(UYVY)|0x32323459|16|Direct copy of UYVY as used by ADS Technologies Pyro WebCam firewire camera.|
|Y42T|0x54323459|16|Format as for UYVY but the lsb of each Y component is used to signal pixel transparency .|
|Y8(Y800)|0x20203859|8|Duplicate of Y800 as far as I can see.|
|Y800|0x30303859|8|Simple, single Y plane for monochrome images.|
|YUNV(YUY2)|0x564E5559|16|A direct copy of YUY2 registered by NVidia to work around problems in some old codecs which did not like hardware which offered more than 2 YUY2 surfaces.|
|YUVP|0x50565559|24?|YCbCr 4:2:2 extended precision 10-bits per component in Y0U0Y1V0 order. Registered by Rich Ehlers of Evans & Sutherland.|
|YUY2|0x32595559|16|YUV 4:2:2 as for UYVY but with different component ordering within the u_int32 macropixel.|
|YUYV (YUY2)|0x56595559|16|Duplicate of YUY2|
|YVYU|0x55595659|16|YUV 4:2:2 as for UYVY but with different component ordering within the u_int32 macropixel.|

### Planar YUV Formats

| Label | FOURCC in Hex| Bits per pixel | Description |
|:-------:|:--------------:|:-------------------:|:-----------------:|
|CLPL|0x4C504C43|12|Format similar to YV12 but including a level of indirection.|
|CXY1|0x31595843|12|Planar YUV 4:1:1 format registered by Conexant.|
|CXY2|0x32595842|16|Planar YUV 4:2:2 format registered by Conexant.|
|I420|0x30323449|12|8 bit Y plane followed by 8 bit 2x2 subsampled U and V planes.|
|IF09|0x39304649|9.5|As YVU9 but an additional 4x4 subsampled plane is appended containing delta information relative to the last frame. (Bpp is reported as 9)|
|IMC1|0x31434D49|12|As YV12 except the U and V planes each have the same stride as the Y plane|
|IMC2|0x32434D49|12|Similar to IMC1 except that the U and V lines are interleaved at half stride boundaries|
|IMC3(IMC1)|0x33434D49|12|As IMC1 except that U and V are swapped|
|IMC4(IMC2)|0x34434D49|12|As IMC2 except that U and V are swapped|
|IYUV(I420)|0x56555949|12|Duplicate FOURCC, identical to I420.|
|NV12|0x3231564E|12|8-bit Y plane followed by an interleaved U/V plane with 2x2 subsampling|
|NV21|0x3132564E|12|As NV12 with U and V reversed in the interleaved plane|
|Y41B|0x42313459|12?|Weitek format listed as "YUV 4:1:1 planar". I have no other information on this format.|
|Y42B|0x42323459|16?|Weitek format listed as "YUV 4:2:2 planar". I have no other information on this format.|
|Y8(Y800)|0x20203859|8|Duplicate of Y800 as far as I can see.|
|Y800|0x30303859|8|Simple, single Y plane for monochrome images.|
|YUV9|0x39565559|9?|Registered by Intel., this is the format used internally by Indeo video code|
|YV12|0x32315659|12|8 bit Y plane followed by 8 bit 2x2 subsampled V and U planes.|
|YV16|0x36315659|16|8 bit Y plane followed by 8 bit 2x1 subsampled V and U planes.|
|YVU9|0x39555659|9|8 bit Y plane followed by 8 bit 4x4 subsampled V and U planes. Registered by Intel.|

### RGB
计算机彩色显示器显示色彩的原理与彩色电视机一样，都是采用R（Red）、G（Green）、B（Blue）相加混色的原理：通过发射出三种不同强度的电子束，使屏幕内侧覆盖的红、绿、蓝磷光材料发光而产生色彩。这种色彩的表示方法称为RGB色彩空间表示（它也是多媒体计算机技术中用得最 多的一种色彩空间表示方法）。根据色度学的介绍，不同波长的单色光会引起不同的彩色感觉，但相同的彩色感觉却可以来源于不同的光谱成分组合。自然界中几乎所有的颜色都能用三种基本彩色混合配出，在彩色电视技术中选择红色、绿色、和蓝色作为三基色。其他的颜色都可以用红色、绿色和蓝色按照不同的比例混合而成。所选取的红色、绿色和蓝色三基色空间。简称为RGB颜色空间。
- RGB565    每个像素用16位表示，RGB分量分别使用5位、6位、5位
- RGB555    每个像素用16位表示，RGB分量都使用5位（剩下1位不用）
- RGB24    每个像素用24位表示，RGB分量各使用8位
- RGB32    每个像素用32位表示，RGB分量各使用8位（剩下8位不用）
- ARGB32    每个像素用32位表示，RGB分量各使用8位（剩下的8位用于表示Alpha通道值）

### RGB565

使用16位表示一个像素，这16位中的5位用于R，6位用于G，5位用于B。

程序中通常使用一个字（WORD，一个字等于两个字节）来操作一个像素。当读出一个像素后，这个字的各个位意义如下：

```
 高字节              低字节 
 R R R R R G G G     G G G B B B B B 
```

可以组合使用屏蔽字和移位操作来得到RGB各分量的值： 
```
#define RGB565_MASK_RED    0xF800 
#define RGB565_MASK_GREEN  0x07E0 
#define RGB565_MASK_BLUE   0x001F 
R = (wPixel & RGB565_MASK_RED) >> 11;   // 取值范围0-31 
G = (wPixel & RGB565_MASK_GREEN) >> 5;  // 取值范围0-63 
B =  wPixel & RGB565_MASK_BLUE;         // 取值范围0-31
#define RGB(r,g,b) (unsigned int)( (r|0x08 << 11) | (g|0x08 << 6) | b|0x08 )
#define RGB(r,g,b) (unsigned int)( (r|0x08 << 10) | (g|0x08 << 5) | b|0x08 )
```

该代码可以解决24位与16位相互转换的问题

### RGB555
是另一种16位的RGB格式，RGB分量都用5位表示（剩下的1位不用）。

使用一个字读出一个像素后，这个字的各个位意义如下： 
```
  高字节             低字节 
 X R R R R G G       G G G B B B B B       （X表示不用，可以忽略）  
```
RGB24使用24位来表示一个像素，RGB分量都用8位表示，取值范围为0-255 

RGB32使用32位来表示一个像素，RGB分量各用去8位，剩下的8位不用

### RGB24
RGB24使用24位来表示一个像素，RGB分量都用8位表示，取值范围为0-255。注意在内存中RGB各分量的排列顺序为：BGR BGR BGR…。通常可以使用RGBTRIPLE数据结构来操作一个像素，它的定义为：
```
typedef struct tagRGBTRIPLE {
BYTE rgbtBlue;		 // 蓝色分量
BYTE rgbtGreen;	 // 绿色分量
BYTE rgbtRed; 		// 红色分量
} RGBTRIPLE;
```

### RGB32
RGB32使用32位来表示一个像素，RGB分量各用去8位，剩下的8位用作Alpha通道或者不用。（ARGB32就是带Alpha通道的RGB24。）注意在内存中RGB各分量的排列顺序为：BGRA BGRA BGRA…。通常可以使用RGBQUAD数据结构来操作一个像素，它的定义为：
```
typedef struct tagRGBQUAD {
BYTE rgbBlue; 		// 蓝色分量
BYTE rgbGreen; 	// 绿色分量
BYTE rgbRed; 		// 红色分量
BYTE rgbReserved; 	// 保留字节（用作Alpha通道或忽略）
} RGBQUAD。
```

### RAW
RAW格式常用的有RAW8和RAW10。即每个像素用8bit或者10bit表示。

![RAW10](/assets/camera_format/RAW10.png)

可以看出，对于RAW10，需要把10bit的数据转换为8bit的数据，需要进行时钟域的转换，RAW10字节打包后的时钟频率 = 打包前的1.25倍。

### 格式转换

网上查到的RGB颜色空间到YUV颜色空间的转换公式:
```
    Y= 0.256788*R + 0.504129*G + 0.097906*B +  16;
    U=-0.148223*R - 0.290993*G + 0.439216*B + 128;
    V= 0.439216*R - 0.367788*G - 0.071427*B + 128;
```
YUV颜色空间到RGB颜色空间的转换公式:
```
    B= 1.164383 * (Y - 16) + 2.017232*(U - 128);
    G= 1.164383 * (Y - 16) - 0.391762*(U - 128) - 0.812968*(V - 128);
    R= 1.164383 * (Y - 16) + 1.596027*(V - 128);
```

或者：
```
    Y =  0.299*R + 0.587*G + 0.114*B;
    U = -0.147*R - 0.289*G + 0.436*B;
    V =  0.615*R - 0.515*G - 0.100*B;
    R = Y + 1.14*V;
    G = Y - 0.39*U - 0.58*V;
    B = Y + 2.03*U;
```

对于第一个公式，先把它转换为整形进行运算，浮点计算需要消耗太多的CPU资源。
```
B  = 1.164383 * (Y - 16) + 2.017232*(U - 128)
    = ( 1.164383 * 2^8 * (Y - 16) ) >> 8 + ( 2.017232 * 2^8 * (U - 128) ) >> 8
    =  ( 298 *  (Y - 16) ) >> 8 + ( 516 * (U - 128) ) >> 8

G  = 1.164383 * (Y - 16) - 0.391762*(U - 128) - 0.812968*(V - 128);
    = ( 298 *  (Y - 16) ) >> 8 - ( 100 * ( U - 128 ) ) >> 8 - ( 208 * ( V -128 ) ) >> 8

R  = 1.164383 * (Y - 16) + 1.596027*(V - 128)
    = ( 298 *  (Y - 16) ) >> 8 + ( 407 * (V - 128) ) >> 8
```

将计算的过程整理到代码中，发现效果并不理想，难道计算的过程出错了吗？

### ITU BT.601
YUV的标准不只有一种，所以分别会对应不同的转换公式。常用的标准有ITU-R BT.601，ITU-R BT.709，ITU-R BT.2020，JPEG。其中，ITU BT.601对应于标准分辨率电视（SDTV）、ITU BT.709对应于高分辨率电视（HDTV）。
对于BT.601和BT.709，还分为了Full Range和TV Range，所以对于这两种常用的标准来说，共有4种对应的公式组合。

几个常见的颜色转换矩阵（YUV-RGB）：

ITU BT.601 Full Range转换矩阵

![](/assets/camera_format/1.png)

ITU BT.601 Full Range转换矩阵

![](/assets/camera_format/2.png)

ITU BT.709默认转换矩阵

![](/assets/camera_format/3.png)

这几个转换矩阵的得来可以去看看标准的文档，这里截取ITU BT.601文档中的部分说明。

![](/assets/camera_format/4.png)

![](/assets/camera_format/5.png)

![](/assets/camera_format/6.png)

![](/assets/camera_format/7.png)

简单解释一下，上面三个公式是真正每个分量（Y/Cb/Cr)与颜色空间RGB的转换关系，下面三个公式表示实际存取时Y、Cr、Cb与YUV颜色空间的转换关系。这里需要说明的是，公式对YUV的值进行了限定，去除了对人眼影响不大的YUV分量（即上面所说的TV Range），所以对于Full Range，也就是Y、Cr、Cb的取值范围都是[0...255]的时候，下面的三个公式需要进行修改，即
```
Y = int{(256E'y)*D}/D
Cr =  int{(256E'cr+128)*D}/D
Cb =  int{(256E'cb+128)*D}/D
```

由此可以得出ITU BT.601 Full Range的转换公式为。
```
R = Y + 1.402 * (V-128)
G = Y - 0.3455 * (U-128) - 0.7169*(V-128)
B = Y + 1.772 * (U-128)
```
这里的出的结论和上面的矩阵有些许差异，但基本上是一致的，没有太大偏差。

### UYVY转RGB32代码
根据上面推导的公式，代码如下

<code class="hljs livecodeserver">{% highlight ruby %}
unsigned char clip(int data)
{
	if (data > 255)
	{
		return 255;
	}
	if (data < 0)
	{
		return 0;
	}
	
	return (unsigned char)data;
}

static int UYVY422ToRGB32(const unsigned char *src_buffer,  const unsigned char *des_buffer,int w, int h)
{
    unsigned char *yuv, *rgb;
    unsigned char u, v, y1, y2;
	int r, g, b;
	int size,i;

    yuv = src_buffer;
    rgb = des_buffer;

    if (yuv == NULL || rgb == NULL)
    {
        printf ("error: input data null!\n");
        return -1;
    }

    size = w * h;

    for( i = 0; i < size; i += 2)
    {
		y1 = yuv[2*i + 1] & 0xff;
		y2 = yuv[2*i + 3] & 0xff;
		u = yuv[2*i] & 0xff;
		v = yuv[2*i + 2] & 0xff;
	
		b = (y1 + (u - 128) + ((104*(u - 128))>>8));
		g = (y1 - (89*(v - 128)>>8) - ((183*(u - 128))>>8));
		r = (y1 + (v - 128) + ((199*(v - 128))>>8));
	
		rgb[4*i]     = clip(b);
		rgb[4*i + 1] = clip(g);
		rgb[4*i + 2] = clip(r);
		rgb[4*i + 3] = 0x00;
		
		b = (y2 + (u - 128) + ((104*(u - 128))>>8));
		g = (y2 - (89*(v - 128)>>8) - ((183*(u - 128))>>8));
		r = (y2 + (v - 128) + ((199*(v - 128))>>8));
	
		rgb[4*i + 4] = clip(b);
		rgb[4*i + 5] = clip(g);
		rgb[4*i + 6] = clip(r);
		rgb[4*i + 7] = 0x00;
    }

    return 0;
}
{% endhighlight %}</code>

参考链接：https://en.wikipedia.org/wiki/YCbCr
