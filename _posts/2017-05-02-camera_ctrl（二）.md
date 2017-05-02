---
layout:     post
title:      V4L2 controls（二）
subtitle:   
date:       2017-05-02
author:     ZGB
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Camera
---

在“[V4L2 controls](http://zhenggaobin.top/2017/04/27/camera_ctrl/)”中，讲述了怎么添加一个ctrl。但是实际做的过程中我们会发现一个新的问题，v4l2_control的ID是怎么来的，能随便定义吗？要自定义增加一个怎么办？这里总结下新增一个ID的方法。

### driver修改

（1）新增一个ID

所有的ID都在`V4l2-controls.h (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\include\uapi\linux)	43767	2016/8/15`中定义，可以看出ctrl分为了很多的class，我们最常用的是user controls。
```c
/* Control classes */
#define V4L2_CTRL_CLASS_USER		0x00980000	/* Old-style 'user' controls */
#define V4L2_CTRL_CLASS_MPEG		0x00990000	/* MPEG-compression controls */
#define V4L2_CTRL_CLASS_CAMERA		0x009a0000	/* Camera class controls */
#define V4L2_CTRL_CLASS_FM_TX		0x009b0000	/* FM Modulator controls */
#define V4L2_CTRL_CLASS_FLASH		0x009c0000	/* Camera flash controls */
#define V4L2_CTRL_CLASS_JPEG		0x009d0000	/* JPEG-compression controls */
#define V4L2_CTRL_CLASS_IMAGE_SOURCE	0x009e0000	/* Image source controls */
#define V4L2_CTRL_CLASS_IMAGE_PROC	0x009f0000	/* Image processing controls */
#define V4L2_CTRL_CLASS_DV		0x00a00000	/* Digital Video controls */
#define V4L2_CTRL_CLASS_FM_RX		0x00a10000	/* FM Receiver controls */
#define V4L2_CTRL_CLASS_RF_TUNER	0x00a20000	/* RF tuner controls */
#define V4L2_CTRL_CLASS_DETECT		0x00a30000	/* Detection controls */
```
user controls中默认定义了44个ctrl。
```c
/* User-class control IDs */

#define V4L2_CID_BASE			(V4L2_CTRL_CLASS_USER | 0x900)
#define V4L2_CID_USER_BASE 		V4L2_CID_BASE
#define V4L2_CID_USER_CLASS 		(V4L2_CTRL_CLASS_USER | 1)
#define V4L2_CID_BRIGHTNESS		(V4L2_CID_BASE+0)
#define V4L2_CID_CONTRAST		(V4L2_CID_BASE+1)
#define V4L2_CID_SATURATION		(V4L2_CID_BASE+2)
#define V4L2_CID_HUE			(V4L2_CID_BASE+3)
#define V4L2_CID_AUDIO_VOLUME		(V4L2_CID_BASE+5)
#define V4L2_CID_AUDIO_BALANCE		(V4L2_CID_BASE+6)
#define V4L2_CID_AUDIO_BASS		(V4L2_CID_BASE+7)
#define V4L2_CID_AUDIO_TREBLE		(V4L2_CID_BASE+8)
#define V4L2_CID_AUDIO_MUTE		(V4L2_CID_BASE+9)
#define V4L2_CID_AUDIO_LOUDNESS		(V4L2_CID_BASE+10)
#define V4L2_CID_BLACK_LEVEL		(V4L2_CID_BASE+11) /* Deprecated */
#define V4L2_CID_AUTO_WHITE_BALANCE	(V4L2_CID_BASE+12)
#define V4L2_CID_DO_WHITE_BALANCE	(V4L2_CID_BASE+13)
#define V4L2_CID_RED_BALANCE		(V4L2_CID_BASE+14)
#define V4L2_CID_BLUE_BALANCE		(V4L2_CID_BASE+15)
#define V4L2_CID_GAMMA			(V4L2_CID_BASE+16)
#define V4L2_CID_WHITENESS		(V4L2_CID_GAMMA) /* Deprecated */
#define V4L2_CID_EXPOSURE		(V4L2_CID_BASE+17)
#define V4L2_CID_AUTOGAIN		(V4L2_CID_BASE+18)
#define V4L2_CID_GAIN			(V4L2_CID_BASE+19)
#define V4L2_CID_HFLIP			(V4L2_CID_BASE+20)
#define V4L2_CID_VFLIP			(V4L2_CID_BASE+21)

#define V4L2_CID_POWER_LINE_FREQUENCY	(V4L2_CID_BASE+24)
enum v4l2_power_line_frequency {
	V4L2_CID_POWER_LINE_FREQUENCY_DISABLED	= 0,
	V4L2_CID_POWER_LINE_FREQUENCY_50HZ	= 1,
	V4L2_CID_POWER_LINE_FREQUENCY_60HZ	= 2,
	V4L2_CID_POWER_LINE_FREQUENCY_AUTO	= 3,
};
#define V4L2_CID_HUE_AUTO			(V4L2_CID_BASE+25)
#define V4L2_CID_WHITE_BALANCE_TEMPERATURE	(V4L2_CID_BASE+26)
#define V4L2_CID_SHARPNESS			(V4L2_CID_BASE+27)
#define V4L2_CID_BACKLIGHT_COMPENSATION 	(V4L2_CID_BASE+28)
#define V4L2_CID_CHROMA_AGC                     (V4L2_CID_BASE+29)
#define V4L2_CID_COLOR_KILLER                   (V4L2_CID_BASE+30)
#define V4L2_CID_COLORFX			(V4L2_CID_BASE+31)
enum v4l2_colorfx {
	V4L2_COLORFX_NONE			= 0,
	V4L2_COLORFX_BW				= 1,
	V4L2_COLORFX_SEPIA			= 2,
	V4L2_COLORFX_NEGATIVE			= 3,
	V4L2_COLORFX_EMBOSS			= 4,
	V4L2_COLORFX_SKETCH			= 5,
	V4L2_COLORFX_SKY_BLUE			= 6,
	V4L2_COLORFX_GRASS_GREEN		= 7,
	V4L2_COLORFX_SKIN_WHITEN		= 8,
	V4L2_COLORFX_VIVID			= 9,
	V4L2_COLORFX_AQUA			= 10,
	V4L2_COLORFX_ART_FREEZE			= 11,
	V4L2_COLORFX_SILHOUETTE			= 12,
	V4L2_COLORFX_SOLARIZATION		= 13,
	V4L2_COLORFX_ANTIQUE			= 14,
	V4L2_COLORFX_SET_CBCR			= 15,
};
#define V4L2_CID_AUTOBRIGHTNESS			(V4L2_CID_BASE+32)
#define V4L2_CID_BAND_STOP_FILTER		(V4L2_CID_BASE+33)

#define V4L2_CID_ROTATE				(V4L2_CID_BASE+34)
#define V4L2_CID_BG_COLOR			(V4L2_CID_BASE+35)

#define V4L2_CID_CHROMA_GAIN                    (V4L2_CID_BASE+36)

#define V4L2_CID_ILLUMINATORS_1			(V4L2_CID_BASE+37)
#define V4L2_CID_ILLUMINATORS_2			(V4L2_CID_BASE+38)

#define V4L2_CID_MIN_BUFFERS_FOR_CAPTURE	(V4L2_CID_BASE+39)
#define V4L2_CID_MIN_BUFFERS_FOR_OUTPUT		(V4L2_CID_BASE+40)

#define V4L2_CID_ALPHA_COMPONENT		(V4L2_CID_BASE+41)
#define V4L2_CID_COLORFX_CBCR			(V4L2_CID_BASE+42)

/* last CID + 1 */
#define V4L2_CID_LASTP1                         (V4L2_CID_BASE+43)
```
我们在这个文件的末尾加上一个
```c
#define V4L2_CID_SCENE_MODE_ZGB		    (V4L2_CID_BASE+44)
```

（2）设置这个ID对应的名字

在`V4l2-ctrls.c (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\drivers\media\v4l2-core)	102529	2016/8/15
`中，增加此ID对应的名字。
```c
/* Return the control name. */
const char *v4l2_ctrl_get_name(u32 id)
{
	switch (id) {
	/* USER controls */
	/* Keep the order of the 'case's the same as in v4l2-controls.h! */
	case V4L2_CID_USER_CLASS:		return "User Controls";
	case V4L2_CID_BRIGHTNESS:		return "Brightness";
	.....
	case V4L2_CID_SCENE_MODE_ZGB:		return "Scene mode change";

	/* Codec controls */
	/* The MPEG controls are applicable to all codec controls
	 * and the 'MPEG' part of the define is historical */
	 ......
}
```

（3）指定这个ID所对应的ctrl的属性
将其设为`V4L2_CTRL_FLAG_SLIDER`即可。
```c
void v4l2_ctrl_fill(u32 id, const char **name, enum v4l2_ctrl_type *type,
		    s64 *min, s64 *max, u64 *step, s64 *def, u32 *flags)
{
	*name = v4l2_ctrl_get_name(id);
	*flags = 0;
	
	......
	
        switch (id) {
        case V4L2_CID_MPEG_AUDIO_ENCODING:
	case V4L2_CID_MPEG_AUDIO_MODE:
	......
	case V4L2_CID_SCENE_MODE_ZGB:
		*flags |= V4L2_CTRL_FLAG_SLIDER;
		break;
	}
	
	......
}
```

（4）添加ctrl和ops

这和上一篇所述一致，参见“V4L2 controls”中 “（3）添加crtl”和“（4）ops函数”

### 应用层函数

应用层对应的调用函数。注意这里不能继续使用V4L2_CID_SCENE_MODE_ZGB宏，因为很可能你的编译器不认识，这是自己新加的嘛。
```c
struct v4l2_control control;
	
if ((fd_v4l = open_video_device()) < 0)
{
	printf("Unable to open v4l2 capture device.\n");
	return -1;
}

memset (&control, 0, sizeof (control));
control.id = V4L2_CID_BASE+44;
control.value = v4l2_scene_mode_val;
if( ioctl(fd_v4l, VIDIOC_S_CTRL , &control) < 0)
{
	printf("VIDIOC_S_CTRL failed\n");
	return -1;
}
```
