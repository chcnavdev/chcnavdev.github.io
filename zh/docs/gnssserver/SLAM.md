# SLAM集成

## SFix

SFix是基于SLAM（Simultaneous Localization and Mapping）技术的实时定位解决方案，结合GNSS和激光雷达数据，为用户提供高精度的位置信息。本文档详细介绍了如何在Android应用中集成和使用SFix功能。

## 核心组件介绍

### 1. SlamDeviceManager

SFix功能的核心管理类，负责：

- SLAM设备的初始化和控制
- 激光雷达的开启/关闭
- 状态监听和回调

### 2. ISlamDeviceListener

SLAM设备状态监听接口，提供以下回调：

- `onDeviceInitStatusChanged()`: 设备初始化状态变化
- `onBatteryInfoChanged()`: 电量信息变化
- `onStorageInfoChanged()`: 存储空间信息变化

### 3. PositionSource

位置数据源，提供实时的坐标信息、解算状态

### 4. SlamInitGuideViewLayout（可选UI组件）

业务模块提供的SLAM初始化引导视图，包含：

- 动态GIF引导动画
- 状态提示文本
- 自动状态切换
- 完成后自动隐藏

## 快速开始

### 1. 基础集成

```java
public class SFixActivity extends AppCompatActivity {
    
    private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onDeviceInitStatusChanged(SlamStatusInfo slamStatusInfo) throws RemoteException {
            runOnUiThread(() -> {
                // 处理状态变化
                updateSlamStatus(slamStatusInfo);
            });
        }

        @Override
        public void onBatteryInfoChanged(int remainingPower) throws RemoteException {
            runOnUiThread(() -> {
                // 处理电量变化
                updateBatteryInfo(remainingPower);
            });
        }

        @Override
        public void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) throws RemoteException {
            runOnUiThread(() -> {
                // 处理存储信息变化
                updateStorageInfo(storageInfo);
            });
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sfix);
        
        try {
            initSFix();
        } catch (RemoteException e) {
            Log.e("SFix", "初始化失败", e);
            handleInitError(e);
        }
    }
    
    private void initSFix() throws RemoteException {
        // 初始化SlamDeviceManager
        SlamDeviceManager.init(this);
        SlamDeviceManager.getInstance().addListener(slamDeviceListener);
        
        // 初始化位置监听
        PositionSource.getInstance().addListener(positionListener);
    }
}
```

### 2. 启动SFix

```java
private void startSFix() {
    try {
        // 1. 启动惯性导航
        NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8); // 设置天线高度（米）
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ); // 设置数据频率
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);
        
        // 2. 开启SFix
        SlamDeviceManager.getInstance().openSfix();
        
    } catch (RemoteException e) {
        Log.e("SFix", "启动SFix失败", e);
        showErrorMessage("启动失败，请检查设备连接");
    }
}
```

### 3. 关闭SFix

```java
private void stopSFix() {
    try {
        SlamDeviceManager.getInstance().close();
    } catch (RemoteException e) {
        Log.e("SFix", "关闭SFix失败", e);
    }
}
```

## 详细API说明

### SlamDeviceManager API

#### 初始化方法

```java
// 单例初始化（必须在使用前调用）
public static void init(@NonNull Context context)

// 获取单例实例
public static SlamDeviceManager getInstance()
```

#### 核心功能方法

```java
// 开启SFix模式
public void openSfix() throws RemoteException

// 开启ViLidar模式（待实现）
public void openViLidar() throws RemoteException

// 关闭SLAM功能
public void close() throws RemoteException

// 添加状态监听器
public void addListener(ISlamDeviceListener listener) throws RemoteException

// 移除状态监听器
public void removeListener(ISlamDeviceListener listener) throws RemoteException
```

### ISlamDeviceListener 回调接口

```java
public interface ISlamDeviceListener {
    /**
     * SLAM设备初始化状态变化回调
     * @param slamStatusInfo 状态信息，包含状态码和提示消息
     */
    void onDeviceInitStatusChanged(SlamStatusInfo slamStatusInfo) throws RemoteException;
    
    /**
     * 电量信息变化回调
     * @param remainingPower 剩余电量百分比 (0-100，设备充电时值为120)
     */
    void onBatteryInfoChanged(int remainingPower) throws RemoteException;
    
    /**
     * 存储空间信息变化回调
     * @param storageInfo 存储信息对象
     */
    void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) throws RemoteException;
}
```

### SlamStatusInfo 状态信息

SlamStatusInfo包含以下主要状态：

- `SYSTEM_STATUS_UNKNOWN`: 未知状态
- `SYSTEM_STATUS_INITIAL`: 初始化中，需要保持静止
- `SYSTEM_STATUS_INITIAL_NOT_FIX`: 初始化中，RTK未固定，需要移动到空旷区域
- `SYSTEM_STATUS_GNSS_INITIAL`: GNSS初始化完成，需要按提示移动完成SLAM初始化
- `SYSTEM_STATUS_SLAM_RUN`: SLAM运行中，初始化完成
- `SYSTEM_STATUS_SLAM_ERROR`: SLAM设备错误

### PositionSource 位置数据

```java
// 添加位置监听器
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo positionInfo) {
        // 处理位置信息
        Position position = positionInfo.getCoorKernal().getWgsBlh();
        double latitude = position.getX();   // 纬度
        double longitude = position.getY();  // 经度
        double height = position.getZ();     // 高程
        
        // 获取解算状态
        int solveStatus = positionInfo.getSolveStatus().getValue();
    }
});
```

## UI组件使用

### SlamInitGuideViewLayout 使用说明

**重要提示**: `SlamInitGuideViewLayout`是business模块提供的可选UI组件。客户可以选择：

1. **使用我们提供的UI组件**：直接集成business模块，使用现成的引导界面
2. **自定义UI实现**：根据`ISlamDeviceListener`回调的状态信息，实现自己的用户界面

#### 选项1：使用提供的UI组件

如果选择使用我们提供的UI组件，需要：

1. **添加business模块依赖**：

```gradle
implementation 'com.huace.gnssserver:business:latest_version'
```

2. **在布局文件中添加组件**：

```xml
<com.huace.gnssserver.business.api.slam.SlamInitGuideViewLayout
    android:id="@+id/viewSlamInit"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:visibility="gone" />
```

3. **在代码中使用**：

```java
private SlamInitGuideViewLayout mSlamInitGuideViewLayout;

private void initViews() {
    mSlamInitGuideViewLayout = findViewById(R.id.viewSlamInit);
}

private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
    @Override
    public void onDeviceInitStatusChanged(SlamStatusInfo slamStatusInfo) throws RemoteException {
        runOnUiThread(() -> {
            // 自动更新UI状态
            if (mSlamInitGuideViewLayout != null) {
                mSlamInitGuideViewLayout.updateStatus(slamStatusInfo.getStatus());
            }
        });
    }
    // ... 其他回调方法
};
```

#### 选项2：自定义UI实现

如果选择自定义UI实现，可以根据状态回调自行处理：

```java
private void updateCustomUI(SlamStatusInfo slamStatusInfo) {
    // 根据状态自行处理对应的UI
}
```

### UI组件功能特性

SlamInitGuideViewLayout提供以下功能：

1. **自动状态切换**：根据SLAM状态自动更新UI显示
2. **动态引导动画**：不同状态显示对应的GIF动画
3. **多语言支持**：支持中英文切换
4. **自动隐藏**：初始化完成后自动隐藏界面
5. **手动控制**：支持手动显示/隐藏

## 常见问题

### Q1: 初始化报错

**A**: 确保在使用SlamDeviceManager之前调用了`SlamDeviceManager.init(context)`方法。

```java
// 正确的初始化顺序
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    
    // 1. 先初始化
    SlamDeviceManager.init(this);
    
    // 2. 再使用
    SlamDeviceManager.getInstance().addListener(listener);
}
```

### Q2: SFix启动后没有状态回调

**A**: 检查以下几点：

1. 确保正确添加了监听器
2. 确保设备支持SLAM功能
3. 检查设备连接状态
4. 查看日志是否有错误信息

### Q3: 位置信息不更新

**A**: 确保：

1. 已添加位置监听器
2. 设备已获取到GNSS信号
3. 检查权限是否正确授予
4. 设备已注册

### Q4: UI组件不显示

**A**: 如果使用SlamInitGuideViewLayout：

1. 确保已添加business模块依赖
2. 检查布局文件中的visibility属性

### Q5: 内存泄漏问题

**A**: 确保在Activity销毁时正确清理：

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    // 移除所有监听器
    SlamDeviceManager.getInstance().removeListener(slamDeviceListener);
    PositionSource.getInstance().removeListener(positionListener);
}
```