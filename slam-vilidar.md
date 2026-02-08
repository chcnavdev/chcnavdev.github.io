# ViLidar SDK 接口说明

ViLidar是基于SLAM（Simultaneous Localization and
Mapping）技术的视觉激光雷达定位解决方案，结合GNSS、激光雷达和相机数据，为用户提供高精度的位置信息。与SFix相比，ViLidar增加了相机视频流、拍照和像素点转坐标功能，可以实现通过点击图片上的像素点获取实际地理坐标。

本文档详细介绍了如何在Android应用中集成和使用ViLidar功能。

## ViLidar 一般工作流程

1. 注册监听器
```java
SlamDeviceManager.getInstance().addListener(slamListener);
```
2. 打开相机
```java
videoDrafter = new SlamVideoFrameDrafter<>(mRtkImageView, new ImageSurveyVideoLayer());
SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
camera.setCallback(videoDrafter);
camera.open();
```
3. 打开IMU并开启 ViLidar，开始 SLAM 初始化
```java
NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8);
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(
                App.sInstance.getApplicationContext(),
                startInfo);
SlamDeviceManager.getInstance().openViLidar();
```
4. 拍照记录当前帧 GPS 时间，并调用接口预处理
```java
SlamDeviceManager.getInstance().preProcessPicture(gpsTime);
```
5. 获取用户在图片中选择的像素点
```java
PointF p = pictureView.getCurrentPointInfo();
```
6. 将像素点转坐标
```java
SlamDeviceManager.getInstance().calcCoordinateByPixel(gpsTime, x, y);
```

## 核心组件

### SlamDeviceManager

负责 ViLidar 的所有生命周期控制：

| 方法 | 说明 |
|------|------|
| `openViLidar()` | 开启 ViLidar 工作模式。触发 onSceneStatus 回调。 |
| `pauseSlam()` | 暂停 SLAM。 |
| `close()` | 停止 SLAM|
| `preProcessPicture(gpsTime)` | 将指定时间点的视频帧生成用于像素点转坐标的“可选点图片”。 |
| `calcCoordinateByPixel(gpsTime, x, y)` | 将图片上的像素（x,y）转换为实际坐标。 |
| `addListener(listener)` | 注册 SLAM 监听器（应用必须在启动前注册）。 |
| `removeListener(listener)` | 移除监听器（建议在退出或onDestroy时调用）。 |

---

### SlamCamera

- 控制相机开关
- 提供实时视频帧（VideoFrame）用于预览与拍照

关键接口：

```java
SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA); // 选择相机类型
camera.setCallback(videoDrafter); // 设置视频帧绘制器回调
camera.open(); // 打开相机
camera.close(); // 关闭相机
```

### SlamVideoFrameDrafter

SlamVideoFrameDrafter 是 SDK 中负责相机视频帧绘制与帧数据分发的核心组件。它连接底层相机数据与应用层 UI，通过将实时视频帧绘制到指定视图上，提供稳定的预览画面。

- 与 UI 绑定（如 RtkImageView）
```java
mRtkImageView = findViewById(R.id.viewSlam);
mDrafter = new SlamVideoFrameDrafter<>(mRtkImageView, new ImageSurveyVideoLayer());
```
- 控制绘制暂停
```java
mDrafter.setStopDrafter(isPause);
```
### ISlamDeviceListener

ISlamDeviceListener 是 ViLidar SDK 的核心回调接口，用于向上层应用实时通知 SLAM 系统的运行状态、初始化进度、图片预处理结果、像素点转坐标结果，以及设备电量和存储等信息。应用通过实现该接口，可以在 ViLidar 启动、暂停、关闭、初始化流程、拍照处理以及坐标转换过程中获取必要的状态反馈，从而控制 UI 展示、流程调度和业务逻辑执行。

1. ViLidar打开状态监听 
回调：`onSceneStatus`
触发时机：调用 openViLidar()、pauseSlam() 、close() 后

2. SLAM 初始化状态变化
回调：`onInitStatus`
触发时机：ViLidar 启动后，初始化状态每变化一次都会触发

3. 图片预处理结果
回调：`onPreProcessPicResult`
触发时机：调用`preProcessPicture()`  后

4. 像素点转坐标结果
回调：`onPixelToCoordinateResult`
触发时机：调用 `calcCoordinateByPixel()` 后

5. 获取实时电量
回调：`onBatteryInfoChanged`

6. 获取实时存储信息变化
回调：`onStorageInfoChanged`


### ViLidarPictureViewLayout

ViLidarPictureViewLayout 是 SDK 提供的可图片显示与像素点选择组件，用于在图片预处理成功后展示由相机帧生成的静态图像。该组件支持用户在图片上直接点击选取像素点，并将选中的坐标返回给应用层，以便进一步用于像素点转地理坐标的计算。一般用法如下：

```java
// 设置显示的图片
pictureView.setBitmapByte(imgBuffer);
// 获取用户点击选择像素点坐标
PointF pt = pictureView.getCurrentPointInfo();
```


### SlamInitGuideViewLayout（可选）

SlamInitGuideViewLayout 是 SDK 提供的可选 SLAM 初始化引导界面组件，用于根据 SLAM 初始化过程中的实时状态变化，向用户展示对应的操作提示与引导动画。SLAM 初始化通常包含静止、移动、GNSS 初始化、融合运行等多个阶段，而该组件能自动根据 onInitStatus 回调更新界面内容，帮助用户理解当前步骤应如何配合设备，从而更顺利地完成初始化。通过集成该组件，开发者无需自行设计初始化流程 UI，即可实现标准化、友好的初始化引导体验，是提升 ViLidar 使用易用性的重要辅助模块。



