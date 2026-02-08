# SLAM集成

## SFix

SFix是基于SLAM（Simultaneous Localization and
Mapping）技术的实时定位解决方案，结合GNSS和激光雷达数据，为用户提供高精度的位置信息。本文档详细介绍了如何在Android应用中集成和使用SFix功能。

### 核心组件介绍

#### 1. SlamDeviceManager

SFix功能的核心管理类，负责：

- SLAM设备的初始化和控制
- SLAM设备的开启/关闭/暂停
- 状态监听

#### 2. ISlamDeviceListener

SLAM设备状态监听接口，提供以下回调：

- `onSceneStatus()`: SLAM场景操作状态变化（开启/暂停/关闭成功或失败）
- `onInitStatus()`: SLAM设备初始化状态变化
- `onBatteryInfoChanged()`: 电量信息变化
- `onStorageInfoChanged()`: 存储空间信息变化

#### 3. PositionSource

位置数据源，提供实时的坐标信息、解算状态

#### 4. SlamInitGuideViewLayout（可选UI组件）

根据状态提供Slam初始化引导视图

### 快速开始

#### 1. 基础集成

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

#### 2. 启动SFix

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

#### 3. 暂停SFix

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

#### 4. 关闭SFix

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

### 详细API说明

#### SlamDeviceManager API

##### 核心功能方法

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

#### ISlamDeviceListener 回调接口

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

#### SlamSceneStatus 场景状态信息

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
        case default:
            // 处理默认
            break;
    }
}
```

#### SlamInitStatus 初始化状态信息

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

#### PositionSource 位置数据

```java
// 添加位置监听器
PositionSource.getInstance().

addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition (SurveyPointPositionInfo positionInfo){
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

### UI组件使用

#### SlamInitGuideViewLayout 使用说明

**重要提示**: `SlamInitGuideViewLayout`是business模块提供的可选UI组件。您可以选择：

1. **使用我们提供的UI组件**：直接集成business模块，使用现成的引导界面
2. **自定义UI实现**：根据`ISlamDeviceListener`回调的状态信息，实现自己的用户界面

##### 选项1：使用提供的UI组件

###### 方式1：在Activity中直接使用

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

###### 方式2：使用独立的引导弹窗Activity

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

##### 选项2：自定义UI实现

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

### 完整集成示例

#### 主Activity完整代码示例

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

### 常见问题

#### Q1: 如何区分不同的SLAM操作结果？

**A**: 使用操作状态枚举来跟踪当前操作：

```java
private enum SlamOperation {
    NONE, OPENING, PAUSING, CLOSING
}

private SlamOperation mCurrentOperation = SlamOperation.NONE;

// 在执行操作前设置状态
mCurrentOperation =SlamOperation.OPENING;
SlamDeviceManager.

getInstance().

openSfix();

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

#### Q2: SLAM初始化状态过多，如何简化处理？

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

#### Q3: 如何处理SLAM操作失败？

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

#### Q4: 引导界面何时自动关闭？

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

#### Q5: 内存泄漏问题

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

## ViLidar

ViLidar是基于SLAM（Simultaneous Localization and
Mapping）技术的视觉激光雷达定位解决方案，结合GNSS、激光雷达和相机数据，为用户提供高精度的位置信息。与SFix相比，ViLidar增加了相机视频流、拍照和像素点转坐标功能，可以实现通过点击图片上的像素点获取实际地理坐标。

本文档详细介绍了如何在Android应用中集成和使用ViLidar功能。

### 核心组件介绍

#### 1. SlamDeviceManager

ViLidar功能的核心管理类，负责：

- SLAM设备的初始化和控制
- ViLidar模式的开启/关闭/暂停
- 图片预处理和像素点转坐标
- 状态监听

#### 2. ISlamDeviceListener

SLAM设备状态监听接口，提供以下回调：

- `onSceneStatus()`: SLAM场景操作状态变化（开启/暂停/关闭成功或失败）
- `onInitStatus()`: SLAM设备初始化状态变化
- `onPreProcessPicResult()`: 图片预处理结果（ViLidar特有）
- `onPixelToCoordinateResult()`: 像素点转坐标结果（ViLidar特有）
- `onBatteryInfoChanged()`: 电量信息变化
- `onStorageInfoChanged()`: 存储空间信息变化

#### 3. SlamCamera

SLAM相机管理类，负责：

- 相机初始化和控制
- 视频帧数据接收
- 相机状态管理

#### 4. SlamVideoFrameDrafter

视频帧绘制器，负责：

- 视频帧的绘制和显示
- 视频帧数据的接收和处理
- 相机状态同步

#### 5. ViLidarPictureViewLayout（可选UI组件）

图片选择点视图组件，用于：

- 显示拍照后的图片
- 支持用户点击选择像素点
- 获取选中点的像素坐标

#### 6. PositionSource

位置数据源，提供实时的坐标信息、解算状态

#### 7. SlamInitGuideViewLayout（可选UI组件）

根据状态提供Slam初始化引导视图

### 快速开始

#### 1. 基础集成

```java
public class ViLidarActivity extends AppCompatActivity {

    // 添加操作类型枚举，用于区分不同的SLAM操作
    private enum SlamOperation {
        NONE,
        OPENING,
        PAUSING,
        CLOSING
    }

    private SlamOperation mCurrentOperation = SlamOperation.NONE;

    // 视频流相关组件
    private RtkImageView mRtkImageView;
    private SlamVideoFrameDrafter<IImageDrafterContext> mDrafter;
    private ViLidarPictureViewLayout mViLidarPictureViewLayout;

    private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
            runOnUiThread(() -> {
                // 检查操作是否成功
                if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
                    // ViLidar开启成功
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        // 启动引导界面
                        SlamGuideActivity.start(ViLidarActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.PAUSE)) {
                    // ViLidar暂停成功
                    handleSlamPaused();
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.CLOSED)) {
                    // ViLidar关闭成功
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
        public void onPreProcessPicResult(SlamPreProcessPictureStatus status) throws RemoteException {
            runOnUiThread(() -> {
                // 处理图片预处理结果
                if (status.getPointToCoordinateStatus() != EnumSlamPrePictureStatus.FAIL) {
                    // 预处理成功，显示图片选择界面
                    showPicSelectPoint();
                } else {
                    // 预处理失败
                    handlePreProcessError();
                }
            });
        }

        @Override
        public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus resultStatus)
                throws RemoteException {
            runOnUiThread(() -> {
                // 处理像素点转坐标结果
                double longitude = resultStatus.getLon();  // 经度
                double latitude = resultStatus.getLat();  // 纬度
                double altitude = resultStatus.getAlt();  // 高程
                showCoordinateResult(longitude, latitude, altitude);
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
        setContentView(R.layout.activity_vilidar);

        try {
            initViLidar();
        } catch (RemoteException e) {
            Log.e("ViLidar", "初始化失败", e);
            handleInitError(e);
        }
    }

    private void initViLidar() throws RemoteException {
        // 初始化视频流视图
        initVideoView();

        // 添加监听器
        PositionSource.getInstance().addListener(positionListener);
        SlamDeviceManager.getInstance().addListener(slamDeviceListener);
    }

    /**
     * 初始化视频流视图
     * 创建视频帧绘制器并设置到RtkImageView
     */
    private void initVideoView() {
        mRtkImageView = findViewById(R.id.viewSlam);
        mViLidarPictureViewLayout = findViewById(R.id.viLidarPicture);

        // 创建视频帧绘制器
        mDrafter = new SlamVideoFrameDrafter<>(mRtkImageView, new ImageSurveyVideoLayer());
        mRtkImageView.setVideoDrafter(mDrafter);
    }
}
```

#### 2. 启动ViLidar

```java
private void startViLidar() {
    try {
        // 设置操作类型为开启
        mCurrentOperation = SlamOperation.OPENING;

        // 1. 打开相机
        SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
        camera.setCallback(mDrafter);
        mDrafter.setFrameReceived(new OnVideoFrameReceived() {
            @Override
            public void onDataReceived(@NonNull VideoFrame vf) {
                // 保存当前视频帧，用于后续拍照和坐标转换
                mCurrentVideoFrame = vf;
            }

            @Override
            public void onError(int error) {
                Log.e("ViLidar", "视频帧接收错误: " + error);
            }
        });
        camera.open();

        // 2. 显示视频流视图
        mRtkImageView.setVisibility(View.VISIBLE);
        mViLidarPictureViewLayout.setVisibility(View.GONE);

        // 3. 启动惯性导航
        NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8); // 设置天线高度（米）
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ); // 设置数据频率
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);

        // 4. 开启ViLidar
        SlamDeviceManager.getInstance().openViLidar();

    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("ViLidar", "启动ViLidar失败", e);
        showErrorMessage("启动失败，请检查设备连接");
    }
}
```

#### 3. 暂停ViLidar

```java
private void pauseViLidar() {
    try {
        // 设置操作类型为暂停
        mCurrentOperation = SlamOperation.PAUSING;

        // 暂停视频绘制
        mDrafter.setStopDrafter(true);

        // 暂停SLAM
        SlamDeviceManager.getInstance().pauseSlam();
    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("ViLidar", "暂停ViLidar失败", e);
    }
}
```

#### 4. 恢复ViLidar

```java
private void recoverViLidar() {
    try {
        // 恢复视频绘制
        mDrafter.setStopDrafter(false);

        // 重新开启ViLidar（与启动流程相同）
        startViLidar();
    } catch (RemoteException e) {
        Log.e("ViLidar", "恢复ViLidar失败", e);
    }
}
```

#### 5. 关闭ViLidar

```java
private void stopViLidar() {
    try {
        // 设置操作类型为关闭
        mCurrentOperation = SlamOperation.CLOSING;

        // 关闭SLAM
        SlamDeviceManager.getInstance().close();

        // 关闭相机
        if (mCamera != null) {
            mCamera.close();
            mCamera.setCallback(null);
            mCamera = null;
        }

        // 隐藏视频流视图
        mRtkImageView.setVisibility(View.GONE);
        mViLidarPictureViewLayout.setVisibility(View.GONE);

    } catch (RemoteException e) {
        // 异常时重置操作状态
        mCurrentOperation = SlamOperation.NONE;
        Log.e("ViLidar", "关闭ViLidar失败", e);
    }
}
```

#### 6. 拍照功能

```java
private void takePicture() {
    // 获取当前视频帧的GPS时间
    if (mCurrentVideoFrame == null) {
        showErrorMessage("视频帧不可用，无法拍照");
        return;
    }

    // 计算GPS时间（秒）
    double gpsTime = mCurrentVideoFrame.getWeek() * 7 * 24 * 3600 +
            mCurrentVideoFrame.getSecOfWeek();

    // 触发图片预处理
    SlamDeviceManager.getInstance().preProcessPicture(gpsTime);

    // 图片预处理结果会在 onPreProcessPicResult 回调中返回
}

/**
 * 显示图片选择点界面
 * 在图片预处理成功后调用
 */
private void showPicSelectPoint() {
    // 暂停视频绘制
    mDrafter.setStopDrafter(true);

    // 隐藏视频流视图
    mRtkImageView.setVisibility(View.GONE);

    // 显示图片选择视图
    mViLidarPictureViewLayout.setVisibility(View.VISIBLE);

    // 设置当前视频帧的JPEG数据到图片视图
    if (mCurrentVideoFrame != null) {
        mViLidarPictureViewLayout.setBitmapByte(mCurrentVideoFrame.getJpegBuffer());
    }
}
```

#### 7. 像素点转坐标

```java
private void getCoordinateFromPixel() {
    // 获取用户在图片上选择的像素点坐标
    PointF pixelPoint = mViLidarPictureViewLayout.getCurrentPointInfo();
    if (pixelPoint == null) {
        showErrorMessage("请先在图片上选择点");
        return;
    }

    // 获取当前视频帧的GPS时间
    if (mCurrentVideoFrame == null) {
        showErrorMessage("视频帧不可用");
        return;
    }

    double gpsTime = mCurrentVideoFrame.getWeek() * 7 * 24 * 3600 +
            mCurrentVideoFrame.getSecOfWeek();

    // 将像素点转换为地理坐标
    int pixelX = (int) pixelPoint.x;
    int pixelY = (int) pixelPoint.y;
    SlamDeviceManager.getInstance().calcCoordinateByPixel(gpsTime, pixelX, pixelY);

    // 转换结果会在 onPixelToCoordinateResult 回调中返回
}
```

### 详细API说明

#### SlamDeviceManager API

##### 核心功能方法

```java
// 开启ViLidar模式
public void openViLidar() throws RemoteException

// 暂停SLAM功能
public void pauseSlam() throws RemoteException

// 关闭SLAM功能
public void close() throws RemoteException

// 图片预处理（在指定GPS时间点）
public void preProcessPicture(double gpsTime)

// 像素点转坐标（将图片上的像素点转换为地理坐标）
public void calcCoordinateByPixel(double gpsTime, int xPixel, int yPixel)

// 添加状态监听器
public void addListener(ISlamDeviceListener listener) throws RemoteException

// 移除状态监听器
public void removeListener(ISlamDeviceListener listener) throws RemoteException
```

#### ISlamDeviceListener 回调接口

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
     * 图片预处理结果回调（ViLidar特有）
     * @param slamPreProcessPictureStatus 图片预处理状态信息
     */
    void onPreProcessPicResult(SlamPreProcessPictureStatus slamPreProcessPictureStatus)
            throws RemoteException;

    /**
     * 像素点转坐标结果回调（ViLidar特有）
     * @param slamSampleCoordinateResultStatus 坐标转换结果，包含经纬度和高程
     */
    void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus slamSampleCoordinateResultStatus)
            throws RemoteException;

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

#### SlamSceneStatus 场景状态信息

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
        default:
            // 处理默认
            break;
    }
}
```

#### SlamInitStatus 初始化状态信息

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

#### SlamPreProcessPictureStatus 图片预处理状态

SlamPreProcessPictureStatus包含以下状态：

- `SUCCESS`: 图片预处理成功
- `FAIL`: 图片预处理失败

**使用示例：**

```java

@Override
public void onPreProcessPicResult(SlamPreProcessPictureStatus status) throws RemoteException {
    EnumSlamPrePictureStatus preStatus = status.getPointToCoordinateStatus();
    if (preStatus == EnumSlamPrePictureStatus.SUCCESS) {
        // 预处理成功，显示图片选择界面
        showPicSelectPoint();
    } else {
        // 预处理失败
        showErrorMessage("图片预处理失败，请重试");
    }
}
```

#### SlamSampleCoordinateResultStatus 像素点转坐标结果

SlamSampleCoordinateResultStatus包含以下信息：

- `getLon()`: 经度
- `getLat()`: 纬度
- `getAlt()`: 高程

**使用示例：**

```java

@Override
public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus resultStatus)
        throws RemoteException {
    double longitude = resultStatus.getLon();  // 经度
    double latitude = resultStatus.getLat();   // 纬度
    double altitude = resultStatus.getAlt();    // 高程

    // 显示坐标结果
    showCoordinateResult(longitude, latitude, altitude);
}
```

#### SlamCamera API

```java
// 创建相机实例
SlamCamera camera = new SlamCamera(SlamCameraType.FRONT_CAMERA);

// 设置视频帧绘制器回调
camera.

setCallback(drafter);

// 打开相机
camera.open();

// 关闭相机
        camera.close();

// 清除回调
        camera.setCallback(null);
```

#### SlamVideoFrameDrafter API

```java
// 创建视频帧绘制器
SlamVideoFrameDrafter<IImageDrafterContext> drafter =
        new SlamVideoFrameDrafter<>(rtkImageView, new ImageSurveyVideoLayer());

// 设置视频帧接收监听器
drafter.

setFrameReceived(new OnVideoFrameReceived() {
    @Override
    public void onDataReceived (@NonNull VideoFrame vf){
        // 处理视频帧
    }

    @Override
    public void onError ( int error){
        // 处理错误
    }
});

// 暂停/恢复视频绘制
        drafter.

setStopDrafter(true);  // 暂停
drafter.

setStopDrafter(false); // 恢复

// 更新相机状态
drafter.

onStatusChanged(SlamCameraStatus.UN_OPENED);
```

#### PositionSource 位置数据

```java
// 添加位置监听器
PositionSource.getInstance().

addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition (SurveyPointPositionInfo positionInfo){
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

### UI组件使用

#### ViLidarPictureViewLayout 使用说明

**重要提示**: `ViLidarPictureViewLayout`是business模块提供的可选UI组件，用于显示拍照后的图片并支持用户选择像素点。

##### 在布局文件中使用

```xml

<com.huace.gnssserver.business.api.slam.vilidar.ViLidarPictureViewLayout
    android:id="@+id/viLidarPicture" android:layout_width="match_parent"
    android:layout_height="match_parent" android:visibility="gone" />
```

##### 在代码中使用

```java
private ViLidarPictureViewLayout mViLidarPictureViewLayout;

private void initView() {
    mViLidarPictureViewLayout = findViewById(R.id.viLidarPicture);
}

/**
 * 显示图片选择点界面
 */
private void showPicSelectPoint() {
    // 设置图片数据（JPEG字节数组）
    if (mCurrentVideoFrame != null) {
        mViLidarPictureViewLayout.setBitmapByte(mCurrentVideoFrame.getJpegBuffer());
    }

    // 显示图片视图，隐藏视频流视图
    mViLidarPictureViewLayout.setVisibility(View.VISIBLE);
    mRtkImageView.setVisibility(View.GONE);
}

/**
 * 获取用户选择的像素点坐标
 */
private void getSelectedPixelPoint() {
    PointF pixelPoint = mViLidarPictureViewLayout.getCurrentPointInfo();
    if (pixelPoint != null) {
        int pixelX = (int) pixelPoint.x;
        int pixelY = (int) pixelPoint.y;
        // 使用像素坐标进行坐标转换
        convertPixelToCoordinate(pixelX, pixelY);
    }
}

/**
 * 取消拍照操作
 */
private void cancelTakePicture() {
    // 隐藏图片视图，显示视频流视图
    mViLidarPictureViewLayout.setVisibility(View.GONE);
    mRtkImageView.setVisibility(View.VISIBLE);

    // 恢复视频绘制
    mDrafter.setStopDrafter(false);
}
```

#### SlamInitGuideViewLayout 使用说明

**重要提示**: `SlamInitGuideViewLayout`是business模块提供的可选UI组件。您可以选择：

1. **使用我们提供的UI组件**：直接集成business模块，使用现成的引导界面
2. **自定义UI实现**：根据`ISlamDeviceListener`回调的状态信息，实现自己的用户界面

##### 选项1：使用提供的UI组件

###### 方式1：在Activity中直接使用

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

###### 方式2：使用独立的引导弹窗Activity

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
        public void onPreProcessPicResult(SlamPreProcessPictureStatus status) {
            // 引导界面不需要处理图片预处理
        }

        @Override
        public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus resultStatus) {
            // 引导界面不需要处理坐标转换
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

##### 选项2：自定义UI实现

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

### 完整集成示例

#### 主Activity完整代码示例

```java
public class ViLidarActivity extends AppCompatActivity {
    private static final String TAG = "ViLidarActivity";

    private enum SlamOperation {
        NONE, OPENING, PAUSING, CLOSING
    }

    private SlamOperation mCurrentOperation = SlamOperation.NONE;

    // 视频流相关组件
    private RtkImageView mRtkImageView;
    private SlamVideoFrameDrafter<IImageDrafterContext> mDrafter;
    private ViLidarPictureViewLayout mViLidarPictureViewLayout;
    private SlamCamera mCamera;
    private VideoFrame mCurrentVideoFrame;

    // UI组件
    private TextView mTvLongitude;
    private TextView mTvLatitude;
    private TextView mTvAltitude;
    private TextView mTvSlamInitState;
    private TextView mTvBatteryLevel;

    private ISlamDeviceListener mSlamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
            runOnUiThread(() -> {
                if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
                    // 只有在开启操作时才启动引导弹窗
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        SlamGuideActivity.start(ViLidarActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
                    String errorMessage = slamSceneStatus.getSlamSceneErrorCode().getMessage();
                    Toast.makeText(ViLidarActivity.this, "操作失败: " + errorMessage,
                            Toast.LENGTH_SHORT).show();
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
        public void onPreProcessPicResult(SlamPreProcessPictureStatus status)
                throws RemoteException {
            runOnUiThread(() -> {
                if (status.getPointToCoordinateStatus() == EnumSlamPrePictureStatus.SUCCESS) {
                    showPicSelectPoint();
                } else {
                    Toast.makeText(ViLidarActivity.this, "图片预处理失败",
                            Toast.LENGTH_SHORT).show();
                }
            });
        }

        @Override
        public void onPixelToCoordinateResult(SlamSampleCoordinateResultStatus resultStatus)
                throws RemoteException {
            runOnUiThread(() -> {
                mTvLongitude.setText(String.valueOf(resultStatus.getLon()));
                mTvLatitude.setText(String.valueOf(resultStatus.getLat()));
                mTvAltitude.setText(String.valueOf(resultStatus.getAlt()));
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
        setContentView(R.layout.activity_vilidar);

        initViews();
        initVideoView();
        initListeners();
    }

    private void initViews() {
        mTvLongitude = findViewById(R.id.tvLongitude);
        mTvLatitude = findViewById(R.id.tvLatitude);
        mTvAltitude = findViewById(R.id.tvAltitude);
        mTvSlamInitState = findViewById(R.id.tvSlamInitStatus);
        mTvBatteryLevel = findViewById(R.id.tvBatteryLevel);

        findViewById(R.id.btnStartViLidar).setOnClickListener(v -> startViLidar());
        findViewById(R.id.btnPauseViLidar).setOnClickListener(v -> pauseViLidar());
        findViewById(R.id.btnRecoverViLidar).setOnClickListener(v -> recoverViLidar());
        findViewById(R.id.btnStopViLidar).setOnClickListener(v -> stopViLidar());
        findViewById(R.id.btnTakePicture).setOnClickListener(v -> takePicture());
        findViewById(R.id.btnGetCoordinate).setOnClickListener(v -> getCoordinateFromPixel());
        findViewById(R.id.btnCancel).setOnClickListener(v -> cancelTakePicture());
    }

    private void initVideoView() {
        mRtkImageView = findViewById(R.id.viewSlam);
        mViLidarPictureViewLayout = findViewById(R.id.viLidarPicture);

        // 创建视频帧绘制器
        mDrafter = new SlamVideoFrameDrafter<>(mRtkImageView, new ImageSurveyVideoLayer());
        mRtkImageView.setVideoDrafter(mDrafter);

        // 设置视频帧接收监听器
        mDrafter.setFrameReceived(new OnVideoFrameReceived() {
            @Override
            public void onDataReceived(@NonNull VideoFrame vf) {
                mCurrentVideoFrame = vf;
            }

            @Override
            public void onError(int error) {
                Log.e(TAG, "视频帧接收错误: " + error);
            }
        });
    }

    private void initListeners() {
        try {
            PositionSource.getInstance().addListener(positionListener);
            SlamDeviceManager.getInstance().addListener(mSlamDeviceListener);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to add listeners", e);
        }
    }

    private void startViLidar() {
        try {
            mCurrentOperation = SlamOperation.OPENING;

            // 打开相机
            mCamera = new SlamCamera(SlamCameraType.FRONT_CAMERA);
            mCamera.setCallback(mDrafter);
            mCamera.open();

            // 显示视频流视图
            mRtkImageView.setVisibility(View.VISIBLE);
            mViLidarPictureViewLayout.setVisibility(View.GONE);

            // 启动惯性导航
            NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
            startInfo.setAntennaHeight(1.8);
            startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
            ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);

            // 开启ViLidar
            SlamDeviceManager.getInstance().openViLidar();
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to start ViLidar", e);
        }
    }

    private void pauseViLidar() {
        try {
            mCurrentOperation = SlamOperation.PAUSING;
            mDrafter.setStopDrafter(true);
            SlamDeviceManager.getInstance().pauseSlam();
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to pause ViLidar", e);
        }
    }

    private void recoverViLidar() {
        try {
            mDrafter.setStopDrafter(false);
            startViLidar();
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to recover ViLidar", e);
        }
    }

    private void stopViLidar() {
        try {
            mCurrentOperation = SlamOperation.CLOSING;
            SlamDeviceManager.getInstance().close();

            if (mCamera != null) {
                mCamera.close();
                mCamera.setCallback(null);
                mCamera = null;
            }

            mRtkImageView.setVisibility(View.GONE);
            mViLidarPictureViewLayout.setVisibility(View.GONE);
        } catch (RemoteException e) {
            mCurrentOperation = SlamOperation.NONE;
            Log.e(TAG, "Failed to stop ViLidar", e);
        }
    }

    private void takePicture() {
        if (mCurrentVideoFrame == null) {
            Toast.makeText(this, "视频帧不可用，无法拍照", Toast.LENGTH_SHORT).show();
            return;
        }

        double gpsTime = mCurrentVideoFrame.getWeek() * 7 * 24 * 3600 +
                mCurrentVideoFrame.getSecOfWeek();
        SlamDeviceManager.getInstance().preProcessPicture(gpsTime);
    }

    private void showPicSelectPoint() {
        mDrafter.setStopDrafter(true);
        mRtkImageView.setVisibility(View.GONE);
        mViLidarPictureViewLayout.setVisibility(View.VISIBLE);

        if (mCurrentVideoFrame != null) {
            mViLidarPictureViewLayout.setBitmapByte(mCurrentVideoFrame.getJpegBuffer());
        }
    }

    private void getCoordinateFromPixel() {
        PointF pixelPoint = mViLidarPictureViewLayout.getCurrentPointInfo();
        if (pixelPoint == null) {
            Toast.makeText(this, "请先在图片上选择点", Toast.LENGTH_SHORT).show();
            return;
        }

        if (mCurrentVideoFrame == null) {
            Toast.makeText(this, "视频帧不可用", Toast.LENGTH_SHORT).show();
            return;
        }

        double gpsTime = mCurrentVideoFrame.getWeek() * 7 * 24 * 3600 +
                mCurrentVideoFrame.getSecOfWeek();
        int pixelX = (int) pixelPoint.x;
        int pixelY = (int) pixelPoint.y;
        SlamDeviceManager.getInstance().calcCoordinateByPixel(gpsTime, pixelX, pixelY);
    }

    private void cancelTakePicture() {
        mDrafter.setStopDrafter(false);
        mRtkImageView.setVisibility(View.VISIBLE);
        mViLidarPictureViewLayout.setVisibility(View.GONE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        try {
            PositionSource.getInstance().removeListener(positionListener);
            SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);

            if (mCamera != null) {
                mCamera.close();
                mCamera.setCallback(null);
                mCamera = null;
            }
        } catch (Exception e) {
            Log.e(TAG, "Error clearing listeners in onDestroy", e);
        }
    }
}
```

### ViLidar与SFix的区别

#### 功能差异

| 功能      | SFix | ViLidar |
|---------|------|---------|
| SLAM定位  | ✅    | ✅       |
| 位置信息获取  | ✅    | ✅       |
| 初始化状态监听 | ✅    | ✅       |
| 电池/存储信息 | ✅    | ✅       |
| 相机视频流   | ❌    | ✅       |
| 拍照功能    | ❌    | ✅       |
| 图片预处理   | ❌    | ✅       |
| 像素点转坐标  | ❌    | ✅       |

#### API差异

| 操作     | SFix          | ViLidar                                |
|--------|---------------|----------------------------------------|
| 开启模式   | `openSfix()`  | `openViLidar()`                        |
| 暂停     | `pauseSlam()` | `pauseSlam()`                          |
| 关闭     | `close()`     | `close()`                              |
| 图片预处理  | -             | `preProcessPicture(gpsTime)`           |
| 像素点转坐标 | -             | `calcCoordinateByPixel(gpsTime, x, y)` |

#### 回调差异

ViLidar相比SFix增加了两个回调：

1. **onPreProcessPicResult**: 图片预处理结果回调
2. **onPixelToCoordinateResult**: 像素点转坐标结果回调

### 常见问题

#### Q1: 如何区分不同的SLAM操作结果？

**A**: 使用操作状态枚举来跟踪当前操作：

```java
private enum SlamOperation {
    NONE, OPENING, PAUSING, CLOSING
}

private SlamOperation mCurrentOperation = SlamOperation.NONE;

// 在执行操作前设置状态
mCurrentOperation =SlamOperation.OPENING;
SlamDeviceManager.

getInstance().

openViLidar();

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

#### Q2: 如何获取GPS时间？

**A**: 从视频帧中获取GPS时间：

```java
private double getGpsTime() {
    if (mCurrentVideoFrame == null) {
        return 0;
    }
    // GPS时间 = GPS周数 * 7天 * 24小时 * 3600秒 + 周内秒数
    return mCurrentVideoFrame.getWeek() * 7 * 24 * 3600 +
            mCurrentVideoFrame.getSecOfWeek();
}
```

#### Q3: 拍照后如何获取用户选择的像素点？

**A**: 使用ViLidarPictureViewLayout获取：

```java
PointF pixelPoint = mViLidarPictureViewLayout.getCurrentPointInfo();
if(pixelPoint !=null){
int pixelX = (int) pixelPoint.x;
int pixelY = (int) pixelPoint.y;
// 使用像素坐标进行转换
}
```

#### Q4: 如何处理图片预处理失败？

**A**: 在onPreProcessPicResult回调中检查状态：

```java

@Override
public void onPreProcessPicResult(SlamPreProcessPictureStatus status) {
    if (status.getPointToCoordinateStatus() == EnumSlamPrePictureStatus.FAIL) {
        // 预处理失败，提示用户重试
        showErrorMessage("图片预处理失败，请重试");
    } else {
        // 预处理成功，显示图片选择界面
        showPicSelectPoint();
    }
}
```

#### Q5: 视频流不显示怎么办？

**A**: 检查以下几点：

1. 确保相机已正确打开：

```java
mCamera =new

SlamCamera(SlamCameraType.FRONT_CAMERA);
mCamera.

setCallback(mDrafter);
mCamera.open();
```

2. 确保视频帧绘制器已正确设置：

```java
mDrafter =new SlamVideoFrameDrafter<>(mRtkImageView,new

ImageSurveyVideoLayer());
        mRtkImageView.

setVideoDrafter(mDrafter);
```

3. 确保视频绘制未被暂停：

```java
mDrafter.setStopDrafter(false);
```

4. 确保视图可见：

```java
mRtkImageView.setVisibility(View.VISIBLE);
```

#### Q6: 内存泄漏问题

**A**: 确保在Activity销毁时正确清理：

```java

@Override
protected void onDestroy() {
    super.onDestroy();
    try {
        // 移除所有监听器
        SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
        PositionSource.getInstance().removeListener(mPositionListener);

        // 关闭相机
        if (mCamera != null) {
            mCamera.close();
            mCamera.setCallback(null);
            mCamera = null;
        }
    } catch (Exception e) {
        Log.e(TAG, "Error clearing listeners", e);
    }
}
```

#### Q7: 拍照和坐标转换的完整流程是什么？

**A**: 完整流程如下：

1. **启动ViLidar**：打开相机，启动SLAM，显示视频流
2. **等待初始化完成**：监听初始化状态，直到`SYSTEM_STATUS_SLAM_RUN`
3. **拍照**：调用`preProcessPicture(gpsTime)`，传入当前视频帧的GPS时间
4. **显示图片**：在`onPreProcessPicResult`回调中，如果预处理成功，显示图片选择界面
5. **选择像素点**：用户在图片上点击选择点
6. **转换坐标**：调用`calcCoordinateByPixel(gpsTime, pixelX, pixelY)`，传入GPS时间和像素坐标
7. **获取结果**：在`onPixelToCoordinateResult`回调中获取转换后的经纬度和高程

#### Q8: 如何取消拍照操作？

**A**: 恢复视频绘制并切换视图：

```java
private void cancelTakePicture() {
    // 恢复视频绘制
    mDrafter.setStopDrafter(false);

    // 显示视频流视图
    mRtkImageView.setVisibility(View.VISIBLE);

    // 隐藏图片选择视图
    mViLidarPictureViewLayout.setVisibility(View.GONE);
}
```

