# Quick Start

This section aims to help you quickly get started with the SDK, focusing on how to connect to receivers via Bluetooth and obtain coordinate information. Before starting, you need to properly import the three AAR files of CHCNAV GnssServerSDK (gnssserver.aar, gnsstoollib.aar, sdk4a.aar). You can place them in the lib directory and import them in Gradle.

## Permissions

Before using the SDK, ensure that your application has the necessary permissions. These permissions include Bluetooth (this section only uses Bluetooth connection, not involving WiFi), and permissions to read and write external storage, as the SDK needs external storage to read and write configuration files and log files. Please add permissions in `AndroidManifest.xml`.

```xml
<!-- Permission to use Bluetooth features -->
<uses-permission android:name="android.permission.BLUETOOTH"/>
<!-- Permission to administer Bluetooth settings -->
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
<!-- Permission to read from external storage -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<!-- Permission to write to external storage -->
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

Note that in Android 10, Android 11, and higher versions, the permission request method for external storage has some differences. Android 10 introduced Scoped Storage, and you can add the following code in AndroidManifest.xml:

```xml
<application
    android:requestLegacyExternalStorage="true"
    ... >
```

The requestLegacyExternalStorage flag is completely deprecated in Android 11, and all applications must follow the Scoped Storage model. At the same time, a new special permission is introduced that allows applications to obtain permissions similar to traditional global file access. The specific declaration code is as follows:

```xml
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE"/>
```

At the same time, in the code, you need to guide users to manually grant permissions in system settings:

```java
Intent intent = new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION);
startActivity(intent);
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

Where `App.getInstance().getApplicationContext()` is the application-level Context, and `SdcardUtils.getAppFolder()` can be any path with read and write permissions. `mService.requestReceiverListener(mReceiverListener)` is mainly used to set data callbacks, such as coordinate data.

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

