# Connection Methods

## Bluetooth Connection

The main steps and code for Bluetooth connection are as follows:

1. Configure connection parameters through ReceiverConnectionOption
2. Get the IReceiverConnectManager object through the service obtained in SDK initialization
3. Pass the ReceiverConnectionOption object obtained in step 1 as a parameter to the connect method of the IReceiverConnectManager object

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

## Bluetooth Injection Connection (Optional)

It is recommended to use the normal Bluetooth connection method. The Bluetooth injection connection method here is only provided as an option for developers with special requirements.

### Step 1: Implementation of Bluetooth Connection

Through this method, developers need to inherit the `BaseReceiverConnectionInjector` class, implement the Bluetooth connection part themselves, and inject it into the `ReceiverConnectionInjectManager`. Here, inheriting BaseReceiverConnectionInjector mainly requires implementing four methods. The specific steps and code are as follows:

1. Implement the connect method, where you need to implement the Bluetooth connection socket yourself
2. Implement the disConnect method
3. Implement the receiver method, where you can directly call the parent class method
4. Implement the writeDataToDevice method

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

### Step 2: Bluetooth Injection

For the Bluetooth connector implemented above, you need to inject its instantiated object into ReceiverConnectionInjectManager, and then build a ReceiverConnectionOption object as a parameter to pass into the connect method of the IReceiverConnectManager object.

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

## WiFi Connection

The main steps and code for WiFi connection are as follows:

1. Configure connection parameters through ReceiverConnectionOption
2. Get the IReceiverConnectManager object through the service obtained in SDK initialization
3. Pass the ReceiverConnectionOption object obtained in step 1 as a parameter to the connect method of the IReceiverConnectManager object

```java
    // Connection event
    mWiFiConnect.setOnClickListener(new View.OnClickListener() {
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
    mWiFiDisConnect.setOnClickListener(new View.OnClickListener() {
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

## Local Connection

Currently, most Android devices have the capability to receive satellite signals for positioning. Based on this, you can perform some functional testing without connecting to RTK via Bluetooth or WiFi, or you can develop the functions you need using only local connection. This section mainly introduces how to implement local connection through the SDK. The connection steps are similar to Bluetooth connection, so they will not be repeated here. Only the relevant code is provided.

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

## LT800H Connection

Referring to the SDK initialization, you only need to modify the connection part of the code. The specific code is as follows:

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

