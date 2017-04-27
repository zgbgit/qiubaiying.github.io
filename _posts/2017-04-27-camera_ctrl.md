---
layout:     post
title:      V4L2 controls
subtitle:   
date:       2017-04-27
author:     ZGB
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - Camera
---

“[V4L2 ioctl函数](http://zhenggaobin.top/2017/04/27/v4l2-ioctl/)”中列出了V4L2的一系列ioctl函数，作为提供给应用程序的接口。虽然这些ioctl函数已经十分丰富，能够完成大部分的功能，但是对于视频系统来说，肯定有些功能是ioctl函数没有涉及的，比如亮度、对比度等调节。那么这个时候就需要control来实现我们自定义的功能。


### 什么是V4L2 control
如上所说，我们在使用v4l2设备时，比如说camera（本文就是讲的camera），需要调节其中的参数，这时就需要ctrl。
```c
Videodev2.h (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\include\uapi\linux)	77514	2016/8/15

/*
 *	C O N T R O L S
 */
struct v4l2_control {
	__u32		     id;
	__s32		     value;
};
```
可以看出，每个ctrl有一个32位的ID，也就是说通过这个ID来区分不同的ctrl。每个ctrl又对应一个32位的value。那么整个流程自然就是应用层通过ID找到对应的ctrl，修改其中的value，达到修改参数的效果。

### 调用流程

```c
V4l2-ioctl.c (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\drivers\media\v4l2-core)

static struct v4l2_ioctl_info v4l2_ioctls[] = {
	......
	IOCTL_INFO_FNC(VIDIOC_G_CTRL, v4l_g_ctrl, v4l_print_control, INFO_FL_CTRL | INFO_FL_CLEAR(v4l2_control, id)),
	IOCTL_INFO_FNC(VIDIOC_S_CTRL, v4l_s_ctrl, v4l_print_control, INFO_FL_PRIO | INFO_FL_CTRL),
	......
	};
```
这里定义了一张ioctl的系统调用函数表。注意这里有两个特殊的函数VIDIOC_G_CTRL和VIDIOC_S_CTRL。如果想要设置某个参数值，就要调用VIDIOC_S_CTRL。

```c
V4l2-ioctl.c (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\drivers\media\v4l2-core)	80520	2016/8/15

static int v4l_s_ctrl(const struct v4l2_ioctl_ops *ops,
				struct file *file, void *fh, void *arg)
{
	struct video_device *vfd = video_devdata(file);
	struct v4l2_control *p = arg;
	struct v4l2_fh *vfh =
		test_bit(V4L2_FL_USES_V4L2_FH, &vfd->flags) ? fh : NULL;
	struct v4l2_ext_controls ctrls;
	struct v4l2_ext_control ctrl;

	if (vfh && vfh->ctrl_handler)
		return v4l2_s_ctrl(vfh, vfh->ctrl_handler, p);
	if (vfd->ctrl_handler)
		return v4l2_s_ctrl(NULL, vfd->ctrl_handler, p);
	if (ops->vidioc_s_ctrl)
		return ops->vidioc_s_ctrl(file, fh, p);
	if (ops->vidioc_s_ext_ctrls == NULL)
		return -ENOTTY;

	ctrls.ctrl_class = V4L2_CTRL_ID2CLASS(p->id);
	ctrls.count = 1;
	ctrls.controls = &ctrl;
	ctrl.id = p->id;
	ctrl.value = p->value;
	if (check_ext_ctrls(&ctrls, 1))
		return ops->vidioc_s_ext_ctrls(file, fh, &ctrls);
	return -EINVAL;
}
```
vfh是v4l2子设备文件的句柄，通过它能轻易的找到v4l2_ctrl_handler。v4l2_ctrl_handler管理设备的ctrls。这里会通过vfh直接调用`v4l2_s_ctrl(vfh, vfh->ctrl_handler, p);`。

```c
V4l2-ctrls.c (x:\zgb\sbcc_wlq\sbcc-ph8800-wlq_linux\drivers\media\v4l2-core)	102529	2016/8/15

int v4l2_s_ctrl(struct v4l2_fh *fh, struct v4l2_ctrl_handler *hdl,
					struct v4l2_control *control)
{
	struct v4l2_ctrl *ctrl = v4l2_ctrl_find(hdl, control->id);
	struct v4l2_ext_control c = { control->id };
	int ret;

	if (ctrl == NULL || !ctrl->is_int)
		return -EINVAL;

	if (ctrl->flags & V4L2_CTRL_FLAG_READ_ONLY)
		return -EACCES;

	c.value = control->value;
	ret = set_ctrl_lock(fh, ctrl, &c);
	control->value = c.value;
	return ret;
}
```
这里首先通过v4l2_ctrl_find函数，利用ID找到对应的ctrl。接着执行set_ctrl_lock函数设置具
