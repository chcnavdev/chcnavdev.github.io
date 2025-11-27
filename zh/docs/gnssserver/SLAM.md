## SLAM 集成
本开发文档用于指导第三方开发者在 Android 应用中集成 SLAM 功能。  
SLAM 系统包含两种模式：

- **SFix**：GNSS + 激光雷达融合定位
- **ViLiDar**：在 SFix 基础上增加相机、拍照、像素点转坐标能力

两种模式通过 SLAM 管理接口 `ISlamDeviceManager` 来完成 SLAM 的启动、暂停、关闭等控制。



## 获取 ISlamDeviceManager

SLAM 功能通过`ISlamDeviceManager`暴露给应用层，因此开发者需要通过 GNSS 服务获得 `ISlamDeviceManager` 实例。

示例代码如下：

```java
// 获取 GNSS 服务
IGnssServiceBinder service = GnssServiceManager.getInstance().getService();
if (service == null) {
    // 服务未连接
    return;
}

// 从 GNSS 服务获取 SLAM 设备管理器
ISlamDeviceManager slamManager =
        ISlamDeviceManager.Stub.asInterface(service.getSlamDeviceManager());

// 之后即可使用 slamManager 调用 SLAM 框架提供的所有接口
```

> **说明：** SLAM 所有操作均通过 `ISlamDeviceManager` 完成



## SLAM 基础接口

`ISlamDeviceManager` 提供 SLAM 的核心控制接口：

| 方法 | 说明                 |
||--|
| `openSfix()` | 开启 SFix 模式         |
| `recoverySfix()` | 从暂停恢复 SFix         |
| `openViLidar()` | 开启 ViLiDar 模式      |
| `recoveryViLidar()` | 从暂停恢复 ViLiDar      |
| `pauseSlam()` | 暂停 SFix/ViLiDar 工作 |
| `close()` | 停止 SFix/ViLiDar 工作 |
| `addListener(ISlamDeviceListener)` | 注册监听器              |
| `removeListener(ISlamDeviceListener)` | 移除监听器              |
| `preProcessPicture(gpsTime)` | （ViLiDar）帧预处理      |
| `calcCoordinateByPixel(gpsTime,x,y)` | （ViLiDar）像素点转坐标    |



## SLAM 状态监听（ISlamDeviceListener）

开发者必须注册监听器以获取 SLAM 事件通知：

```java
slamManager.addListener(new ISlamDeviceListener.Stub() {

    @Override
    public void onSceneStatus(SlamSceneStatus status) {
        // SLAM 开启/暂停/关闭 结果
    }
    
    @Override
    public void onInitStatus(SlamInitStatus status) {
        // 初始化状态变化
    }
    
    @Override
    public void onPreProcessPicResult(SlamPreProcessPictureStatus slamPreProcessPictureStatus) {
        // ViLidar 图片预处理结果
    }
    
    @Override
    public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus slamSampleCoordinateResultStatus) {
        // ViLidar 像素点转坐标结果
    }
    
    @Override
    public void onBatteryInfoChanged(BatteryLevelDetail batteryLevelDetail) {}
    
    @Override
    public void onStorageInfoChanged(SlamDeviceStorageInfo storage) {}
});
```



## SLAM 初始化流程说明（通用）

开发者可通过 `onInitStatus()` 获取 SLAM 初始化的各个阶段变化，从而决定：

- 是否显示初始化界面  
- 是否允许用户开始测量  
- 是否提示用户移动设备或静止等待  

示例：

```java
@Override
public void onInitStatus(SlamInitStatus status) {
    switch (status.getSlamInitStatus()) {
        case SYSTEM_STATUS_INITIAL:
            // 需要静止
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            // 按提示移动
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            // 初始化完成，可开始测量
            break;
    }
}
```



## UI 组件

SDK 提供部分 UI 组件，开发者如需可以直接使用。

### SlamInitGuideViewLayout（可选的初始化流程引导 UI）

用于展示 SLAM 初始化步骤，例如“请保持静止”、“请移动设备”、“初始化完成”等。

示例：

```java
@Override
public void onInitStatus(SlamInitStatus status) {
    if (initGuideView != null) {
        initGuideView.updateStatus(status);
    }
}
```



### ViLiDarPictureViewLayout（ViLiDar 图片显示 + 像素点选择）

用于显示预处理后的图，并让用户点击选择像素点。

```java
pictureView.setBitmapByte(imgBytes);
PointF point = pictureView.getCurrentPointInfo();
```



### SlamVideoFrameDrafter（相机实时帧绘制）

将相机实时帧绘制到指定 View。

```java
SlamVideoFrameDrafter drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());
```

暂停绘制：

```java
drafter.setStopDrafter(true);
```



## SFix 集成指南

SFix = **GNSS + 激光雷达** 融合定位



### 启动 SFix

```java
// 1. 启动惯导
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

// 2. 开启 SFix
slamManager.openSfix();
```



### 暂停 / 恢复 / 停止

```java
slamManager.pauseSlam();      // 暂停
slamManager.recoverySfix();  // 恢复
slamManager.close();         // 停止
```



### 获取实时位置（PositionSource）

```java
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo info) {
        Position pos = info.getCoorKernal().getWgsBlh();
        double lat = pos.getX(); // 纬度
        double lon = pos.getY(); // 经度
        double h   = pos.getZ(); // 高程
    }
});
```



## ViLiDar 集成指南

ViLiDar 在 SFix 基础上扩展了相机能力，可实现 **像素点 → 地理坐标**。



### 整体流程

1. 注册 SLAM 监听器  
2. 打开相机  
3. 启动惯导  
4. `openViLidar()` 开启 SLAM  
5. 调用 `preProcessPicture()` 获取静态图  
6. 用户点击像素点  
7. 调用 `calcCoordinateByPixel()` 转坐标  



### 打开相机

```java
SlamVideoFrameDrafter drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());

SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
camera.setCallback(drafter);
camera.open();
```



### 启动 ViLiDar

```java
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);

ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

slamManager.openViLidar();
```



### 拍照与图片预处理

```java
slamManager.preProcessPicture(gpsTime);
```

通过`ISlamDeviceListener`监听结果监听结果：

```java
@Override
public void onPreProcessPicResult(SlamPreProcessPictureStatus result) {
    // 获取静态帧数据
}
```



### 获取像素点

```java
PointF pt = pictureView.getCurrentPointInfo();
```



### 像素点转坐标

```java
slamManager.calcCoordinateByPixel(gpsTime, x, y);
```

通过`ISlamDeviceListener`监听结果：

```java
@Override
public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus result) {
    // 获取对应地理坐标
}
```



## 资源清理

```java
slamManager.removeListener(slamListener);
PositionSource.getInstance().removeListener(positionListener);
```


