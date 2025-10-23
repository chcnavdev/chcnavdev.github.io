本节旨在帮助您快速上手使用SDK，重点介绍如何通过蓝牙连接接收机并获取坐标信息。在开始之前，您需要先正确引入华测GnssServerSDK（gnssserver.aar、gnsstoollib.aar、sdk4a.aar）的三个aar文件，可以将其放入lib目录，在gradle中将其引入。

## 权限

在使用SDK之前，请确保应用具有必要的权限。这些权限包括蓝牙（本节中仅使用蓝牙连接，不涉及WIFI）、读取和写入外部存储的权限，因为SDK需要外部存储来读写配置文件和日志文件。请在`AndroidManifest.xml`中添加权限。

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

需要注意的是在Android10、Android11及更高版本中，外部存储的权限申请方式会有些许差异。Android 10引入了Scoped Storage（分区存储），可以在 AndroidManifest.xml 中添加以下代码：

```xml
<application
    android:requestLegacyExternalStorage="true"
    ... >
```

requestLegacyExternalStorage 标记在Android 11中被完全废弃，所有应用都必须遵循Scoped Storage模型，同时引入了一个新的特殊权限，可以让应用获得类似于传统全局文件访问的权限，具体声明代码如下：

```xml
<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE"/>
```

同时，在代码中需要引导用户到系统设置中手动授予权限：

```java
Intent intent = new Intent(Settings.ACTION_MANAGE_ALL_FILES_ACCESS_PERMISSION);
startActivity(intent);
```



其中App.getInstance().getApplicationContext()为应用级别的Context，SdcardUtils.getAppFolder()可以是任何有读写权限的路径。mService.requestReceiverListener(mReceiverListener)主要用于设置数据的回调，如坐标数据等。

##  初始化SDK 

在获得权限后，初始化GNSS服务并开始监听连接状态。以下是初始化步骤：

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

其中App.getInstance().getApplicationContext()为应用级别的Context，SdcardUtils.getAppFolder()可以是任何有读写权限的路径。mService.requestReceiverListener(mReceiverListener)主要用于设置数据的回调，如坐标数据等。

## 连接设备

SDK为您提供两种连接设备的方式，分别为蓝牙连接和WIFI连接，其中蓝牙连接的方式为您提供了两种：通过SDK提供的接口连接，以及通过注入方式实现蓝牙连接和断开，这里为便于您快速上手，仅介绍通过SDK直接实现蓝牙连接的方式，连接主要步骤和代码如下：

- 通过 ReceiverConnectionOption 配置连接参数；
- 通过初始化中得到的 service 来获取 IReceiverConnectManager 对象
- 将“1”中得到的 ReceiverConnectionOption 对象作为参数传入IReceiverConnectManager 对象的 connect 方法中

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

## 断开设备

通过初始化SDK中得到的 service 来获取 IReceiverConnectManager 对象，然后通过调用IReceiverConnectManager 对象的 disconnect 方法即可断开设备。具体代码如下：

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

## 获取位置数据

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

