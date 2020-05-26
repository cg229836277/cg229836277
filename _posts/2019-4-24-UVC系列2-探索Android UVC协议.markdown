---
layout: post
title:  "UVC系列2-探索Android UVC协议"
date:   2019-4-24 18:19:25 +0800
categories: Android
---


> 微信公众号：Android部落格
>
> 文章选取android下linux-3.10作为分析对象，具体的UVC初始化过程可以参考csdn大神写的博客，地址是：http://blog.csdn.net/orz415678659。

## 1、UVC初始化设备
uvc加载摄像头的过程无非是初始化设备，加载设备，获取设备相关参数并加载相关参数到buffer，此时就已经将视频和控制参数加载到buffer了，这篇文章主要关注的是控制相关的参数。
需要关注的两个核心文件是：
- drivers\media\usb\uvc\uvc_ctrl.c
- drivers\media\usb\uvc\uvc_v4l2.c

首先看看uvc_ctrl.c文件中的`struct uvc_control_info uvc_ctrls[]`结构体，这个结构体中定义了摄像头的控制参数详情，主要包含了各种类型的控制，比如白平衡，曝光度等。其中需要重点关注的参数是上下左右和焦距的控制参数，如下：

```c
{
	.entity		= UVC_GUID_UVC_CAMERA,
	.selector	= UVC_CT_ZOOM_ABSOLUTE_CONTROL,
	.index		= 9,
	.size		= 2,
	.flags		= UVC_CTRL_FLAG_SET_CUR
			| UVC_CTRL_FLAG_GET_RANGE
			| UVC_CTRL_FLAG_RESTORE
			| UVC_CTRL_FLAG_AUTO_UPDATE,
},
{
	.entity		= UVC_GUID_UVC_CAMERA,
	.selector	= UVC_CT_ZOOM_RELATIVE_CONTROL,
	.index		= 10,
	.size		= 3,
	.flags		= UVC_CTRL_FLAG_SET_CUR
			| UVC_CTRL_FLAG_GET_MIN
			| UVC_CTRL_FLAG_GET_MAX 
			| UVC_CTRL_FLAG_GET_RES
			| UVC_CTRL_FLAG_GET_DEF
			| UVC_CTRL_FLAG_AUTO_UPDATE,
},
{
	.entity		= UVC_GUID_UVC_CAMERA,
	.selector	= UVC_CT_PANTILT_ABSOLUTE_CONTROL,
	.index		= 11,
	.size		= 8,
	.flags		= UVC_CTRL_FLAG_SET_CUR
			| UVC_CTRL_FLAG_GET_RANGE
			| UVC_CTRL_FLAG_RESTORE
			| UVC_CTRL_FLAG_AUTO_UPDATE,
},
{
	.entity		= UVC_GUID_UVC_CAMERA,
	.selector	= UVC_CT_PANTILT_RELATIVE_CONTROL,
	.index		= 12,
	.size		= 4,
	.flags		= UVC_CTRL_FLAG_SET_CUR
			| UVC_CTRL_FLAG_GET_MIN
			| UVC_CTRL_FLAG_GET_MAX 
			| UVC_CTRL_FLAG_GET_RES
			| UVC_CTRL_FLAG_GET_DEF
			| UVC_CTRL_FLAG_AUTO_UPDATE,
}
```
在这里可以看到这里定义了相对控制和绝对控制的参数，有焦距和上下左右，其中zoom是焦距调节，pan是左右，tilt是上下，缩写是ptz，所以云台摄像头也叫ptz摄像头。

### 1.1 初始化参数
另外需要重点关注的变量是`struct uvc_control_mapping uvc_ctrl_mappings[]`，这个结构体枚举出来了所有的控制类型，比如`id = V4L2_CID_PAN_ABSOLUTE`和`id = V4L2_CID_TILT_ABSOLUTE`的selector等于uvc_ctrls[]中的selector	是`UVC_CT_PANTILT_RELATIVE_CONTROL`参数，这两个id分别表示左右和上下的绝对控制，绝对控制的意思是给一个角度，摄像头转到指定的角度之后就停了。

焦距的参数与pan和tilt的参数类似。稍微有点不一样的地方是zoom在3.10版本中已经添加了相对和绝对控制，参数id分别为`V4L2_CID_ZOOM_ABSOLUTE，V4L2_CID_ZOOM_CONTINUOUS`，其中`V4L2_CID_ZOOM_CONTINUOUS`是相对控制，从名称也可以看出continuous是持续的意思，相对控制就是发给摄像头一个参数，摄像头会一直不停的转动，直到最大角度才停止，中途如果给发送了停止位，才会停止。

## 2、焦距的相对控制
在这么多控制参数中，重点看一下焦距的相对控制，如下:

```c
{
	.id		= V4L2_CID_ZOOM_CONTINUOUS,
	.name		= "Zoom, Continuous",
	.entity		= UVC_GUID_UVC_CAMERA,
	.selector	= UVC_CT_ZOOM_RELATIVE_CONTROL,
	.size		= 0,
	.offset		= 0,
	.v4l2_type	= V4L2_CTRL_TYPE_INTEGER,
	.data_type	= UVC_CTRL_DATA_TYPE_SIGNED,
	.get		= uvc_ctrl_get_zoom,
	.set		= uvc_ctrl_set_zoom,
}
```
看看这个参数中定义了get和set，其实这个是映射到的具体get和set方法，具体方法是：

```c
static __s32 uvc_ctrl_get_zoom(struct uvc_control_mapping *mapping,__u8 query, const __u8 *data)
{
	__s8 zoom = (__s8)data[0];
	switch (query) {
	case UVC_GET_CUR:
		return (zoom == 0) ? 0 : (zoom > 0 ? data[2] : -data[2]);
	case UVC_GET_MIN:
	case UVC_GET_MAX:
	case UVC_GET_RES:
	case UVC_GET_DEF:
	default:
		return data[2];
	}
}
static void uvc_ctrl_set_zoom(struct uvc_control_mapping *mapping,__s32 value, __u8 *data)
{
	data[0] = value == 0 ? 0 : (value > 0) ? 1 : 0xff;
	data[2] = min((int)abs(value), 0xff);
}
```
第一个函数是获取当前焦距所处的位置，在`UVC_GET_CUR`处，当前位置为0是，返回0，否则的话大于0，返回当前值，否则返回相反数。

第二个函数是设置zoom的值，data[0]填写的值有三种情况，0,1,0xff，其中0表示停止，1表示拉远，0xff表示拉近。

在这两个函数中可以看到`UVC_GET_MIN`和`UVC_GET_MAX`，意思是获取最大值和最小值，其实在设备初始化的时候，会将设备的这些各个参数的最大值和最小值读取并保存。

具体取值的地方在__uvc_query_v4l2_ctrl这个函数中，函数定义如下：

```c
static int __uvc_query_v4l2_ctrl(struct uvc_video_chain *chain,
	struct uvc_control *ctrl,
	struct uvc_control_mapping *mapping,
	struct v4l2_queryctrl *v4l2_ctrl)
{
	struct uvc_control_mapping *master_map = NULL;
	struct uvc_control *master_ctrl = NULL;
	struct uvc_menu_info *menu;
	unsigned int i;
	memset(v4l2_ctrl, 0, sizeof *v4l2_ctrl);
	v4l2_ctrl->id = mapping->id;
	v4l2_ctrl->type = mapping->v4l2_type;
	strlcpy(v4l2_ctrl->name, mapping->name, sizeof v4l2_ctrl->name);
	v4l2_ctrl->flags = 0;

	if (!(ctrl->info.flags & UVC_CTRL_FLAG_GET_CUR))
		v4l2_ctrl->flags |= V4L2_CTRL_FLAG_WRITE_ONLY;
	if (!(ctrl->info.flags & UVC_CTRL_FLAG_SET_CUR))
		v4l2_ctrl->flags |= V4L2_CTRL_FLAG_READ_ONLY;
	if (mapping->master_id)
		__uvc_find_control(ctrl->entity, mapping->master_id,&master_map, &master_ctrl, 0);
	if (master_ctrl && (master_ctrl->info.flags & UVC_CTRL_FLAG_GET_CUR)) {
		s32 val;
		int ret = __uvc_ctrl_get(chain, master_ctrl, master_map, &val);
		if (ret < 0)
			return ret;
		if (val != mapping->master_manual)
				v4l2_ctrl->flags |= V4L2_CTRL_FLAG_INACTIVE;
	}
	if (!ctrl->cached) {
		int ret = uvc_ctrl_populate_cache(chain, ctrl);
		if (ret < 0)
			return ret;
	}
	if (ctrl->info.flags & UVC_CTRL_FLAG_GET_DEF) {
		v4l2_ctrl->default_value = mapping->get(mapping, UVC_GET_DEF,uvc_ctrl_data(ctrl,UVC_CTRL_DATA_DEF));
	}
	switch (mapping->v4l2_type) {
	case V4L2_CTRL_TYPE_MENU:
		v4l2_ctrl->minimum = 0;
		v4l2_ctrl->maximum = mapping->menu_count - 1;
		v4l2_ctrl->step = 1;
		menu = mapping->menu_info;
		for (i = 0; i < mapping->menu_count; ++i, ++menu) {
			if (menu->value == v4l2_ctrl->default_value) {
				v4l2_ctrl->default_value = i;
				break;
			}
		}
		return 0;
	case V4L2_CTRL_TYPE_BOOLEAN:
		v4l2_ctrl->minimum = 0;
		v4l2_ctrl->maximum = 1;
		v4l2_ctrl->step = 1;
		return 0;
	case V4L2_CTRL_TYPE_BUTTON:
		v4l2_ctrl->minimum = 0;
		v4l2_ctrl->maximum = 0;
		v4l2_ctrl->step = 0;
		return 0;
	default:
		break;
	}
	if (ctrl->info.flags & UVC_CTRL_FLAG_GET_MIN)
		v4l2_ctrl->minimum = mapping->get(mapping, UVC_GET_MIN,uvc_ctrl_data(ctrl, UVC_CTRL_DATA_MIN));
	if (ctrl->info.flags & UVC_CTRL_FLAG_GET_MAX)
		v4l2_ctrl->maximum = mapping->get(mapping, UVC_GET_MAX,uvc_ctrl_data(ctrl, UVC_CTRL_DATA_MAX));
	if (ctrl->info.flags & UVC_CTRL_FLAG_GET_RES)
		v4l2_ctrl->step = mapping->get(mapping, UVC_GET_RES,uvc_ctrl_data(ctrl, UVC_CTRL_DATA_RES));
	return 0;
}
```
其中结构体```v4l2_queryctrl```的定义在\include\uapi\linux\videodev2.h文件中，

```c
struct v4l2_queryctrl {
	__u32		     id;
	__u32		     type;	/* enum v4l2_ctrl_type */
	__u8		     name[32];	/* Whatever */
	__s32		     minimum;	/* Note signedness */
	__s32		     maximum;
	__s32		     step;
	__s32		     default_value;
	__u32                flags;
	__u32		     reserved[2];
};
```
包含了最大值，最小值，步长，默认值，控制名称等信息。
通过传递过来的控制id和类型遍历，并将遍历的结果保存在 v4l2_queryctrl中返回。

顺带说一下，在这个文件下面这个函数也很重要：

```int uvc_ctrl_set(struct uvc_video_chain *chain,struct v4l2_ext_control *xctrl)```，这个函数掌控着控制的大权。其中```v4l2_ext_control```参数就是上层传递过来的控制参数，非常重要。

从相关控制参数一路分析过来，大致明白了控制的过程和控制代码的逻辑，下一步就是熟悉USB协议之后对这个参数做出定制，以适合不同厂商的摄像头。