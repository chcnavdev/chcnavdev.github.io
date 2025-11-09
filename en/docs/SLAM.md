# SLAM Integration

## SFix

SFix is a real-time positioning solution based on SLAM (Simultaneous Localization and Mapping) technology, combining GNSS and LiDAR data to provide users with high-precision position information. This document details how to integrate and use SFix functionality in Android applications.

## Core Components

### 1. SlamDeviceManager

The core management class for SFix functionality, responsible for:

- SLAM device initialization and control
- SLAM device start/stop/pause operations
- Status monitoring

### 2. ISlamDeviceListener

SLAM device status listener interface that provides the following callbacks:

- `onSceneStatus()`: SLAM scene operation status changes (start/pause/stop success or failure)
- `onInitStatus()`: SLAM device initialization status changes
- `onBatteryInfoChanged()`: Battery information changes
- `onStorageInfoChanged()`: Storage space information changes

### 3. PositionSource

Position data source that provides real-time coordinate information and solution status

### 4. SlamInitGuideViewLayout (Optional UI Component)

Provides SLAM initialization guidance view based on status

## Quick Start

### 1. Basic Integration

```java
public class SFixActivity extends AppCompatActivity {
    
    // Add operation type enum to distinguish different SLAM operations
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
                // Check if operation is successful
                if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
                    // SLAM opened successfully
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        // Start guidance interface
                        SlamGuideActivity.start(SFixActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.PAUSE)) {
                    // SLAM paused successfully
                    handleSlamPaused();
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.CLOSED)) {
                    // SLAM closed successfully
                    handleSlamClosed();
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
                    // SLAM operation failed
                    String errorMessage = slamSceneStatus.getSlamSceneErrorCode().getMessage();
                    handleSlamError(errorMessage);
                }
                // Reset operation status
                mCurrentOperation = SlamOperation.NONE;
            });
        }

        @Override
        public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
            runOnUiThread(() -> {
                // Handle initialization status changes
                updateSlamInitStatus(slamInitStatus);
            });
        }

        @Override
        public void onBatteryInfoChanged(int remainingPower) throws RemoteException {
            runOnUiThread(() -> {
                // Handle battery changes
                updateBatteryInfo(remainingPower);
            });
        }

        @Override
        public void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) throws RemoteException {
            runOnUiThread(() -> {
                // Handle storage information changes
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
            Log.e("SFix", "Initialization failed", e);
            handleInitError(e);
        }
    }
    
    private void initSFix() throws RemoteException {
        // Add listeners
        PositionSource.getInstance().addListener(positionListener);
        SlamDeviceManager.getInstance().addListener(slamDeviceListener);
    }
}
```

### 2. Start SFix

```java
private void startSFix() {
    try {
        // Set operation type to opening
        mCurrentOperation = SlamOperation.OPENING;
        
        // 1. Start inertial navigation
        NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
        startInfo.setAntennaHeight(1.8); // Set antenna height (meters)
        startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ); // Set data frequency
        ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);
        
        // 2. Open SFix
        SlamDeviceManager.getInstance().openSfix();
        
    } catch (RemoteException e) {
        // Reset operation status on exception
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "Failed to start SFix", e);
        showErrorMessage("Start failed, please check device connection");
    }
}
```

### 3. Pause SFix

```java
private void pauseSFix() {
    try {
        // Set operation type to pausing
        mCurrentOperation = SlamOperation.PAUSING;
        SlamDeviceManager.getInstance().pauseSlam();
    } catch (RemoteException e) {
        // Reset operation status on exception
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "Failed to pause SFix", e);
    }
}
```

### 4. Stop SFix

```java
private void stopSFix() {
    try {
        // Set operation type to closing
        mCurrentOperation = SlamOperation.CLOSING;
        SlamDeviceManager.getInstance().close();
    } catch (RemoteException e) {
        // Reset operation status on exception
        mCurrentOperation = SlamOperation.NONE;
        Log.e("SFix", "Failed to stop SFix", e);
    }
}
```

## Detailed API Reference

### SlamDeviceManager API

#### Core Methods

```java
// Open SFix mode
public void openSfix() throws RemoteException

// Pause SLAM functionality
public void pauseSlam() throws RemoteException

// Close SLAM functionality
public void close() throws RemoteException

// Add status listener
public void addListener(ISlamDeviceListener listener) throws RemoteException

// Remove status listener
public void removeListener(ISlamDeviceListener listener) throws RemoteException
```

### ISlamDeviceListener Callback Interface

```java
public interface ISlamDeviceListener {
    /**
     * SLAM scene operation status callback
     * @param slamSceneStatus Scene status information, including operation result and error code
     */
    void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException;
    
    /**
     * SLAM device initialization status change callback
     * @param slamInitStatus Initialization status information
     */
    void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException;
    
    /**
     * Battery information change callback
     * @param remainingPower Remaining battery percentage (0-100, value is 120 when device is charging)
     */
    void onBatteryInfoChanged(int remainingPower) throws RemoteException;
    
    /**
     * Storage space information change callback
     * @param storageInfo Storage information object
     */
    void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) throws RemoteException;
}
```

### SlamSceneStatus Scene Status Information

SlamSceneStatus contains the following main statuses:

- `OPENED`: SLAM opened successfully
- `PAUSE`: SLAM paused successfully
- `CLOSED`: SLAM closed successfully
- `FAIL`: SLAM operation failed

**Usage Example:**

```java
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) throws RemoteException {
    EnumSlamSceneStatus status = slamSceneStatus.getSlamSceneStatus();
    switch (status) {
        case OPENED:
            // Handle open success
            break;
        case PAUSE:
            // Handle pause success
            break;
        case CLOSED:
            // Handle close success
            break;
        case FAIL:
            // Handle operation failure
            EnumSlamSceneErrorCode errorCode = slamSceneStatus.getSlamSceneErrorCode();
            String errorMessage = errorCode.getMessage();
            break;
        default:
            // Handle default
            break;
    }
}
```

### SlamInitStatus Initialization Status Information

SlamInitStatus contains the following main statuses:

- `SYSTEM_STATUS_UNKNOWN(-1)`: Unknown status, SLAM status updating
- `SYSTEM_STATUS_INITIAL(0)`: Initializing, need to remain stationary
- `SYSTEM_STATUS_INITIAL_NOT_FIX(1)`: Initializing, RTK not fixed, need to move to open area
- `SYSTEM_STATUS_GNSS_INITIAL(2)`: GNSS initialization complete, need to move as prompted to complete SLAM initialization
- `SYSTEM_STATUS_SLAM_RUN(3)`: SLAM running, initialization complete, can start surveying
- `SYSTEM_STATUS_SLAM_RUN_LOW(4)`: SLAM running, accuracy degraded
- `SYSTEM_STATUS_SLAM_RUN_RTK_NOT_FIX(5)`: SLAM running, RTK accuracy poor or not fixed
- `SYSTEM_STATUS_SLAM_STOP(6)`: SLAM stopped
- `SYSTEM_STATUS_SLAM_SAVING(7)`: SLAM saving
- `SYSTEM_STATUS_SLAM_ERROR(8)`: SLAM device error, need to reinitialize
- `SYSTEM_STATUS_SLAM_SYSTEM_INIT(9)`: SLAM system initialization
- `SYSTEM_STATUS_INITIAL_RTK_LESS(10)`: During initialization, RTK signal weak
- `SYSTEM_STATUS_SLAM_RUN_RTK_LESS(11)`: During operation, RTK signal weak

**Usage Example:**

```java
@Override
public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
    EnumSlamInitStatus status = slamInitStatus.getSlamInitStatus();
    switch (status) {
        case SYSTEM_STATUS_INITIAL:
            showMessage("Please remain stationary, initializing...");
            break;
        case SYSTEM_STATUS_INITIAL_NOT_FIX:
            showMessage("Please move to open area, RTK not fixed");
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            showMessage("Please move as prompted to complete SLAM initialization");
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            showMessage("Initialization complete, can start surveying");
            break;
        case SYSTEM_STATUS_SLAM_ERROR:
            showMessage("SLAM device error, please reinitialize");
            break;
        // ... other status handling
    }
}
```

### PositionSource Position Data

```java
// Add position listener
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo positionInfo) {
        // Handle position information
        Position position = positionInfo.getCoorKernal().getWgsBlh();
        double latitude = position.getX();   // Latitude
        double longitude = position.getY();  // Longitude
        double height = position.getZ();     // Height
        
        // Get solution status
        int solveStatus = positionInfo.getSolveStatus().getValue();
    }
});
```

## UI Component Usage

### SlamInitGuideViewLayout Usage

**Important Note**: `SlamInitGuideViewLayout` is an optional UI component provided by the business module. You can choose:

1. **Use the provided UI component**: Directly integrate the business module and use the ready-made guidance interface
2. **Custom UI implementation**: Implement your own user interface based on the status information from `ISlamDeviceListener` callbacks

#### Option 1: Use the Provided UI Component

##### Method 1: Use directly in Activity

```java
private SlamInitGuideViewLayout mSlamInitGuideViewLayout;

private void initViews() {
    mSlamInitGuideViewLayout = findViewById(R.id.viewSlamInit);
}

private ISlamDeviceListener slamDeviceListener = new ISlamDeviceListener.Stub() {
    @Override
    public void onInitStatus(SlamInitStatus slamInitStatus) throws RemoteException {
        runOnUiThread(() -> {
            // Automatically update UI status
            if (mSlamInitGuideViewLayout != null) {
                mSlamInitGuideViewLayout.updateStatus(slamInitStatus);
            }
        });
    }
    // ... other callback methods
};
```

##### Method 2: Use independent guidance dialog Activity

```java
/**
 * SLAM initialization guidance dialog Activity
 * Displays SlamInitGuideViewLayout as a dialog
 */
public class SlamGuideActivity extends AppCompatActivity {
    
    private SlamInitGuideViewLayout mSlamInitGuideViewLayout;
    
    private final ISlamDeviceListener mSlamDeviceListener = new ISlamDeviceListener.Stub() {
        @Override
        public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
            // No need to handle scene status
        }

        @Override
        public void onInitStatus(SlamInitStatus slamInitStatus) {
            runOnUiThread(() -> {
                // Update guidance interface status
                mSlamInitGuideViewLayout.updateStatus(slamInitStatus);
                
                // If SLAM runs successfully, automatically close guidance interface
                if (slamInitStatus.getSlamInitStatus() == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN) {
                    new Handler().postDelayed(() -> finish(), 200);
                }
            });
        }

        @Override
        public void onBatteryInfoChanged(int remainingPower) {
            // Guidance interface doesn't need to handle battery information
        }

        @Override
        public void onStorageInfoChanged(SlamDeviceStorageInfo storageInfo) {
            // Guidance interface doesn't need to handle storage information
        }
    };

    /**
     * Start SLAM guidance dialog
     */
    public static void start(Context context) {
        Intent intent = new Intent(context, SlamGuideActivity.class);
        context.startActivity(intent);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setupDialogStyle(); // Set dialog style
        setContentView(R.layout.activity_slam_guide);
        
        initView();
        initListener();
    }

    private void initListener() {
        // Register SLAM device listener
        SlamDeviceManager.getInstance().addListener(mSlamDeviceListener);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // Remove listener
        SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
    }
}
```

#### Option 2: Custom UI Implementation

If you choose to implement a custom UI, you can handle it based on status callbacks:

```java
private void updateCustomUI(SlamInitStatus slamInitStatus) {
    EnumSlamInitStatus status = slamInitStatus.getSlamInitStatus();
    switch (status) {
        case SYSTEM_STATUS_INITIAL:
            showCustomGuidance("Please remain stationary", R.drawable.keep_still_animation);
            break;
        case SYSTEM_STATUS_GNSS_INITIAL:
            showCustomGuidance("Please move as prompted", R.drawable.move_guidance_animation);
            break;
        case SYSTEM_STATUS_SLAM_RUN:
            hideGuidanceAndStartSurvey();
            break;
        // ... other status handling
    }
}
```

## Complete Integration Example

### Main Activity Complete Code Example

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
                    // Only start guidance dialog when opening operation
                    if (mCurrentOperation == SlamOperation.OPENING) {
                        SlamGuideActivity.start(SFixActivity.this);
                    }
                } else if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
                    String errorMessage = slamSceneStatus.getSlamSceneErrorCode().getMessage();
                    Toast.makeText(SFixActivity.this, "Operation failed: " + errorMessage, Toast.LENGTH_SHORT).show();
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
            // Handle storage information
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
            
            // Start inertial navigation
            NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
            startInfo.setAntennaHeight(1.8);
            startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
            ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);

            // Open SFix
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

## FAQ

### Q1: How to distinguish different SLAM operation results?

**A**: Use an operation status enum to track the current operation:

```java
private enum SlamOperation {
    NONE, OPENING, PAUSING, CLOSING
}

private SlamOperation mCurrentOperation = SlamOperation.NONE;

// Set status before executing operation
mCurrentOperation = SlamOperation.OPENING;
SlamDeviceManager.getInstance().openSfix();

// Handle result based on status in callback
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
    if (mCurrentOperation == SlamOperation.OPENING && 
        slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.OPENED)) {
        // Handle open success
    }
    mCurrentOperation = SlamOperation.NONE;
}
```

### Q2: Too many SLAM initialization statuses, how to simplify handling?

**A**: You can group statuses for handling:

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

### Q3: How to handle SLAM operation failures?

**A**: Check the error code in SlamSceneStatus:

```java
@Override
public void onSceneStatus(SlamSceneStatus slamSceneStatus) {
    if (slamSceneStatus.getSlamSceneStatus().equals(EnumSlamSceneStatus.FAIL)) {
        EnumSlamSceneErrorCode errorCode = slamSceneStatus.getSlamSceneErrorCode();
        String errorMessage = errorCode.getMessage();
        
        // Handle accordingly based on error code
        handleSlamError(errorCode, errorMessage);
    }
}
```

### Q4: When does the guidance interface automatically close?

**A**: It automatically closes when SLAM status changes to `SYSTEM_STATUS_SLAM_RUN`:

```java
@Override
public void onInitStatus(SlamInitStatus slamInitStatus) {
    if (slamInitStatus.getSlamInitStatus() == EnumSlamInitStatus.SYSTEM_STATUS_SLAM_RUN) {
        // Delay closing to let user see completion status
        new Handler().postDelayed(() -> finish(), 200);
    }
}
```

### Q5: Memory leak issues

**A**: Ensure proper cleanup when Activity is destroyed:

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    try {
        // Remove all listeners
        SlamDeviceManager.getInstance().removeListener(mSlamDeviceListener);
        PositionSource.getInstance().removeListener(mPositionListener);
    } catch (Exception e) {
        Log.e(TAG, "Error clearing listeners", e);
    }
}
```