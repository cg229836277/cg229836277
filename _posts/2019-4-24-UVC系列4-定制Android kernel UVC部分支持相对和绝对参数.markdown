---
layout: post
title:  "UVC系列4-定制Android kernel UVC部分支持相对和绝对参数"
date:   2019-4-24 18:19:25 +0800
categories: Android
---


> 微信公众号：Android部落格


##  1、添加参数

在熟悉了android uvc控制参数和UVC协议之后，现在可以着手定制android UVC协议了，添加相对控制参数。
### 1.1 添加相对控制pan和tilt

第一步，添加相对控制pan和tilt修改的文件是：drivers\media\usb\uvc\uvc_ctrl.c
在`uvc_control_info uvc_ctrls[]`结构体中添加：

```c
{
	.entity     = UVC_GUID_UVC_CAMERA,
	.selector  	= UVC_CT_PANTILT_RELATIVE_CONTROL,
	.index      = 12,
	.size       = 4,
	.flags     	= UVC_CTRL_FLAG_SET_CUR
	             |UVC_CTRL_FLAG_GET_RANGE
	             |UVC_CTRL_FLAG_AUTO_UPDATE,
}
```
在结构体`uvc_control_mapping uvc_ctrl_mappings[]`中添加：

```c
{
	.id = V4L2_CID_PAN_RELATIVE,
	.name = "Pan (Relative)",
	.entity = UVC_GUID_UVC_CAMERA,
	.selector = UVC_CT_PANTILT_RELATIVE_CONTROL,
	.size = 16,
	.offset = 0,
	.v4l2_type = V4L2_CTRL_TYPE_INTEGER,
	.data_type = UVC_CTRL_DATA_TYPE_SIGNED,
	.get = uvc_ctrl_get_rel_speed,
	.set = uvc_ctrl_set_rel_speed,
},
{
	.id = V4L2_CID_TILT_RELATIVE,
	.name= "Tilt (Relative)",
	.entity= UVC_GUID_UVC_CAMERA,
	.selector= UVC_CT_PANTILT_RELATIVE_CONTROL,
	.size= 16,
	.offset= 16,
	.v4l2_type= V4L2_CTRL_TYPE_INTEGER,
	.data_type= UVC_CTRL_DATA_TYPE_SIGNED,
	.get= uvc_ctrl_get_rel_speed,
	.set= uvc_ctrl_set_rel_speed,
}
```

### 1.2 添加pan和tilt的速度控制
其中`uvc_ctrl_get_rel_speed`和`uvc_ctrl_set_rel_speed`映射到的方法对应UVC协议里面的速度控制，在uvc_ctrl.c文件中也要添加这两个方法的实现，与zoom对应的控制方法类似，具体实现方法是：

```c
static __s32 uvc_ctrl_get_rel_speed(structuvc_control_mapping *mapping,__u8query, const __u8 *data)
{
	intfirst = mapping->offset / 8;
	__s8rel = (__s8)data[first];
	switch (query) {
	case UVC_GET_CUR:
		return (rel == 0) ? 0 : (rel > 0 ?data[first+1]:-data[first+1]);
	case UVC_GET_MIN:
		return -data[first+1];
	case UVC_GET_MAX:
	case UVC_GET_RES:
	case UVC_GET_DEF:
	default:
		return data[first+1];
}
}
static void uvc_ctrl_set_rel_speed(structuvc_control_mapping *mapping,__s32 value, __u8 *data)
{
	intfirst = mapping->offset / 8;
	data[first] = value == 0 ? 0 : (value > 0)? 1 : 0xff;
	data[first+1] = min_t(int, abs(value), 0xff);
}
```
可以看到这里的赋值也是与UVC协议对应的。另外针对绝对控制，目前在结构体`uvc_control_mappinguvc_ctrl_mappings[]`中的定义是：

```c
{
	.id             = V4L2_CID_PAN_ABSOLUTE,
	.name               = "Pan (Absolute)",
	.entity               = UVC_GUID_UVC_CAMERA,
	.selector  = UVC_CT_PANTILT_ABSOLUTE_CONTROL,
	.size          = 32,
	.offset              = 0,
	.v4l2_type        = V4L2_CTRL_TYPE_INTEGER,
	.data_type       = UVC_CTRL_DATA_TYPE_UNSIGNED,
},
{
	.id             = V4L2_CID_TILT_ABSOLUTE,
	.name               = "Tilt (Absolute)",
	.entity               = UVC_GUID_UVC_CAMERA,
	.selector  = UVC_CT_PANTILT_ABSOLUTE_CONTROL,
	.size          = 32,
	.offset              = 32,
	.v4l2_type        = V4L2_CTRL_TYPE_INTEGER,
	.data_type       = UVC_CTRL_DATA_TYPE_UNSIGNED,
}
```
可以看看这两个控制参数的data_type是`UVC_CTRL_DATA_TYPE_UNSIGNED`，而UVC协议里面定义的是：
![](https://user-gold-cdn.xitu.io/2020/5/26/1724eb883111e240?w=840&h=307&f=webp&s=8780)

### 1.3 修改参数类型
Value的类型是signed number，此时我们需要将UNSIGNED改为signed，将这个data_type统一改成signed，即`UVC_CTRL_DATA_TYPE_SIGNED`。
下一步`uvc_control_mapping uvc_ctrl_mappings[]`中添加速度控制的参数，如下：

```c
{
	.id             = V4L2_CID_PAN_SPEED,
	.name               = "Pan (Speed)",
	.entity               = UVC_GUID_UVC_CAMERA,
	.selector  = UVC_CT_PANTILT_RELATIVE_CONTROL,
	.size          = 16,
	.offset              = 0,
	.v4l2_type        = V4L2_CTRL_TYPE_INTEGER,
	.data_type       = UVC_CTRL_DATA_TYPE_SIGNED,
	.get          = uvc_ctrl_get_rel_speed,
	.set           = uvc_ctrl_set_rel_speed,
},
{
	.id             = V4L2_CID_TILT_SPEED,
	.name               = "Tilt (Speed)",
	.entity               = UVC_GUID_UVC_CAMERA,
	.selector  = UVC_CT_PANTILT_RELATIVE_CONTROL,
	.size          =16,
	.offset              = 16,
	.v4l2_type        = V4L2_CTRL_TYPE_INTEGER,
	.data_type       = UVC_CTRL_DATA_TYPE_SIGNED,
	.get          = uvc_ctrl_get_rel_speed,
	.set           = uvc_ctrl_set_rel_speed,
}
```
针对相对控制的两个参数id `V4L2_CID_PAN_RELATIVE`和`V4L2_CID_PAN_RELATIVE`，两个控制速度的参数`V4L2_CID_PAN_SPEED`和`V4L2_CID_TILT_SPEED`需要定义，修改两个文件，第一个文件位置位于drivers\media\v4l2-core\v4l2-ctrls.c文件中，`const char *v4l2_ctrl_get_name`中添加：

```c
caseV4L2_CID_PAN_RELATIVE:      return"Pan, Relative";
caseV4L2_CID_TILT_RELATIVE:     return"Tilt, Relative";
caseV4L2_CID_PAN_SPEED:         return"Pan, Speed";
caseV4L2_CID_TILT_SPEED:        return"Tilt, Speed";
```
第二个文件位于include/uapi/linux/v4l2-controls.h，添加定义：

```c
#define V4L2_CID_PAN_RELATIVE                   (V4L2_CID_CAMERA_CLASS_BASE+4)
#define V4L2_CID_TILT_RELATIVE                  (V4L2_CID_CAMERA_CLASS_BASE+5)
#define V4L2_CID_PAN_SPEED                      (V4L2_CID_CAMERA_CLASS_BASE+32)
#define V4L2_CID_TILT_SPEED                     (V4L2_CID_CAMERA_CLASS_BASE+33)
```

### 1.4 xml文件修改
另外还有两个xml说明文件，需要添加这两个控制的说明，分别是：
> Documentation/DocBook/media/v4l/controls.xml
> Documentation/DocBook/media/v4l/compat.xml

具体修改网址可以参考：
[https://patchwork.kernel.org/patch/4836491/](https://patchwork.kernel.org/patch/4836491/)

至此，android UVC kernel部分定制完毕，下一步就是打通app到底层kernel的通道,将这些代码合入完毕之后，开始编译kernel代码，并刷机重启。

微信公众号:Android部落格