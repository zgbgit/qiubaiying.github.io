---
layout:     post
title:      V4L2 controls（三）
subtitle:   
date:       2017-05-24
author:     ZGB
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Camera
---

### V4L2 controls（三）

前面的两篇文章说明了怎么新建ctrl和自定义ctrl，我们主要用v4l2_ctrl_new_std函数来新增一个控制。但是有一种情况是，我们的控制量是从0开始的步进为1的整数，甚至有些时候控制量是不连续的，这个时候可以设置menu ctrl。

#### driver修改
（1）添加菜单数组
```c
static const char * const ov5640_test_pattern_menu[] = {
	"Disabled",
	"Vertical Color Bars",
};
```

（2）添加ctrl
```c
v4l2_ctrl_new_std_menu_items(&ov5640->ctrl_handler, &ov5640_ctrl_ops,
				     V4L2_CID_TEST_PATTERN,
				     ARRAY_SIZE(ov5640_test_pattern_menu) - 1,
				     0, 0, ov5640_test_pattern_menu);
```

#### 应用程序
```c
memset (&control, 0, sizeof (control));
control.id = V4L2_CID_TEST_PATTERN;
control.value = 1;
if( ioctl(fd_v4l, VIDIOC_S_CTRL , &control) < 0)
{
    printf("TEST_PATTERN failed\n");
    return -1;
}
```

#### 各种添加ctrl方法总结
（1）v4l2_ctrl_new_std
```c
struct v4l2_ctrl *v4l2_ctrl_new_std(struct v4l2_ctrl_handler *hdl,
const struct v4l2_ctrl_ops *ops,u32 id, s64 min, s64 max, u64 step, s64 def)
```
这个函数能适应大多数的情况，函数会根据crtl id、最小值min、最大值max、每个value之间的步进step来填充ctrl。当前值会被设为默认值def。

（2）v4l2_ctrl_new_std_menu
```c
struct v4l2_ctrl *v4l2_ctrl_new_std_menu(struct v4l2_ctrl_handler *hdl,
			const struct v4l2_ctrl_ops *ops,
			u32 id, u8 _max, u64 mask, u8 _def)
```
这个函数用来填充一个menu ctrl。这个ctrl没有最小值，因为都是从0开始。mask用来代替步进step设置，如果mask的X位为1，那么菜单项X就会被跳过。

（3）v4l2_ctrl_new_int_menu
```c
struct v4l2_ctrl *v4l2_ctrl_new_int_menu(struct v4l2_ctrl_handler *hdl,
			const struct v4l2_ctrl_ops *ops,
			u32 id, u8 _max, u8 _def, const s64 *qmenu_int)
```
这个函数用来增加整数型菜单的控制变量，取消了mask设置，_def是菜单索引号，最后一个参数是菜单数组指针。这个函数适用于变量值是不连续的无规则的整数变量。

（4）v4l2_ctrl_new_std_menu_items
```c
struct v4l2_ctrl *v4l2_ctrl_new_std_menu_items(struct v4l2_ctrl_handler *hdl,
			const struct v4l2_ctrl_ops *ops, u32 id, u8 _max,
			u64 mask, u8 _def, const char * const *qmenu)
```
这个函数和v4l2_ctrl_new_std_menu相类似，不过加入了一个新的参数qmenu用来关联另一个菜单控制。