# SLAM集成

## SFix

SFix是基于SLAM（Simultaneous Localization and Mapping）技术的实时定位解决方案，结合GNSS和激光雷达数据，为用户提供高精度的位置信息。本文档详细介绍了如何在Android应用中集成和使用SFix功能。

## 核心组件介绍

### 1. SlamDeviceManager

SFix功能的核心管理类，负责：

- SLAM设备的初始化和控制
- 激光雷达的开启/关闭/暂停
- 状态监听和回调

### 2. ISlamDeviceListener

SLAM设备状态监听接口，提供以下回调：

- `onSceneStatus()`: SLAM场景操作状态变化（开启/暂停/关闭成功或失败）
- `onInitStatus()`: SLAM设备初始化状态变化
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
    
    // 添加操作类型枚举，用于区分不同的SLAM操作
    private enum SlamOperation {
        NONE,
        OPENING,
        PAUSING,
        CLOSING
    }

    private SlamOperation mCurrentOperation = SlamOperation.NONE;
    
    private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
            runOnUiThread(() -> {
                // 检查操作是否成功
                if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
                    // SLAM开启成功
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        // 启动引导界面
                        SlamGuideActivity.start(SFixActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.PAUSE)) {
                    // SLAM暂停成功
                    handleSlamPaused();
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.CLOSED)) {
                    // SLAM关闭成功
                    handleSlamClosed();
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
                    // SLAM操作失败
                    String errorMessage = slamSceneStatus.getSlamSceneErrorCode().getMessage();
                    handleSlamError(errorMessage);
                }
                // 重置操作状态
                mCurrentOperation = SlamOperation.NONE;
            });
        }

        @Override
        public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
            runOnUiThread(() -> {
                // 处理初始化状态变化
                updateSlamInitStatus(slamInitStatus);
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
        // 添加监听器
        PositionSource.getInstance().addListener(positionListener);
        SlamDeviceManager.getInstance().addListener(slamDeviceListener);
    }
}
```

### 2. 启动SFix

```java
private void startSFix() {
    try {
        // 设置操作类型为开启
        mCurrentOperation = SlamOperation.OPENING;
        
        // 1. 启动惯性导航
        NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8); // 设置天线高度（米）
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ); // 设置数据频率
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);
        
        // 2. 开启SFix
        SlamDeviceManager.getInstance().openSfix();
        
    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "启动SFix失败", e);
        showErrorMessage("启动失败，请检查设备连接");
    }
}
```

### 3. 暂停SFix

```java
private void pauseSFix() {
    try {
        // 设置操作类型为暂停
        mCurrentOperation = SlamOperation.PAUSING;
        SlamDeviceManager.getInstance().pauseSlam();
    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "暂停SFix失败", e);
    }
}
```

### 4. 关闭SFix

```java
private void stopSFix() {
    try {
        // 设置操作类型为关闭
        mCurrentOperation = SlamOperation.CLOSING;
        SlamDeviceManager.getInstance().close();
    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "关闭SFix失败", e);
    }
}
```

## 详细API说明

### SlamDeviceManager API

#### 核心功能方法

```java
// 开启SFix模式
public void openSfix() throws RemoteException

// 暂停SLAM功能
public void pauseSlam() throws RemoteException

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
     * SLAM场景操作状态回调
     * @param slamSceneStatus 场景状态信息，包含操作结果和错误码
     */
    void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException;
    
    /**
     * SLAM设备初始化状态变化回调
     * @param slamInitStatus 初始化状态信息
     */
    void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException;
    
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

### SlamSceneStatus 场景状态信息

SlamSceneStatus包含以下主要状态：

- `OPENED`: SLAM开启成功
- `PAUSE`: SLAM暂停成功
- `CLOSED`: SLAM关闭成功
- `FAIL`: SLAM操作失败

**使用示例：**

```java
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
    EnumSlamSceneStatus status = slamSceneStatus.getSlamSceneStatus();
    switch (status) {
        case OPENED:
            // 处理开启成功
            break;
        case PAUSE:
            // 处理暂停成功
            break;
        case CLOSED:
            // 处理关闭成功
            break;
        case FAIL:
            // 处理操作失败
            EnumSlamSceneErrorCode errorCode = slamSceneStatus.getSlamSceneErrorCode();
            String errorMessage = errorCode.getMessage();
            break;
    }
}
```

### SlamInitStatus 初始化状态信息

SlamInitStatus包含以下主要状态：

- `SYSTEM_STATUS_UNKNOWN(-1)`: 未知状态，SLAM状态更新中
- `SYSTEM_STATUS_INITIAL(0)`: 初始化中，需要保持静止
- `SYSTEM_STATUS_INITIAL_NOT_FIX(1)`: 初始化中，RTK未固定，需要移动到空旷区域
- `SYSTEM_STATUS_GNSS_INITIAL(2)`: GNSS初始化完成，需要按提示移动完成SLAM初始化
- `SYSTEM_STATUS_SLAM_RUN(3)`: SLAM运行中，初始化完成，可以开始测量
- `SYSTEM_STATUS_SLAM_RUN_LOW(4)`: SLAM运行中，精度变差
- `SYSTEM_STATUS_SLAM_RUN_RTK_NOT_FIX(5)`: SLAM运行中，RTK精度差或未固定
- `SYSTEM_STATUS_SLAM_STOP(6)`: SLAM停止
- `SYSTEM_STATUS_SLAM_SAVING(7)`: SLAM保存中
- `SYSTEM_STATUS_SLAM_ERROR(8)`: SLAM设备错误，需要重新初始化
- `SYSTEM_STATUS_SLAM_SYSTEM_INIT(9)`: SLAM系统初始化
- `SYSTEM_STATUS_INITIAL_RTK_LESS(10)`: 初始化过程中，RTK信号弱
- `SYSTEM_STATUS_SLAM_RUN_RTK_LESS(11)`: 运行过程中，RTK信号弱

**使用示例：**

```java
@Override
public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
    EnumSlamInitStatus status = slamInitStatus.getSlamInitStatus();
    switch (status) {
        case SYSTEM_STATUS_INITIAL:
            showMessage("请保持静止，正在初始化...");
            break;
        case SYSTEM_STATUS_INITIAL_NOT_FIX:
            showMessage("请移动到空旷区域，RTK未固定");
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            showMessage("请按提示移动完成SLAM初始化");
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            showMessage("初始化完成，可以开始测量");
            break;
        case SYSTEM_STATUS_SLAM_ERROR:
            showMessage("SLAM设备错误，请重新初始化");
            break;
        // ... 其他状态处理
    }
}
```

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

**重要提示**: `SlamInitGuideViewLayout`是business模块提供的可选UI组件。您可以选择：

1. **使用我们提供的UI组件**：直接集成business模块，使用现成的引导界面
2. **自定义UI实现**：根据`ISlamDeviceListener`回调的状态信息，实现自己的用户界面

#### 选项1：使用提供的UI组件

##### 方式1：在Activity中直接使用

```java
private SlamInitGuideViewLayout mSlamInitGuideViewLayout;

private void initViews() {
    mSlamInitGuideViewLayout = findViewById(R.id.viewSlamInit);
}

private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
    @Override
    public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
        runOnUiThread(() -> {
            // 自动更新UI状态
            if (mSlamInitGuideViewLayout != null) {
                mSlamInitGuideViewLayout.updateStatus(slamInitStatus);
            }
        });
    }
    // ... 其他回调方法
};
```

##### 方式2：使用独立的引导弹窗Activity

```java
/**
 * SLAM 初始化引导弹窗 Activity
 * 以弹窗形式显示 SlamInitGuideViewLayout
 */
public class SlamGuideActivity extends AppCompatActivity {
    
    private SlamInitGuideViewLayout mSlamInitGuideViewLayout;
    
    private final ISlamDeviceListener mSlamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
            // 不需要处理场景状态
        }

        @Override
        public void onInitStatus(SlamInitStatus slamInitStatus) {
            runOnUiThread(() -> {
                // 更新引导界面状态
                mSlamInitGuideViewLayout.updateStatus(slamInitStatus);
                
                // 如果SLAM运行成功，自动关闭引导界面
                if (slamInitStatus.getSlamInitStatus() == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN) {
                    new Handler().postDelayed(() -> finish(), 200);
                }
            });
        }

        @Override
        public void onBatteryInfoChanged(int remainingPower) {
            // 引导界面不需要处理电池信息
        }

        @Override
        public void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) {
            // 引导界面不需要处理存储信息
        }
    };

    /**
     * 启动 SLAM 引导弹窗
     */
    public static void start(Context context) {
        Intent intent = new Intent(context, SlamGuideActivity.class);
        context.startActivity(intent);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setupDialogStyle(); // 设置弹窗样式
        setContentView(R.layout.activity_slam_guide);
        
        initView();
        initListener();
    }

    private void initListener() {
        // 注册 SLAM 设备监听器
        SlamDeviceManager.getInstance().addListener(mSlamDeviceListener);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 移除监听器
        SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
    }
}
```

#### 选项2：自定义UI实现

如果选择自定义UI实现，可以根据状态回调自行处理：

```java
private void updateCustomUI(SlamInitStatus slamInitStatus) {
    EnumSlamInitStatus status = slamInitStatus.getSlamInitStatus();
    switch (status) {
        case SYSTEM_STATUS_INITIAL:
            showCustomGuidance("请保持静止", R.drawable.keep_still_animation);
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            showCustomGuidance("请按提示移动", R.drawable.move_guidance_animation);
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            hideGuidanceAndStartSurvey();
            break;
        // ... 其他状态处理
    }
}
```

## 完整集成示例

### 主Activity完整代码示例

```java
public class SFixActivity extends AppCompatActivity {
    private static final String TAG = "SFixActivity";

    private enum SlamOperation {
        NONE, OPENING, PAUSING, CLOSING
    }

    private SlamOperation mCurrentOperation = SlamOperation.NONE;
    private TextView mTvSlamInitState;
    private TextView mTvBatteryLevel;

    private ISlamDeviceListener mSlamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
            runOnUiThread(() -> {
                if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
                    // 只有在开启操作时才启动引导弹窗
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        SlamGuideActivity.start(SFixActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
                    String errorMessage = slamSceneStatus.getSlamSceneErrorCode().getMessage();
                    Toast.makeText(SFixActivity.this, "操作失败: " + errorMessage, Toast.LENGTH_SHORT).show();
                }
                mCurrentOperation = SlamOperation.NONE;
            });
        }

        @Override
        public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
            runOnUiThread(() -> {
                mTvSlamInitState.setText(slamInitStatus.getSlamInitStatus().name());
            });
        }

        @Override
        public void onBatteryInfoChanged(int remainingPower) throws RemoteException {
            runOnUiThread(() -> {
                mTvBatteryLevel.setText(remainingPower + "%");
            });
        }

        @Override
        public void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) throws RemoteException {
            // 处理存储信息
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_sfix);
        
        initViews();
        initListeners();
    }

    private void initViews() {
        mTvSlamInitState = findViewById(R.id.tvSlamInitStatus);
        mTvBatteryLevel = findViewById(R.id.tvBatteryLevel);
        
        findViewById(R.id.btnStartSFix).setOnClickListener(v -> startSFix());
        findViewById(R.id.btnPauseSFix).setOnClickListener(v -> pauseSFix());
        findViewById(R.id.btnStopSFix).setOnClickListener(v -> stopSFix());
    }

    private void initListeners() {
        try {
            PositionSource.getInstance().addListener(positionListener);
            SlamDeviceManager.getInstance().addListener(mSlamDeviceListener);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to add listeners", e);
        }
    }

    private void startSFix() {
        try {
            mCurrentOperation = SlamOperation.OPENING;
            
            // 启动惯性导航
            NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
            startInfo.setAntennaHeight(1.8);
            startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
            ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);

            // 开启SFix
            SlamDeviceManager.getInstance().openSfix();
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to start SFix", e);
        }
    }

    private void pauseSFix() {
        try {
            mCurrentOperation = SlamOperation.PAUSING;
            SlamDeviceManager.getInstance().pauseSlam();
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to pause SFix", e);
        }
    }

    private void stopSFix() {
        try {
            mCurrentOperation = SlamOperation.CLOSING;
            SlamDeviceManager.getInstance().close();
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to stop SFix", e);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        try {
            PositionSource.getInstance().removeListener(positionListener);
            SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
        } catch (Exception e) {
            Log.e(TAG, "Error clearing listeners in onDestroy", e);
        }
    }
}
```

## 常见问题

### Q1: 如何区分不同的SLAM操作结果？

**A**: 使用操作状态枚举来跟踪当前操作：

```java
private enum SlamOperation {
    NONE, OPENING, PAUSING, CLOSING
}

private SlamOperation mCurrentOperation = SlamOperation.NONE;

// 在执行操作前设置状态
mCurrentOperation = SlamOperation.OPENING;
SlamDeviceManager.getInstance().openSfix();

// 在回调中根据状态处理结果
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
    if (mCurrentOperation == SlamOperation.OPENING && 
        slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
        // 处理开启成功
    }
    mCurrentOperation = SlamOperation.NONE;
}
```

### Q2: SLAM初始化状态过多，如何简化处理？

**A**: 可以将状态分组处理：

```java
private void handleSlamInitStatus(SlamInitStatus slamInitStatus) {
    EnumSlamInitStatus status = slamInitStatus.getSlamInitStatus();
    
    if (isInitializingStatus(status)) {
        showInitializingUI(status);
    } else if (isRunningStatus(status)) {
        showRunningUI(status);
    } else if (isErrorStatus(status)) {
        showErrorUI(status);
    }
}

private boolean isInitializingStatus(EnumSlamInitStatus status) {
    return status == EnumSlamInitStatus.SYSTEM_STATUS_INITIAL ||
           status == EnumSlamInitStatus.SYSTEM_STATUS_INITIAL_NOT_FIX ||
           status == EnumSlamInitStatus.SYSTEM_STATUS_GNSS_INITIAL;
}

private boolean isRunningStatus(EnumSlamInitStatus status) {
    return status == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN ||
           status == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN_LOW;
}

private boolean isErrorStatus(EnumSlamInitStatus status) {
    return status == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_ERROR;
}
```

### Q3: 如何处理SLAM操作失败？

**A**: 检查SlamSceneStatus中的错误码：

```java
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
    if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
        EnumSlamSceneErrorCode errorCode = slamSceneStatus.getSlamSceneErrorCode();
        String errorMessage = errorCode.getMessage();
        
        // 根据错误码进行相应处理
        handleSlamError(errorCode, errorMessage);
    }
}
```

### Q4: 引导界面何时自动关闭？

**A**: 当SLAM状态变为`SYSTEM_STATUS_SLAM_RUN`时自动关闭：

```java
@Override
public void onInitStatus(SlamInitStatus slamInitStatus) {
    if (slamInitStatus.getSlamInitStatus() == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN) {
        // 延迟关闭，让用户看到完成状态
        new Handler().postDelayed(() -> finish(), 200);
    }
}
```

### Q5: 内存泄漏问题

**A**: 确保在Activity销毁时正确清理：

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    try {
        // 移除所有监听器
        SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
        PositionSource.getInstance().removeListener(mPositionListener);
    } catch (Exception e) {
        Log.e(TAG, "Error clearing listeners", e);
    }
}
```

## 总结

新版SLAM API的主要变化：

1. **回调接口变化**：
    - `onDeviceInitStatusChanged()` → `onInitStatus()`
    - 新增 `onSceneStatus()` 回调用于处理操作结果

2. **数据结构变化**：
    - `SlamStatusInfo` → `SlamInitStatus` + `SlamSceneStatus`
    - 更详细的状态枚举和错误码

3. **新增功能**：
    - `pauseSlam()` 方法支持暂停SLAM
    - 操作状态跟踪，区分不同操作的回调

4. **UI组件使用**：
    - 支持独立弹窗形式的引导界面
    - 自动状态管理和界面关闭

这些变化使得SLAM集成更加灵活和可控，开发者可以更精确地处理各种SLAM操作状态。
