# SLAM 集成开发文档（SFix + ViLidar）
（Markdown 源码，不渲染）

本开发文档用于指导第三方开发者在 Android 应用中集成 SLAM 功能。  
SLAM 系统包含两种模式：

- **SFix**：GNSS + 激光雷达融合定位（无相机）
- **ViLidar**：在 SFix 基础上增加相机、拍照、像素点转坐标能力

两种模式均依赖 AIDL 服务获取 SLAM 管理接口 `ISlamDeviceManager`，并通过其完成 SLAM 的启动、暂停、关闭等控制。

---

# 1. 获取 ISlamDeviceManager（最重要）

SLAM 功能通过 AIDL 暴露给应用层，因此开发者需要通过 GNSS 服务获得 `ISlamDeviceManager` 实例。

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

---

# 2. SLAM 基础接口

`ISlamDeviceManager` 提供 SLAM 的核心控制接口：

| 方法 | 说明 |
|------|------|
| `openSfix()` | 开启 SFix 模式 |
| `recoverySfix()` | 从暂停恢复 SFix |
| `openViLidar()` | 开启 ViLidar 模式 |
| `recoveryViLidar()` | 从暂停恢复 ViLidar |
| `pauseSlam()` | 暂停 SFix/ViLiDar 工作 |
| `close()` | 停止 SFix/ViLiDar 工作 |
| `addListener(ISlamDeviceListener)` | 注册监听器 |
| `removeListener(ISlamDeviceListener)` | 移除监听器 |
| `preProcessPicture(gpsTime)` |（ViLidar）帧预处理 |
| `calcCoordinateByPixel(gpsTime,x,y)` |（ViLidar）像素点转坐标 |

---

# 3. SLAM 状态监听（ISlamDeviceListener）

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
    public void onPreProcessPicResult(byte[] img) {
        // ViLidar 图片预处理结果
    }
    
    @Override
    public void onPixelToCoordinateResult(CoordinateInfo info) {
        // ViLidar 像素点转坐标结果
    }
    
    @Override
    public void onBatteryInfoChanged(int power) {}
    
    @Override
    public void onStorageInfoChanged(SlamDeviceStorageInfo storage) {}
});
```

---

# 4. SFix 集成指南

SFix = **GNSS + 激光雷达** 融合定位  
不依赖相机，无拍照、无像素点转换。

---

## 4.1 启动 SFix

```java
// 1. 启动惯导
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

// 2. 开启 SFix
slamManager.openSfix();
```

---

## 4.2 暂停 / 恢复 / 停止

```java
slamManager.pauseSlam();      // 暂停
slamManager.recoverySfix();  // 恢复
slamManager.close();         // 停止
```

---

## 4.3 获取实时位置（PositionSource）

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

---

# 5. ViLidar 集成指南

ViLidar 在 SFix 基础上扩展了相机能力，可实现 **像素点 → 地理坐标**。

---

# 5.1 整体流程

1. 注册 SLAM 监听器  
2. 打开相机  
3. 启动惯导  
4. `openViLidar()` 开启 SLAM  
5. 调用 `preProcessPicture()` 获取静态图  
6. 用户点击像素点  
7. 调用 `calcCoordinateByPixel()` 转坐标  

---

# 5.2 打开相机

```java
SlamVideoFrameDrafter drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());

SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
camera.setCallback(drafter);
camera.open();
```

---

# 5.3 启动 ViLidar

```java
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);

ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

slamManager.openViLidar();
```

---

# 5.4 拍照与图片预处理

```java
slamManager.preProcessPicture(gpsTime);
```

监听结果：

```java
@Override
public void onPreProcessPicResult(byte[] img) {
    pictureView.setBitmapByte(img);
}
```

---

# 5.5 获取像素点

```java
PointF pt = pictureView.getCurrentPointInfo();
```

---

# 5.6 像素点转坐标

```java
slamManager.calcCoordinateByPixel(gpsTime, (int)pt.x, (int)pt.y);
```

监听结果：

```java
@Override
public void onPixelToCoordinateResult(CoordinateInfo info) {
    double lat = info.getLat();
    double lon = info.getLon();
    double h   = info.getHeight();
}
```

---

# 6. 资源清理

```java
slamManager.removeListener(slamListener);
PositionSource.getInstance().removeListener(positionListener);
```

---

# 7. 总结

| 模式 | 场景 | 是否需要相机 | 是否支持像素点转坐标 |
|------|------|--------------|------------------------|
| **SFix** | 常规 GNSS+激光雷达定位 | 否 | 否 |
| **ViLidar** | 视觉定位，需预览/拍照 | 是 | 是 |
