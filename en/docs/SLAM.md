# SLAM Integration

This development document guides third-party developers on how to integrate SLAM functionality into Android applications.

The SLAM system includes three modes:

- **SFix**: GNSS + LiDAR fusion positioning
- **ViLiDar**: Based on SFix, adds camera, image capture, and pixel-to-coordinate transformation capabilities
- **Point Cloud**: Provides point cloud stream port and project path query

All modes are controlled through the SLAM management interface `ISlamDeviceManager`, which is used to start, pause, and stop SLAM operations.



## Obtaining ISlamDeviceManager

SLAM functionality is exposed to the application layer through `ISlamDeviceManager`, so developers must obtain an `ISlamDeviceManager` instance from the GNSS service.

Example code:

```java
// Get GNSS service
IGnssServiceBinder service = GnssServiceManager.getInstance().getService();
if (service == null) {
    // Service not connected
    return;
}

// Get the SLAM device manager from GNSS service
ISlamDeviceManager slamManager =
        ISlamDeviceManager.Stub.asInterface(service.getSlamDeviceManager());

// slamManager can now be used to call all SLAM APIs
```

> **Note:** All SLAM operations are executed via `ISlamDeviceManager`.



## SLAM Basic Interfaces

`ISlamDeviceManager` provides the core control interfaces for SLAM:

| Method | Description |
||--|
| `openSfix()` | Start SFix mode |
| `recoverySfix()` | Resume SFix after pause |
| `openViLidar()` | Start ViLiDar mode |
| `recoveryViLidar()` | Resume ViLiDar after pause |
| `openPointCloud()` | Start Point Cloud mode|
| `queryProjectInfo(projectId)` | Query project remote path and packaging progress |
| `pauseSlam()` | Pause SFix/ViLiDar |
| `close()` | Stop SFix/ViLiDar/Point Cloud |
| `addListener(ISlamDeviceListener)` | Register listener |
| `removeListener(ISlamDeviceListener)` | Remove listener |
| `preProcessPicture(gpsTime)` | (ViLiDar) Frame preprocessing |
| `calcCoordinateByPixel(gpsTime,x,y)` | (ViLiDar) Pixel-to-coordinate conversion |



## SLAM State Listening (ISlamDeviceListener)

Developers must register a listener to receive SLAM events:

```java
slamManager.addListener(new ISlamDeviceListener.Stub() {

    @Override
    public void onSceneStatus(SlamSceneStatus status) {
        // SLAM start/pause/stop result
    }

    @Override
    public void onProjectInfoStatus(SlamProjectInfo slamProjectInfo) {
        // Project info query result (used by Point Cloud)
    }

    @Override
    public void onInitStatus(SlamInitStatus status) {
        // Initialization state changes
    }
    
    @Override
    public void onPreProcessPicResult(SlamPreProcessPictureStatus slamPreProcessPictureStatus) {
        // ViLiDar picture preprocessing result
    }
    
    @Override
    public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus slamSampleCoordinateResultStatus) {
        // ViLiDar pixel-to-coordinate result
    }
    
    @Override
    public void onBatteryInfoChanged(BatteryLevelDetail batteryLevelDetail) {}
    
    @Override
    public void onStorageInfoChanged(SlamDeviceStorageInfo storage) {}
});
```



## SLAM Initialization Flow Description (General)

Developers can use `onInitStatus()` to track each stage of the SLAM initialization process to determine:

- Whether to show initialization UI  
- Whether the user can start measurement  
- Whether to prompt the user to move or keep still  

Example:

```java
@Override
public void onInitStatus(SlamInitStatus status) {
    switch (status.getSlamInitStatus()) {
        case SYSTEM_STATUS_INITIAL:
            // Need to keep still
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            // Move as instructed
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            // Initialization complete, ready for measurement
            break;
    }
}
```



## UI Components

The SDK provides several optional UI components that developers may use directly.

### SlamInitGuideViewLayout (Optional Initialization Guidance UI)

Used to display SLAM initialization steps such as “keep still,” “move device,” “initialization complete,” etc.

Example:

```java
@Override
public void onInitStatus(SlamInitStatus status) {
    if (initGuideView != null) {
        initGuideView.updateStatus(status);
    }
}
```



### ViLiDarPictureViewLayout (ViLiDar Image Display + Pixel Selection)

Used to display preprocessed images and allow users to select pixel points.

```java
pictureView.setBitmapByte(imgBytes);
PointF point = pictureView.getCurrentPointInfo();
```



### SlamVideoFrameDrafter (Camera Real-time Frame Rendering)

Renders real-time camera frames to a specified view.

```java
SlamVideoFrameDrafter drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());
```

Pause rendering:

```java
drafter.setStopDrafter(true);
```



## SFix Integration Guide

SFix = **GNSS + LiDAR** fusion positioning



### Starting SFix

```java
// 1. Start inertial navigation
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

// 2. Start SFix
slamManager.openSfix();
```



### Pause / Resume / Stop

```java
slamManager.pauseSlam();      // Pause
slamManager.recoverySfix();  // Resume
slamManager.close();         // Stop
```



### Getting Real-time Position (PositionSource)

```java
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo info) {
        Position pos = info.getCoorKernal().getWgsBlh();
        double lat = pos.getX(); // Latitude
        double lon = pos.getY(); // Longitude
        double h   = pos.getZ(); // Height
    }
});
```



## ViLiDar Integration Guide

ViLiDar extends SFix by adding camera features, enabling **pixel → geographic coordinate** conversion.



### Overall Flow

1. Register SLAM listener  
2. Open camera  
3. Start inertial navigation  
4. `openViLidar()`  
5. Call `preProcessPicture()` to capture a static frame  
6. User taps a pixel point  
7. Call `calcCoordinateByPixel()` to convert coordinates  



### Open Camera

```java
SlamVideoFrameDrafter drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());

SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
camera.setCallback(drafter);
camera.open();
```



### Start ViLiDar

```java
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);

ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

slamManager.openViLidar();
```



### Capture & Preprocess Image

```java
slamManager.preProcessPicture(gpsTime);
```

Listening for result:

```java
@Override
public void onPreProcessPicResult(SlamPreProcessPictureStatus result) {
    // Retrieve static frame data
}
```



### Get Pixel Point

```java
PointF pt = pictureView.getCurrentPointInfo();
```



### Pixel-to-Coordinate Conversion

```java
slamManager.calcCoordinateByPixel(gpsTime, x, y);
```

Listen for result:

```java
@Override
public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus result) {
    // Retrieve geographic coordinate result
}
```



## Point Cloud Integration Guide


### Device Capability Check

Before use, confirm that the device supports point cloud:

```java
ISlamDeviceInfoProvider provider = slamManager.getSlamDeviceInfoProvider();
if (provider != null && provider.isSupportPointCloud()) {
    // Point cloud supported
}
```



### Starting Point Cloud

```java
// 1. Start inertial navigation (same as SFix)
NoneMagneticTiltStartInfo info = new NoneMagneticTiltStartInfo();
info.setAntennaHeight(1.8);
info.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(context, info);

// 2. Start Point Cloud
slamManager.openPointCloud();
```



### Getting Point Cloud Stream Port and Project ID

In `onSceneStatus`, when the status is `OPENED`:

```java
@Override
public void onSceneStatus(SlamSceneStatus status) {
    if (status.getSlamSceneStatus() == EnumSlamSceneStatus.OPENED) {
        int port = slamManager.getSlamDeviceInfoProvider().getPointCloudStreamPort();
        long projectId = status.getProjectId();
        // Use port to connect to point cloud stream, use projectId to query project info
    }
}
```



### Query Project Remote Path

```java
slamManager.queryProjectInfo(projectId);
```

Get result via `onProjectInfoStatus`:

```java
@Override
public void onProjectInfoStatus(SlamProjectInfo info) {
    if (info.getEnumSlamProjectInfoStatus() == EnumSlamProjectInfoStatus.SUCCESS
            && info.getSlamProjectInfoStatus() != null) {
        SlamProjectInfoStatus status = info.getSlamProjectInfoStatus();
        String path = status.getProjectPath();   // SFTP remote path
        int progress = status.getProgress();     // Packaging progress 0–100
        long remainTime = status.getRemainTime(); // Estimated remaining time
    }
}
```



### Stopping Point Cloud

```java
slamManager.close();
```



### Notes

- Point Cloud mode **does not support** pause and resume, only `openPointCloud()` and `close()` are supported
- `SlamInitGuideViewLayout` can be used to display initialization guidance



## Resource Cleanup

```java
slamManager.removeListener(slamListener);
PositionSource.getInstance().removeListener(positionListener);
```
