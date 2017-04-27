---
layout:     post
title:      V4L2 controls
subtitle:   
date:       2017-04-27
author:     ZGB
header-img: assets/camera_crtl/title.jpg
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
这里首先通过v4l2_ctrl_find函数，利用ID找到对应的ctrl。接着执行set_ctrl_lock函数设置具体的值。

```c
static int set_ctrl_lock(struct v4l2_fh *fh, struct v4l2_ctrl *ctrl,
			 struct v4l2_ext_control *c)
{
	int ret;

	v4l2_ctrl_lock(ctrl);
	user_to_new(c, ctrl);
	ret = set_ctrl(fh, ctrl, 0);
	if (!ret)
		cur_to_user(c, ctrl);
	v4l2_ctrl_unlock(ctrl);
	return ret;
}
```
先对ctrl上锁，接着利用user_to_new读取应用层传来的value值。接着利用set_ctrl对value值进行更新，然后对ctrl进行解锁。

```c
static int set_ctrl(struct v4l2_fh *fh, struct v4l2_ctrl *ctrl, u32 ch_flags)
{
	struct v4l2_ctrl *master = ctrl->cluster[0];
	int ret;
	int i;

	/* Reset the 'is_new' flags of the cluster */
	for (i = 0; i < master->ncontrols; i++)
		if (master->cluster[i])
			master->cluster[i]->is_new = 0;

	ret = validate_new(ctrl, ctrl->p_new);
	if (ret)
		return ret;

	/* For autoclusters with volatiles that are switched from auto to
	   manual mode we have to update the current volatile values since
	   those will become the initial manual values after such a switch. */
	if (master->is_auto && master->has_volatiles && ctrl == master &&
	    !is_cur_manual(master) && ctrl->val == master->manual_mode_value)
		update_from_auto_cluster(master);

	ctrl->is_new = 1;
	return try_or_set_cluster(fh, master, true, ch_flags);
}
```
这里做了一堆的判断，如果value值不是新的，就不更新value的值。最后调用try_or_set_cluster调用底层函数设置寄存器。

```c
static int try_or_set_cluster(struct v4l2_fh *fh, struct v4l2_ctrl *master,
			      bool set, u32 ch_flags)
{
	bool update_flag;
	int ret;
	int i;

	/* Go through the cluster and either validate the new value or
	   (if no new value was set), copy the current value to the new
	   value, ensuring a consistent view for the control ops when
	   called. */
	for (i = 0; i < master->ncontrols; i++) {
		struct v4l2_ctrl *ctrl = master->cluster[i];

		if (ctrl == NULL)
			continue;

		if (!ctrl->is_new) {
			cur_to_new(ctrl);
			continue;
		}
		/* Check again: it may have changed since the
		   previous check in try_or_set_ext_ctrls(). */
		if (set && (ctrl->flags & V4L2_CTRL_FLAG_GRABBED))
			return -EBUSY;
	}

	ret = call_op(master, try_ctrl);

	/* Don't set if there is no change */
	if (ret || !set || !cluster_changed(master))
		return ret;
	ret = call_op(master, s_ctrl);
	if (ret)
		return ret;

	/* If OK, then make the new values permanent. */
	update_flag = is_cur_manual(master) != is_new_manual(master);
	for (i = 0; i < master->ncontrols; i++)
		new_to_cur(fh, master->cluster[i], ch_flags |
			((update_flag && i > 0) ? V4L2_EVENT_CTRL_CH_FLAGS : 0));
	return 0;
}

```
通过call_op调用了try_ctrl和s_ctrl方法来设置寄存器。

### ov5640 driver
通过上面的描述，在驱动函数中，我们最重要的就是要注册ctrl的s_ctrl方法。下面给出具体的步骤。

（1）添加handler到驱动的设备结构体中
```c
struct ov5640 {
	......
	/* the ctrl_handler */
	struct v4l2_ctrl_handler ctrl_handler;
};
```

（2）在probe函数中初始化handler
```c
v4l2_ctrl_handler_init(&ov5640->ctrl_handler, 2);
```
第二个参数是提示该函数有多少个该handler的controls。它会根据这个信息分配一个hastable。它仅仅是一个提示。

（3）添加crtl
```c
v4l2_ctrl_new_std(&ov5640->ctrl_handler, &ov5640_ctrl_ops,
			V4L2_CID_BRIGHTNESS, 0, 255, 1, 0);
ov5640->sd.ctrl_handler = &ov5640->ctrl_handler;
```

（4）ops函数
```c
static int ov5640_s_ctrl(struct v4l2_ctrl *ctrl)
{
	struct v4l2_subdev *sd =
		&container_of(ctrl->handler, struct ov5640, ctrl_handler)->sd;
	struct i2c_client  *client = v4l2_get_subdevdata(sd);

	switch (ctrl->id) {
	case V4L2_CID_BRIGHTNESS:
		ov5640_write(client, 0x3a1e, ctrl->val - 2);
		return ov5640_write(client, 0x3a1b, ctrl->val);
	case V4L2_CID_CONTRAST:
		return ov5640_write(client, 0x5587, ctrl->val);
		default:
		break;
	}

	return -EINVAL;
}

static const struct v4l2_ctrl_ops ov5640_ctrl_ops = {
	.s_ctrl = ov5640_s_ctrl,
};

```

### 应用层函数
应用层对应的调用函数。
```c
struct v4l2_control control;
	
if ((fd_v4l = open_video_device()) < 0)
{
	printf("Unable to open v4l2 capture device.\n");
	return -1;
}

memset (&control, 0, sizeof (control));
control.id = V4L2_CID_BRIGHTNESS;
control.value = v4l2_brightness_val;
if( ioctl(fd_v4l, VIDIOC_S_CTRL , &control) < 0)
{
	printf("VIDIOC_S_CTRL failed\n");
	return -1;
}
```
