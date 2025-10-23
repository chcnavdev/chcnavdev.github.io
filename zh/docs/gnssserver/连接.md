## 连接方式

### 蓝牙连接

蓝牙连接主要步骤和代码如下：

1. 通过 ReceiverConnectionOption 配置连接参数；
2. 通过初始化SDK中得到的 service 来获取 IReceiverConnectManager 对象
3. 将“1”中得到的 ReceiverConnectionOption 对象作为参数传入IReceiverConnectManager 对象的 connect 方法中

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

### 蓝牙注入连接（可选） 

推荐使用普通的蓝牙连接方式，这里蓝牙注入的连接方式仅为有特殊需求的开发者提供选择。

#### 步骤一：蓝牙连接的实现

通过这种方式，开发者需要继承`BaseReceiverConnectionInjector`类，自行实现蓝牙连接部分，并将其注入到`ReceiverConnectionInjectManager`中。这里继承 BaseReceiverConnectionInjector 需要实现的主要有四个方法，具体步骤和代码如下：

1. 实现 connnect 方法，这部分您需要自行实现蓝牙连接的socket；
2. 实现 disConnect 方法；
3. 实现 receiver 方法，这里直接调用父类方法即可；
4. 实现 writeDataToDevice 方法。

```java
/**
 * BluetoothConnectionInjector handles the establishment, maintenance, and termination
 * of a Bluetooth connection. It extends BaseReceiverConnectionInjector to inherit 
 * common connection functionalities.
 */
public class BluetoothConnectionInjector extends BaseReceiverConnectionInjector {
    /**
     * Log tag for debugging.
     */
    private static final String TAG = "BluetoothReceiverConn";
    /**
     * UUID for the Bluetooth service.
     */
    private static final UUID MY_UUID = 
      UUID.fromString("00001101-0000-1000-8000-00805F9B34FB");
    /**
     * Bluetooth device to connect to.
     */
    private final BluetoothDevice mBluetoothDevice;
    /**
     * Bluetooth socket for the connection.
     */
    private BluetoothSocket mBluetoothSocket;
    /**
     * Input stream for reading data from the Bluetooth connection.
     */
    private InputStream mInputStream;
    /**
     * Output stream for writing data to the Bluetooth connection.
     */
    private OutputStream mOutputStream;
    /**
     * Constructor initializes the Bluetooth device by its name.
     *
     * @param btName Name of the Bluetooth device to connect to.
     */
    public BluetoothConnectionInjector(@NonNull String btName) {
        this.mBluetoothDevice = getBluetoothDeviceByName(btName);
    }
    
    /**
     * Establishes the Bluetooth connection.
     *
     * @param context Application context.
     * @return true if connection is successful, false otherwise.
     */
    @Override
    public boolean connect(@NonNull Context context) {
        try {
            initializeSocket();
            startDataReceiverThread();
            onStateChanged(EnumConnectionStatus.CONNECT_SUCCESED);
            return true;
        } catch (IOException e) {
            handleConnectionError(e);
            return false;
        }
    }

    /**
     * Initializes the Bluetooth socket and streams.
     *
     * @throws IOException if an I/O error occurs.
     */
    private void initializeSocket() throws IOException {
        mBluetoothSocket = mBluetoothDevice.createRfcommSocketToServiceRecord(MY_UUID);
        mBluetoothSocket.connect();
        mInputStream = mBluetoothSocket.getInputStream();
        mOutputStream = mBluetoothSocket.getOutputStream();
    }

    /**
     * Starts a thread to receive data from the Bluetooth connection.
     */
    private void startDataReceiverThread() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    byte[] buffer = new byte[1024];
                    int bytesRead;
                    while ((bytesRead = mInputStream.read(buffer)) > 0) {
                        byte[] data = new byte[bytesRead];
                        System.arraycopy(buffer, 0, data, 0, bytesRead);
                        receiver(data);
                    }
                } catch (IOException e) {
                    handleDataReceiverError(e);
                } finally {
                    disConnect();
                }
            }
        }).start();
    }

    /**
     * Handles connection errors by logging and disconnecting.
     *
     * @param e IOException that occurred.
     */
    private void handleConnectionError(IOException e) {
        Log.e(TAG, "Connection error: ", e);
        disConnect();
    }

    /**
     * Handles data receiver errors by logging.
     *
     * @param e IOException that occurred.
     */
    private void handleDataReceiverError(IOException e) {
        Log.e(TAG, "Data receiver error: ", e);
    }

    /**
     * Disconnects the Bluetooth connection and closes all streams.
     */
    @Override
    public void disConnect() {
        closeStream(mInputStream);
        closeStream(mOutputStream);
        closeSocket(mBluetoothSocket);
    }

    /**
     * Closes a given stream.
     *
     * @param stream Stream to close.
     */
    private void closeStream(@Nullable Closeable stream) {
        if (stream != null) {
            try {
                stream.close();
            } catch (IOException e) {
                Log.e(TAG, "Error closing stream: ", e);
            }
        }
    }
​
    /**
     * Closes the Bluetooth socket.
     *
     * @param socket Bluetooth socket to close.
     */
    private void closeSocket(@Nullable BluetoothSocket socket) {
        if (socket != null) {
            try {
                socket.close();
            } catch (IOException e) {
                Log.e(TAG, "Error closing socket: ", e);
            }
        }
    }

    /**
     * Writes data to the Bluetooth device.
     *
     * @param cmd Command containing the data to write.
     * @return true if data is successfully written, false otherwise.
     */
    @Override
    public boolean writeDataToDevice(@NonNull CMD cmd) {
        if (mOutputStream == null) return false;

        try {
            mOutputStream.write(cmd.cmd);
            mOutputStream.flush();
            return true;
        } catch (IOException e) {
            Log.e(TAG, "Error writing data: ", e);
            return false;
        }
    }

    /**
     * Receives data and passes it to the superclass method.
     *
     * @param data Data received from the Bluetooth connection.
     */
    @Override
    public void receiver(byte[] data) {
        super.receiver(data);
    }

    /**
     * Gets a Bluetooth device by its name.
     *
     * @param btName Name of the Bluetooth device.
     * @return BluetoothDevice if found, null otherwise.
     */
    public static BluetoothDevice getBluetoothDeviceByName(@NonNull String btName) {
        BluetoothAdapter bluetoothAdapter = BluetoothAdapter.getDefaultAdapter();
        if (bluetoothAdapter == null) {
            return null;
        }
        for (BluetoothDevice device : bluetoothAdapter.getBondedDevices()) {
            if (btName.equalsIgnoreCase(device.getName())) {
                return device;
            }
        }
        return null;
    }
}

```

#### 步骤二：蓝牙注入 

对上面自行实现的蓝牙连接器，您需要将它的实例化对象注入到 ReceiverConnectionInjectManager中，然后需要构建ReceiverConnectionOption对象作为参数传入IReceiverConnectManager 对象的 connect 方法中。

```java
// Establish connection using Bluetooth injection method
BluetoothConnectionInjector injector = 
  new BluetoothConnectionInjector("GNSS-9999848");
ReceiverConnectionInjectManager.getInstance().inject(injector);
ReceiverConnectionOption option = new ReceiverConnectionOption()
        .setConnectionType(ConnectionType.CONNECTION_BLUETOOTH)
        .setProductName("SMART-RTK");
try {
    // Get receiverConnectManager through mService and then connect using receiverConnectManager
    IReceiverConnectManager receiverConnectManager = IReceiverConnectManager
            .Stub.asInterface(mService.getReceiverConnectManager());
    // Connect using receiverConnectManager
    receiverConnectManager.connect(option);
} catch (RemoteException e) {
    throw new RuntimeException(e);
}
```

### WIFI连接

WIFI连接主要步骤和代码如下：

1. 通过 ReceiverConnectionOption 配置连接参数；
2. 通过初始化SDK中得到的 service 来获取 IReceiverConnectManager 对象
3. 将“1”中得到的 ReceiverConnectionOption 对象作为参数传入IReceiverConnectManager 对象的 connect 方法中

```java
    // Connection event
    mBTConnect.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
            ReceiverConnectionOption option = new ReceiverConnectionOption()
                    .setConnectionType(ConnectionType.CONNECTION_WIFI)
                    .setProductName("SMART-RTK");
           
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

### 本地连接

目前大多数Android设备都具备接收卫星信号来实现定位的功能，基于此，您可以在不通过蓝牙、WIFI连接RTK的情况下做一些功能测试，也可以仅通过本地连接来开发您需要的功能。本章节主要介绍如何通过SDK实现本地连接，连接步骤同蓝牙连接类似，这里不再赘述，仅给出相关代码。

```java
// Connection event
    mLocalConnect.setOnClickListener(new View.OnClickListener() {
        @Override
        public void onClick(View view) {
            ReceiverConnectionOption option = new ReceiverConnectionOption()
                    .setConnectionType(ConnectionType.CONNECTION_ANDROID)
                    .setProductName("LT40");
           
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
    ​
    // Disconnection event
    mLocalDisConnect.setOnClickListener(new View.OnClickListener() {
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



### LT800H 连接

参考初始化SDK，您只需要修改连接部分的代码即可，具体代码如下：

```java
mBTConnect.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        // Areas that need to be revised
        ReceiverConnectionOption option = new ReceiverConnectionOption()
                .setConnectionType(ConnectionType.CONNECTION_ANDROID)
                .setProductName("LT800H");
       // ......
    }
});
```

