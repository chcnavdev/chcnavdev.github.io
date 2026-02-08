# SLAM 集成（精简版）

## SFix 概述

SFix 是基于 SLAM（Simultaneous Localization and Mapping）技术的实时定位系统，通过 **GNSS + 激光雷达** 融合为 Android 应用提供高精度位置。本文介绍如何在 Android 中快速集成与使用 SFix。

---

# 核心组件

### 1. SlamDeviceManager（SLAM 管理器）

负责 SLAM 的生命周期：

- `openSfix()`：开启 SLAM  
- `pauseSlam()`：暂停 SLAM  
- `close()`：关闭 SLAM  
- `addListener()` / `removeListener()`：监听器管理

---

### 2. ISlamDeviceListener（SLAM 状态回调）

关键回调：

| 回调方法                 | 说明                     |
| ------------------------ | ------------------------ |
| `onSceneStatus()`        | SLAM 开启/暂停/关闭 结果 |
| `onInitStatus()`         | 初始化流程状态           |
| `onBatteryInfoChanged()` | 电量变化                 |
| `onStorageInfoChanged()` | 存储变化                 |

---

### 3. PositionSource（位置源）

- 获取实时坐标（经纬高）
- 获取解算状态（FIX/FLOAT 等）

---

### 4. SlamInitGuideViewLayout（可选）

SLAM 初始化引导 UI，自动根据 `onInitStatus()` 更新操作提示。

---

# 快速开始

## 1. 初始化与监听器注册

```java
private void initSFix() throws RemoteException {
    PositionSource.getInstance().addListener(positionListener);
    SlamDeviceManager.getInstance().addListener(slamDeviceListener);
}
```

---

## 2. 启动 SFix

```java
private void startSFix() {
    try {
        // 启动惯性导航
        NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8);
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);

        // 启动 SFix
        SlamDeviceManager.getInstance().openSfix();
    } catch (Exception e) {
        Log.e("SFix", "启动失败", e);
    }
}
```

---

## 3. 暂停 / 关闭 SFix

```java
private void pauseSFix() {
    SlamDeviceManager.getInstance().pauseSlam();
}

private void stopSFix() {
    SlamDeviceManager.getInstance().close();
}
```

---

# ISlamDeviceListener 回调

```java
private final ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
    @Override
    public void onSceneStatus(SlamSceneStatus status) {
        EnumSlamSceneStatus s = status.getSlamSceneStatus();

        if (s == EnumSlamSceneStatus.OPENED) {
            SlamGuideActivity.start(SFixActivity.this);
        } else if (s == EnumSlamSceneStatus.FAIL) {
            Toast.makeText(SFixActivity.this,
                "操作失败：" + status.getSlamSceneErrorCode().getMessage(),
                Toast.LENGTH_SHORT).show();
        }
    }
    
    @Override
    public void onInitStatus(SlamInitStatus initStatus) {}
    
    @Override public void onBatteryInfoChanged(int power) {}
    
    @Override public void onStorageInfoChanged(SlamDeviceStorageInfo info) {}
};
```

---

# 位置监听

```java
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo info) {
        Position pos = info.getCoorKernal().getWgsBlh();
        double lat = pos.getX();
        double lon = pos.getY();
        double h = pos.getZ();
    }
});
```

---

# UI：自定义 or 内置引导

## 内置引导组件（自动）

```java
@Override
public void onInitStatus(SlamInitStatus status) {
    mGuideView.updateStatus(status);
}
```

---

## 自定义 UI 示例

```java
private void updateCustomUI(EnumSlamInitStatus status) {
    switch (status) {
        case SYSTEM_STATUS_INITIAL:
            showMessage("请保持静止");
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            showMessage("按提示移动");
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            hideGuide();
            break;
    }
}
```

---

# 清理资源

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    SlamDeviceManager.getInstance().removeListener(slamDeviceListener);
    PositionSource.getInstance().removeListener(positionListener);
}
```