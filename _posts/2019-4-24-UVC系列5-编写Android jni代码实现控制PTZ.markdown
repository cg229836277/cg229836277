---
layout: post
title:  "UVC系列5-编写Android jni代码实现控制PTZ"
date:   2019-4-24 18:19:25 +0800
categories: Android
---


> 微信公众号：Android部落格

## 1、编写上层jni代码

在Android kernel层完成定制之后，需要写app实现对摄像头的控制，主要通过jni代码实现。
在jni代码中主要定义这几个函数：

```jintArrayJava_com_chuck_android_uvccamera_MainActivity1_startControlCamera(JNIEnv*env,jobject thiz , jint controlId ,jint value)```

实现控制的主函数:

```c
//实现获取当前控制参数的值
jintJava_com_chuck_android_uvccamera_MainActivity1_getCurrentControlValue
//判断当前摄像头是否支持PTZ控制。
jbooleanJava_com_chuck_android_uvccamera_UVCCameraPreview_isSupportPtz
```

现在逐个函数说明，startControlCamera函数如下：

```c
//返回控制是否成功
void Java_com_chuck_android_uvccamera_MainActivity_startControlCamera(JNIEnv* env,jobject thiz , jint controlId ,jint value){
    switch (controlId) {
        case V4L2_CID_PAN_RELATIVE:
            //控制左右转动
            __android_log_print(ANDROID_LOG_ERROR , TAG , "控制左右的值是 = %d" , value);
//          controlId = V4L2_CID_PAN_ABSOLUTE;
            controlId = V4L2_CID_PAN_RELATIVE;
//          controlId = PAN_SPEED_ID;
//          controlType = "V4L2_CID_PAN_ABSOLUTE ";
            break;
        case V4L2_CID_TILT_RELATIVE:
            //控制上下转动
            __android_log_print(ANDROID_LOG_ERROR , TAG , "控制上下的值是 = %d" , value);
//          controlId = V4L2_CID_TILT_ABSOLUTE;
            controlId = V4L2_CID_TILT_RELATIVE;
//          controlId = TILT_SPEED_ID;
//          controlType = "V4L2_CID_TILT_ABSOLUTE ";
            break;
        case ZOOM_RELATIVE_ID:
            //控制聚焦
            controlId = ZOOM_RELATIVE_ID;
            break;
        case PAN_SPEED_ID:
            controlId = PAN_SPEED_ID;
            break;
        case TILT_SPEED_ID:
            controlId = TILT_SPEED_ID;
            break;
        default:
            break;
    }
    int result = startControlPanTilt(controlId , value);
    if(result != 0){
        __android_log_print(ANDROID_LOG_ERROR , TAG , "控制失败");
    }
}
```
这里的controlId就是我们在kernel添加的id，value就是每个控制id对应的值，值要按照UVC协议规定的传递。

## 2、添加控制函数
接下来看看startControlPanTilt函数：

```c
int startControlPanTilt(int controlId , int value){
    //getControlValue(controlId);
    __android_log_print(ANDROID_LOG_ERROR , TAG , "start controlId = %d , value = %d" , controlId , value);
    //jint setBefore;
    if(fd < 0){
        __android_log_print(ANDROID_LOG_ERROR , TAG , "device open fail");
        return -1;
    }
    struct v4l2_control ctrl;
 
    CLEAR(ctrl);
 
    ctrl.id = controlId;
    ctrl.value = value;
 
    int result = ioctl(fd, VIDIOC_S_CTRL, &ctrl);
    __android_log_print(ANDROID_LOG_ERROR , TAG , "result = %d , controlId = %d , controlValue = %d" , result , ctrl.id , ctrl.value);
    if (result == -1){
        __android_log_print(ANDROID_LOG_ERROR , TAG , "VIDIOC_S_EXT_CTRLS failed while tilting down , reason = %s" , strerror (errno));
        return -1;
    }else{
        __android_log_print(ANDROID_LOG_ERROR , TAG , "VIDIOC_S_EXT_CTRLS success while tilting down , after value = %d" , ctrl.value);
    }
    return 0;
}
```
可以看到控制的核心函数是linux的ioctl，传入的参数是`v4l2_control`，这个结构体里面有什么东西呢，可以看看其定义[https://www.linuxtv.org/downloads/legacy/video4linux/API/V4L2_API/spec-single/v4l2.html#v4l2-control](https://www.linuxtv.org/downloads/legacy/video4linux/API/V4L2_API/spec-single/v4l2.html#v4l2-control)：
![](https://user-gold-cdn.xitu.io/2020/5/26/1724ebc3e66760ea?w=1489&h=567&f=png&s=23947)

### 2.1 赋值
结构体里面就id和value，按照UVC协议赋值，其他的交给ioctrl就可以了。此时ioctrl到了底层到了哪里呢？其实到了drivers/media/usb/uvc/uvc_v4l2.c文件中的`uvc_v4l2_do_ioctl`方法，在这个方法中需要关注的是以下几行代码：

```c
case VIDIOC_S_CTRL:
{
	struct v4l2_control *ctrl = arg;
	struct v4l2_ext_control xctrl;

	memset(&xctrl, 0, sizeof xctrl);
	xctrl.id = ctrl->id;
	xctrl.value = ctrl->value;

	uvc_ctrl_begin(video);
	ret = uvc_ctrl_set(video, &xctrl);
	if (ret < 0) {
		uvc_ctrl_rollback(video);
		return ret;
	}
	ret = uvc_ctrl_commit(video);
	break;
}
```
`v4l2_prio_check`应该是检查是否有执行的优先级，没有执行的优先级的话返回当前繁忙设备，如下：

```c
int v4l2_prio_check(struct v4l2_prio_state*global, enum v4l2_priority local)
{
	return(local < v4l2_prio_max(global)) ? -EBUSY : 0;
}
```
接下来的执行顺序与数据库的插入操作比较类似，开始执行控制，设置相关参数，检查控制结果，失败的话回滚操作，否则提交执行，同时记住当前的设定值。

### 2.2 获取参数值
第二个需要关注的是实现获取当前控制参数，getControlValue，有两种写法，一种是不用循环，一种是循环遍历UVC控制参数，查看设备当前控制参数的值，如下：

```c
int getControlValues(int controlId){
    //an array of v4l2_ext_control
    struct v4l2_ext_control clist[1];
    struct v4l2_ext_controls ctrls;
 
    CLEAR(clist);
    CLEAR(ctrls);
 
    clist[0].id    = controlId;
    clist[0].value = 0;
 
    //v4l2_ext_controls with list of v4l2_ext_control
    ctrls.ctrl_class = V4L2_CTRL_CLASS_CAMERA;
    ctrls.count = 1;
    ctrls.controls = clist;
 
    //read back the value
    if (-1 == ioctl (fd, VIDIOC_G_EXT_CTRLS, &ctrls))
    {
        __android_log_print(ANDROID_LOG_ERROR,TAG,"get current value failed fd = %d,reason=%s" , fd,strerror(errno));
        return -1;
    }
    __android_log_print(ANDROID_LOG_ERROR,TAG,"get before value success , %d" , clist[0].value);
    return clist[0].value;
}
 
int getControlValue(int controlId){
    //an array of v4l2_ext_control
    struct v4l2_control ctrl;
 
    CLEAR(ctrl);
 
    ctrl.id    = controlId;
    ctrl.value = 0;
 
    //read back the value
    if (-1 == ioctl (fd, VIDIOC_G_CTRL, &ctrl))
    {
        __android_log_print(ANDROID_LOG_ERROR,TAG,"get current value failed fd = %d,reason=%s" , fd,strerror(errno));
        return -1;
    }
    __android_log_print(ANDROID_LOG_ERROR,TAG,"get before value success , %d" , ctrl.value);
    return ctrl.value;
}
```
这里获取控制id当前的值也是在drivers/media/usb/uvc/uvc_v4l2.c的uvc_v4l2_do_ioctl方法中，其中使用的获取参数是`VIDIOC_G_EXT_CTRLS`，其实查询单个控制参数的值用`VIDIOC_G_CTRL`也可以，只不过`VIDIOC_G_EXT_CTRLS`是多个控制参数一起查找，在底层源码里面是循环遍历查找，而`VIDIOC_G_CTRL`是直接查找，相关源码如下：

```c
case VIDIOC_G_CTRL:
{
	struct v4l2_control *ctrl = arg;
	struct v4l2_ext_control xctrl;

	memset(&xctrl, 0, sizeof xctrl);
	xctrl.id = ctrl->id;

	uvc_ctrl_begin(video);
	ret = uvc_ctrl_get(video, &xctrl);
	uvc_ctrl_rollback(video);
	if (ret >= 0)
		ctrl->value = xctrl.value;
	break;
}
case VIDIOC_G_EXT_CTRLS:
{
	struct v4l2_ext_controls *ctrls = arg;
	struct v4l2_ext_control *ctrl = ctrls->controls;
	unsigned int i;

	uvc_ctrl_begin(video);
	for (i = 0; i < ctrls->count; ++ctrl, ++i) {
		ret = uvc_ctrl_get(video, ctrl);
		if (ret < 0) {
			uvc_ctrl_rollback(video);
			ctrls->error_idx = i;
			return ret;
		}
	}
	ctrls->error_idx = 0;
	ret = uvc_ctrl_rollback(video);
	break;
}
```
 有关v4l2_ext_controls和v4l2_ext_control的控制参数见下图：
![](https://user-gold-cdn.xitu.io/2020/5/26/1724ebc3e8e18088?w=1860&h=259&f=png&s=25418)
![](https://user-gold-cdn.xitu.io/2020/5/26/1724ebc3e95f7426?w=1910&h=377&f=png&s=45107)

## 3、查看设备能力
现在看看判断当前摄像头是否支持PTZ控制的函数，实际上是遍历摄像头的控制id，查看是否有PTZ相关的控制id，函数中定义的几个变量值定义如下:

```c
const char* CONTROL_FLAG_PAN = "Pan(Absolute)";
const char* CONTROL_FLAG_TILT = "Tilt(Absolute)";
const char* CONTROL_FLAG_ZOOM = "Zoom,Absolute";
const int PAN_SPEED_ID = 10094880;
const int TILT_SPEED_ID = 10094881;
const int ZOOM_RELATIVE_ID = 10094863;
```
详细的函数如下:

```c
jboolean queryControls(){
    jint canControl = 0;
    __android_log_print(ANDROID_LOG_ERROR , TAG , "设备号 = %d" , fd);
 
    struct v4l2_queryctrl qctrl;
    qctrl.id = V4L2_CTRL_CLASS_CAMERA | V4L2_CTRL_FLAG_NEXT_CTRL;
    int i = ioctl(fd, VIDIOC_QUERYCTRL, &qctrl);
    LOGE("error %d, %s , %d",errno, strerror (errno) , i);
    while (0 == i){
        __android_log_print(ANDROID_LOG_ERROR , TAG , "开始查找");
        if (V4L2_CTRL_ID2CLASS(qctrl.id) != V4L2_CTRL_CLASS_CAMERA)
            continue;
 
        if(strcmp(qctrl.name , CONTROL_FLAG_PAN) == 0 ){
            ++canControl;
            panMin = qctrl.maximum;
            panMax = qctrl.minimum;
            __android_log_print(ANDROID_LOG_ERROR , TAG , "panMin=%d , panMax=%d , panStep=%d" , panMin , panMax , qctrl.step);
        }
        if(strcmp(qctrl.name , CONTROL_FLAG_TILT) == 0){
            ++canControl;
            tiltMin = qctrl.maximum;
            tiltMax = qctrl.minimum;
            __android_log_print(ANDROID_LOG_ERROR , TAG , "tiltMin=%d , tiltMax=%d , tiltStep=%d" , tiltMin , tiltMax , qctrl.step);
        }
        if(strcmp(qctrl.name , CONTROL_FLAG_ZOOM) == 0){
            ++canControl;
            zoomMin = qctrl.maximum;
            zoomMax = qctrl.minimum;
            __android_log_print(ANDROID_LOG_ERROR , TAG , "zoomMin=%d , zoomMax=%d , zoomStep=%d" , zoomMin , zoomMax , qctrl.step);
        }
 
        __android_log_print(ANDROID_LOG_ERROR , TAG , "找到的控制函数是%s" , qctrl.name);
        __android_log_print(ANDROID_LOG_ERROR , TAG , "继续查找");
        __android_log_print(ANDROID_LOG_ERROR , TAG , "id = %d" , qctrl.id);
        __android_log_print(ANDROID_LOG_ERROR , TAG , "Next_Ctrl = %x" , V4L2_CTRL_FLAG_NEXT_CTRL);
        __android_log_print(ANDROID_LOG_ERROR , TAG , "Camera_Class = %x" , V4L2_CTRL_CLASS_CAMERA);
 
        qctrl.id |= V4L2_CTRL_FLAG_NEXT_CTRL;
 
        __android_log_print(ANDROID_LOG_ERROR , TAG , "id+ = %x" , qctrl.id);
 
        i = ioctl(fd, VIDIOC_QUERYCTRL, &qctrl);
        if(i != 0){
            __android_log_print(ANDROID_LOG_ERROR, TAG,"uvcioc ctrl add error: errno=%d (reason=%s)\n", errno,strerror(errno));
        }
    }
    //如果存在ptz控制的话，应该会有Pan,Tilt,Zoom字符串，变量自加三次
    return canControl == 3;
}
```
这里可以看到这个函数里面详细输出了支持哪些控制参数，各个控制参数的最大值，最小值，步长等信息，这些查找最后在底层的调用是在drivers\media\usb\uvc\uvc_ctrl.c的__uvc_query_v4l2_ctrl函数中，具体的描述在前一篇文章中有详细的讲述。

## 4、编译脚本
上述代码编写完毕，写MK文件，如下：

```mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE    := UVCCamera
LOCAL_SRC_FILES := UVCCamera_Rel.cpp
LOCAL_LDLIBS    := -llog -ljnigraphics
include $(BUILD_SHARED_LIBRARY)
```
app层的代码编写完毕，在AS中编译运行生成so文件，在app中就可以使用了。同时要注意的是，这个app的权限级别要比较高，不然可能没有权限执行ioctrl。

另外这个代码中还涉及到了打开/dev/video0文件的操作，如下：

```c
int opendevice(int i) {
    struct stat st;
 
    sprintf(dev_name,"/dev/video%d",i);
 
    if (-1 == stat (dev_name, &st)) {
        __android_log_print(ANDROID_LOG_ERROR , TAG , "Cannot identify '%s': %d, %s", dev_name, errno, strerror (errno));
        LOGE("Cannot identify '%s': %d, %s", dev_name, errno, strerror (errno));
        return ERROR_LOCAL;
    }
    __android_log_print(ANDROID_LOG_ERROR , TAG , "device is %s", dev_name);
    if (!S_ISCHR (st.st_mode)) {
        LOGE("%s is no device", dev_name);
        __android_log_print(ANDROID_LOG_ERROR , TAG , "%s is no device", dev_name);
        return ERROR_LOCAL;
    }
 
    fd = open (dev_name, O_RDWR | O_NONBLOCK, 0);
 
    if (-1 == fd) {
        LOGE("Cannot open '%s': %d, %s", dev_name, errno, strerror (errno));
        return ERROR_LOCAL;
    }else{
        LOGE("open device success %d" , fd);
 
        queryControls();
    }
    return SUCCESS_LOCAL;
}
```

## 5、总结
至此，android UVC控制云台摄像头系列完成，在整个方案研发的过程中最大的阻碍是android kernel不支持相对控制，需要搜索相关资料自己去打patch，打了patch之后还要不断的调试试错；另外一个阻碍是摄像头厂商不同UVC版本的摄像头，这是在Android适配的过程中出现各种报错的主要原因之一。

在这个过程中涉及到了kernel层源码的解读，kernel源码的修改和调试，kernel层的编译和打包刷机，android jni代码等。

微信公众号:Android部落格