# IMU

## Coordinate Information

If you need to use inertial navigation functionality, you can obtain coordinate information in the inertial navigation state by getting an instance of the SurveyPointPositionInfo class and adding a listener. Using this method, you don't need to worry about data source switching, i.e., you don't need to manually switch data sources based on the inertial navigation state. The specific steps are as follows:

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

## Enable or Disable IMU

You can send commands to the receiver through the ReceiverCmdManager class. Therefore, the enable or disable operations of IMU are also performed through this class. The specific operation code is as follows:

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

## Other Data

For other IMU data, you can refer to "Receiver Communication" to obtain it from the receiver. The related code is as follows:

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





