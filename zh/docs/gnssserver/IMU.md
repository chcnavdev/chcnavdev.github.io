## 坐标信息

如果您需要使用惯性导航功能，可以通过获取 SurveyPointPositionInfo 类的实例并添加监听器，以获取惯导状态下的坐标信息。使用这种方式，您无需关心数据源的切换，即无需根据惯导状态手动切换数据源。具体步骤如下：

```java
// Listening for coordinate information
PositionSource.getInstance().addListener(new IPositionListener.Stub() {
    @Override
    public void onCoordinatePosition(SurveyPointPositionInfo positionInfo) {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                updatePositionInfoUi(positionInfo);
            }
        });
    }
});

// Render the coordinate information to the page
private void updatePositionInfoUi(@NonNull SurveyPointPositionInfo surveyPointPositionInfo) {
    Position position = surveyPointPositionInfo.getCoorKernal().getWgsBlh();
    mTVB.setText(String.valueOf(position.getX()));
    mTVL.setText(String.valueOf(position.getY()));
    mTVH.setText(String.valueOf(position.getZ()));
}

```

## 开启或关闭IMU

您可以通过 ReceiverCmdManager 类向接收机发送命令。因此，IMU 的开启或关闭操作也是通过该类进行的。具体的操作代码如下：

```java
	private void setIMUListener() {
        mOpenIMU.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                NoneMagneticTiltStartInfo startInfo = new NoneMagneticTiltStartInfo();
                // Setting the antenna height and frequency
                startInfo.setAntennaHeight(0);
                startInfo.setFrequency(EnumDataFrequency.DATA_FREQUENCY_5HZ);
                isOpenIMU(true, startInfo);
            }
        });

        mCloseIMU.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                isOpenIMU(false, null);
            }
        });
    }

    private void isOpenIMU(boolean isOpen, @Nullable NoneMagneticTiltStartInfo startInfo) {
    	
        if (isOpen && (startInfo != null)) {
            // Open IMU
            ReceiverCmdManager.getInstance().setCmdStartNoneMagneticTilt(this, startInfo);
        } else {
            // Close IMU
            ReceiverCmdManager.getInstance().setCmdStopNoneMagneticTilt(this);
        }
    }
```

## 其他

关于IMU的其他数据，您可以参考"接收机通信"从接收机获取，相关代码如下:

```java
public class ReceiverListener extends IReceiverListener.Stub {
    private static final String TAG = "ReceiverListener";

    // ......
	
    @Override
    public void getNoneMagneticTiltInfo(NoneMagneticTiltInfo info) throws RemoteException {
        // Implementation of code related to obtaining data
    }
    
    // ......
}
```





