# Quick Start

This section aims to help you quickly get started with the SDK, focusing on how to connect to receivers via Bluetooth and obtain coordinate information. Before starting, you need to properly import the three AAR files of CHCNAV GnssServerSDK (gnssserver.aar, gnsstoollib.aar, sdk4a.aar). You can place them in the lib directory and import them in Gradle.

## Permissions

Before using the SDK, ensure that your application has the necessary permissions. This section only uses Bluetooth connection and does not involve WiFi. Please add Bluetooth permissions in `AndroidManifest.xml`:

```xml
<!-- Permission to use Bluetooth features -->
<uses-permission android:name="android.permission.BLUETOOTH"/>
<!-- Permission to administer Bluetooth settings -->
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
```

## Initialize SDK

After obtaining permissions, initialize the GNSS service and start listening to connection status. The following are the initialization steps:

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        try {
            // Initialize the Service using ApplicationContext and resource file path parameter
            GnssServiceHelper.getInstance().start(App.getInstance().getApplicationContext(), SdcardUtils.getAppFolder());
            // Get the Service instance
            mService = GnssServiceHelper.getInstance().getGnssService();
            // Receiver data callback
            mService.requestReceiverListener(mReceiverListener);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}).start();
```

Where `App.getInstance().getApplicationContext()` is the application-level Context, and `SdcardUtils.getAppFolder()` can be an appropriate path. `mService.requestReceiverListener(mReceiverListener)` is mainly used to set data callbacks, such as coordinate data.

## Connect Device

The SDK provides you with two ways to connect devices: Bluetooth connection and WiFi connection. For Bluetooth connection, two methods are provided: connecting through the interface provided by the SDK, and implementing Bluetooth connection and disconnection through injection. For your quick start, only the method of directly implementing Bluetooth connection through the SDK is introduced here. The main steps and code for connection are as follows:

- Configure connection parameters through ReceiverConnectionOption
- Get the IReceiverConnectManager object through the service obtained in initialization
- Pass the ReceiverConnectionOption object obtained in step 1 as a parameter to the connect method of the IReceiverConnectManager object

```java
// Connection event
mBTConnect.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        // Connect directly through the SDK
        ReceiverConnectionOption option = new ReceiverConnectionOption()
                .setConnectionType(ConnectionType.CONNECTION_BLUETOOTH)
                .setProductName("SMART-RTK")
                .setBluetoothName("GNSS-9999848");
       
        try {
            // Get receiverConnectManager through mService and then connect using receiverConnectManager
            IReceiverConnectManager receiverConnectManager = IReceiverConnectManager
                    .Stub.asInterface(mService.getReceiverConnectManager());
            // Connect using receiverConnectManager
            if (!receiverConnectManager.connect(option)) {
                // Connection failure
            }
        } catch (RemoteException e) {
            throw new RuntimeException(e);
        }
    }
});
```

## Disconnect Device

Get the IReceiverConnectManager object through the service obtained in SDK initialization, and then disconnect the device by calling the disconnect method of the IReceiverConnectManager object. The specific code is as follows:

```java
// Disconnection event
mBTDisConnect.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        try {
            IReceiverConnectManager receiverConnectManager = IReceiverConnectManager
                    .Stub.asInterface(mService.getReceiverConnectManager());
            if (receiverConnectManager != null) {
                receiverConnectManager.disConnect();
            }
        } catch (RemoteException e) {
            throw new RuntimeException(e);
        }
    }
});
```

## Get Position Data

```java
// Monitor location data
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
    mPoint3DMutable = new Point3DMutable(
            position.getX(),
            position.getY(),
            position.getZ()
    );
    tvB.setText(String.valueOf(position.getX()));
    tvL.setText(String.valueOf(position.getY()));
    tvH.setText(String.valueOf(position.getZ()));
    tvPositionState.setText(String.valueOf(surveyPointPositionInfo.getSolveStatus().getValue()));
}
```

