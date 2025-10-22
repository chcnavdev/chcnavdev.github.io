# Receiver Communication

## Getting Data from Receiver

After obtaining the service, you can implement listening to receiver data through the service's requestReceiverListener method. requestReceiverListener requires you to pass in an IReceiverListener.Stub object. You can inherit this object and implement related interfaces to obtain the required data through interface implementation. Some callback interfaces will callback at a certain frequency (such as position information and satellite information), while some interfaces require sending commands to the receiver to trigger callbacks.

## Sending Data to Receiver (Setting Receiver)

You can send commands to the receiver through the ReceiverCmdManager class. This class is a singleton and can be called directly. The meaning of each interface can be found in the interface comments.

```java
/***
 * Send differential data
 *
 * @param bts Differential data
 */
public void setCmdSendDiffDataToOem(Context context, byte[] bts) {}

/***
 * Disable all IOs except for the radio, built-in network, and handheld network
 *
 * @param currentIos IO
 */
public void setCmdDisableOtherIos(Context context, int currentIos) {}

/***
 * Update the radio power switch
 */
public void setCmdUpdateRadioPowerOn(Context context, boolean powerOn) {}

/***
 * Update whether the radio auto powers on
 */
public void setCmdUpdateRadioAutoPower(Context context, boolean autoPowerOn) {}

/***
 * Set the receiver's radio power
 */
public void setCmdUpdateRadioPower(Context context, double power) {}

/***
 * Set the receiver's radio air baud rate
 */
public void setCmdUpdateRadioAirBaudrate(Context context, int baudrate) {}

/***
 * Set the receiver's radio frequency
 */
public void setCmdUpdateRadioFrequency(Context context, double frequency) {}

/***
 * Set the receiver's radio protocol
 */
public void setCmdUpdateRadioProtocol(Context context, EnumRadioProtocol protocol) {}

/***
 * Set the receiver's radio stepper value
 */
public void setCmdUpdateRadioStepper(Context context, double stepper) {}

/***
 * Set the radio sensitivity
 */
public void setCmdUpdateRadioSensitivity(Context context, EnumSensitivity sensitivity) {}

/***
 * Enable or disable FEC
 */
public void setCmdUpdateRadioFec(Context context, boolean enable) {}

/***
 * Set the call sign
 */
public void setCmdUpdateRadioCallSign(Context context, DataRadioCallsign callSign) {}

/***
 * Set the receiver's GPRS module information
 */
public void setCmdUpdateGprsInfo(Context context, GprsInfo info) {}

/***
 * Set the receiver's CORS information
 */
public void setCmdUpdateCorsInfo(Context context, CorsInfo info) {}

/***
 * Log in to the receiver's GPRS module
 */
public void setCmdLoginGprs(Context context) {}

/***
 * Set the 3G module state to GSM or GPRS
 */
public void setCmdUpdateModemCommunicationMode(Context context, EnumModemCommunicationMode mode) {}

/***
 * Set the 3G module GSM dial parameters
 */
public void setCmdUpdateCsdInfo(Context context, CsdInfo info) {}

/***
 * Dial or disconnect the 3G module
 */
public void setCmdDialModem(Context context, boolean isDial) {}

/***
 * Turn the 3G module on or off
 */
public void setCmdPowerModem(Context context, boolean powerOn) {}

/***
 * Set the 3G module dial parameters
 */
public void setCmdUpdateModemDialParams(Context context, ModemDialParams params) {}

/***
 * Set the receiver position information output
 */
public void setCmdOutputPosData(Context context, EnumDataFrequency rtkFrequency) {}

/***
 * Set the receiver rover parameters
 */
public void setCmdStartRover(Context context, RoverParams roverParams) {}

/***
 * Set the receiver base station parameters
 */
public void setCmdStartBase(Context context, BaseParams params) {}

/***
 * Start or stop file recording
 */
public void setCmdStartFileRecord(Context context, boolean isOpenStaticSet) {}

/***
 * Update file auto-recording settings
 */
public void setCmdUpdateFileRecordAutoStart(Context context, boolean isAutoStart) {}

/***
 * Update file recording parameters
 */
public void setCmdUpdateFileRecordParams(Context context, FileRecordInfo info) {}

/***
 * Query the receiver IO corresponding output data
 */
public void setCmdQueryIoData(Context context, EnumGnssIoId ioId) {}

/***
 * Query the RTK position output frequency interface
 */
public void setCmdQueryPosDataFrequency(Context context) {}

/***
 * Get the receiver information
 */
public void setCmdQueryReceiverInfo(Context context) {}

/***
 * Get the receiver base station parameters
 */
public void setCmdQueryBaseParams(Context context) {}

/***
 * Query file recording status
 */
public void setCmdQueryFileRecordStatus(Context context) {}

/***
 * Get the receiver's radio module power status
 */
public void setCmdQueryRadioPowerStatus(Context context) {}

/***
 * Get the 3G module state, GSM or GPRS
 */
public void setCmdQueryModemCommunicationMode(Context context) {}

/***
 * Get the working status of the receiver's radio, network, and serial ports
 */
public void setCmdQueryIoEnable(Context context) {}

/***
 * Get the receiver registration code
 */
public void setCmdQueryRegCode(Context context) {}

/***
 * Get the 3G module GSM dial status
 */
public void setCmdQueryCsdDialStatus(Context context) {}

/***
 * Get the receiver's GPRS module status
 */
public void setCmdQueryGprsStatus(Context context) {}

/***
 * Get the receiver's GPRS module signal strength
 */
public void setCmdQueryModemSignal(Context context) {}

/***
 * Get whether the 3G module auto dials
 */
public void setCmdQueryModemAutoDial(Context context) {}

/***
 * Get whether the 3G module auto powers on
 */
public void setCmdQueryModemAutoPowerOn(Context context) {}

/***
 * Set whether the 3G module auto powers on
 */
public void setCmdUpdateModemAutoPowerOn(Context context, boolean autoPower) {}

/***
 * Set whether the 3G module auto dials
 */
public void setCmdUpdateModemAutoDial(Context context, boolean autoDial) {}

/***
 * Get the 3G module dial status
 */
public void setCmdQueryModemDialStatus(Context context) {}

/***
 * Get the 3G module power status
 */
public void setCmdQueryModemPowerStatus(Context context) {}

/***
 * Get the 3G module frequency band
 */
public void setCmdQueryModemBandMode(Context context) {}

/***
 * Set the 3G module frequency band
 */
public void setCmdUpdateModemBandMode(Context context, EnumModemBandMode modemBandMode) {}

/***
 * Get the 3G module GSM information
 */
public void setCmdQueryCsdInfo(Context context) {}

/***
 * Query file recording parameters
 */
public void setCmdQueryFileRecordParams(Context context) {}

/***
 * Get whether the file auto records
 */
public void setCmdQueryFileRecordAutoStart(Context context) {}

/***
 * Get the receiver's radio information
 */
public void setCmdQueryRadioInfo(Context context) {}

/***
 * Get the receiver's radio channel list
 */
public void setCmdQueryRadioChannelList(Context context) {}

/***
 * Get the 3G module dial parameters
 */
public void setCmdQueryModemDialParams(Context context) {}

/***
 * Get the receiver's CORS settings information
 */
public void setCmdQueryCorsInfo(Context context) {}

/***
 * Get whether the receiver's radio auto powers on
 */
public void setCmdQueryRadioAutoPower(Context context) {}

/***
 * Get whether the WIFI module auto powers on
 */
public void setCmdQueryWifiAutoPowerOn(Context context) {}

/***
 * Get the WIFI module parameters
 */
public void setCmdQueryWifiParams(Context context) {}

/***
 * Get the WIFI status
 */
public void setCmdQueryWifiStatus(Context context) {}

/***
 * Set whether the WIFI module auto powers on
 */
public void setCmdUpdateWifiAutoPowerOn(Context context, boolean isAutoOpen) {}

/***
 * Update the WIFI module parameters
 */
public void setCmdUpdateWifiParams(Context context, ReceiverWifiInfo info) {}

/***
 * Turn the WIFI module on or off
 */
public void setCmdUpdateWifiPowerOn(Context context, boolean isOpenWifi) {}

/***
 * Get the receiver's GPRS module information
 */
public void setCmdQueryGprsInfo(Context context) {}

/***
 * Disconnect the receiver's GPRS module
 */
public void setCmdBreakGprs(Context context) {}

/***
 * Reset the receiver
 */
public void setCmdResetReceiver(Context context) {}

/***
 * Reset the receiver, clearing only the ephemeris without clearing the configuration
 */
public void setCmdEphremisReset(Context context) {}

/***
 * Register the receiver
 */
public void setCmdRegReceiver(Context context, RegisterCode code) {}

/***
 * Set the receiver's serial port baud rate
 */
public void setCmdUpdateComBaudrate(Context context, EnumBaudrate baudrate) {}

/***
 * Output NMEA data from the receiver
 */
public void setCmdOutputNmea(Context context, NmeaSetParam param) {}

/***
 * Get the current NMEA output list of the specified IO of the receiver
 */
public void setCmdQueryNmeaOutputList(Context context, EnumGnssIoId ioId) {}

/***
 * Stop outputting information on a specific IO port of the receiver
 */
public void setCmdSetGnssDataUnLogall(Context context, EnumGnssIoId ioId) {}

/***
 * Get the receiver's serial port baud rate
 */
public void setCmdQueryComBaudrate(Context context) {}

/***
 * Get the source list
 */
public void setCmdQuerySourceTable(Context context) {}

/***
 * Get the receiver battery life
 */
public void setCmdQueryBatteryLife(Context context) {}

/***
 * Power off the receiver
 */
public void setCmdPowerOffReceiver(Context context) {}

/***
 * Get the receiver's elevation mask angle
 */
public void setCmdQueryGnssElevMask(Context context) {}

/***
 * Get the receiver's PDOP limit value
 */
public void setCmdQueryGnssPDopMask(Context context) {}

/***
 * Get the advanced base station information list
 */
public void setCmdQueryBasePositionList(Context context) {}

/***
 * Clear receiver advanced reference station information
 */
public void setCmdClearBasePostionList(Context context) {}

/***
 * Set receiver advanced reference station information
 */
public void setCmdAddPostionToBaseList(Context context, BasePositionInfo info) {}

/***
 * Delete a piece of receiver advanced reference station information
 */
public void setCmdRemovePostionFromBaseList(Context context, int index) {}

/***
 * Set the receiver advanced reference station translation amount
 */
public void setCmdUpdateBasePositionDifference(Context context, float diff) {}

/***
 * Obtain receiver advanced reference station translation
 */
public void setCmdQueryBasePositionDifference(Context context) {}

/***
 * Get whether the receiver's GPRS module automatically logs in
 */
public void setCmdQueryGprsLoginMdl(Context context) {}

/***
 * Set whether the receiver's GPRS module automatically logs in
 */
public void setCmdUpdateGprsLoginMdl(Context context, boolean autoLogin) {}

/***
 * Set the tilt sensor data to be output at a fixed frequency
 */
public void setCmdOutputEBubbleData(Context context, EnumDataFrequency frequency) {}

/***
 * Set gyroscope data to be output at a fixed frequency
 */
public void setCmdOutputMagneticData(Context context, EnumDataFrequency frequency) {}

/***
 * Sensor calibration
 */
public void setCmdEbubbleCalibration(Context context, EbubbleCalibration calibration) {}

/***
 * Download files msgdata PpkStartInfo
 */
public void setCmdStartPpk(Context context, PpkStartInfo ppkStartInfo) {}

/***
 * Download files msgdata PPKStopInfo
 */
public void setCmdStopPPK(Context context, PpkStopInfo ppkStopInfo) {}

/***
 * Data routing
 */
public void setCmdDataRouting(Context context, DataRoutingInfo info) {}
```

## Satellite Data Acquisition

### Getting Satellite Count

This section provides supplementary explanations for common issues reported by users, for readers' reference and implementation of required functions. You only need to implement the `getSatelliteUsedNums` method. The code example is as follows:

```java
@Override
public void getSatelliteUsedNums(SatelliteNumber satNumbers) throws RemoteException {
    Log.d(TAG, "Number of visible satellites：" + satNumbers.getSatNum());
    Log.d(TAG, "Number of available satellites：" + satNumbers.getSatUsedNum());
}
```

### Video Stream Data Acquisition

This section introduces how to obtain video stream data from RTK through the SDK. Assuming your RTK device has both ground camera and front camera capabilities, the core steps are as follows:

1. Get camera image data
2. Render image data to the interface

#### Getting Image Data and Rendering to Interface

You can use the RtkCamera class to specify the camera type for obtaining image data, then get each frame of image data by setting callback functions, as follows:

```java
// CameraInfo
private static class CameraInfo {
    ImageView imageView;
    Button button;
    RtkCamera camera;
    Bitmap bitmap;
}

/**
 * @param cameraInfo: camera info
 * @author xuzengbao
 * @description set camera callback
 * @date 2024/11/15 10:09
 */
private void setupCameraCallback(CameraInfo cameraInfo) {
    cameraInfo.camera.setCallback(new IRtkCameraCallback() {
        @Override
        public void onStatusChanged(@NonNull RtkCameraStatus status,
                                    @NonNull RtkCameraParam cameraParam) {
            // Handle state change logic
        }

        @Override
        public void onReceivedPictureSize(int pictureWidth, int pictureHeight) {
            // Logic for processing size changes of received pictures
        }

        @Override
        public void onDataReceived(@NonNull VideoFrame vf) {
            if (cameraInfo.camera.getCameraStatus() != RtkCameraStatus.OPENED) {
                return;
            }

            // get image data
            byte[] imageData = vf.getJpegBuffer();

            if (imageData != null) {
                // Decoding Bitmap
                Bitmap bitmap =
                        BitmapFactory.decodeByteArray(imageData, 0, imageData.length);
                runOnUiThread(() -> {
                    // Recycle old Bitmap to avoid memory leaks
                    if (cameraInfo.bitmap != null && !cameraInfo.bitmap.isRecycled()) {
                        cameraInfo.bitmap.recycle();
                    }
                    cameraInfo.bitmap = bitmap;
                    // update ImageView
                    cameraInfo.imageView.setImageBitmap(bitmap);
                });
            }
        }

        @Override
        public void onError(int error) {
            LogWrapper.printException(new Exception("Camera error code: " + error));
        }
    });
}
```

## Complete Code

### Layout File Code:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    tools:context=".video.VideoStreamActivity">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:background="#000000"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/viewGroundVideo"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"/>

        <ImageView
            android:id="@+id/viewFrontVideo"
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:layout_weight="1"/>

    </LinearLayout>


    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="8dp">

        <!-- Ground Camera Label + Button -->
        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:gravity="center_vertical"
            android:orientation="horizontal">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Ground Camera:" />

            <Button
                android:id="@+id/btnGroundCamera"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="On" />
        </LinearLayout>

        <!-- Front Camera Label + Button -->
        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:gravity="center_vertical"
            android:orientation="horizontal">

            <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:text="Front Camera:" />

            <Button
                android:id="@+id/btnFrontCamera"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:text="On" />
        </LinearLayout>

    </LinearLayout>

</LinearLayout>
```

### Activity Code:

```java
/**
 * @author xuzengbao
 * @description video stream view
 * @date 2024/11/15 09:53
 */
public class VideoStreamActivity extends AppCompatActivity {

    private static class CameraInfo {
        ImageView imageView;
        Button button;
        RtkCamera camera;
        Bitmap bitmap;
    }

    private final Map<RtkCameraType, CameraInfo> mCameraInfos = new HashMap<>();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_video_stream);

        setupCameraInfo(RtkCameraType.GROUND_CAMERA, R.id.viewGroundVideo, R.id.btnGroundCamera);
        setupCameraInfo(RtkCameraType.FRONT_CAMERA, R.id.viewFrontVideo, R.id.btnFrontCamera);
    }

    /**
     * @param type: camera type
     * @param viewId: resource id
     * @param buttonId: resource id
     * @author xuzengbao
     * @description init camera info
     * @date 2024/11/15 10:07
     */
    private void setupCameraInfo(@NonNull RtkCameraType type, int viewId, int buttonId) {
        ImageView imageView = findViewById(viewId);
        Button button = findViewById(buttonId);
        button.setOnClickListener(view -> toggleRendering(type));

        CameraInfo cameraInfo = new CameraInfo();
        cameraInfo.imageView = imageView;
        cameraInfo.button = button;

        mCameraInfos.put(type, cameraInfo);
    }

    /**
     * @param type: camera type
     * @author xuzengbao
     * @description on or off camera
     * @date 2024/11/15 10:08
     */
    private void toggleRendering(@NonNull RtkCameraType type) {
        CameraInfo cameraInfo = mCameraInfos.get(type);
        if (cameraInfo == null) {
            return;
        }

        if (cameraInfo.camera == null) {
            cameraInfo.camera = new RtkCamera(type);
        }

        if (cameraInfo.camera.getCameraStatus() == RtkCameraStatus.OPENED) {
            cameraInfo.button.setText("On");
            closeCamera(cameraInfo.camera);
        } else {
            setupCameraCallback(cameraInfo);
            cameraInfo.camera.open();
            cameraInfo.button.setText("Off");
        }
    }

/**
 * @param cameraInfo: camera info
 * @author xuzengbao
 * @description set camera callback
 * @date 2024/11/15 10:09
 */
private void setupCameraCallback(CameraInfo cameraInfo) {
    cameraInfo.camera.setCallback(new IRtkCameraCallback() {
        @Override
        public void onStatusChanged(@NonNull RtkCameraStatus status,
                                    @NonNull RtkCameraParam cameraParam) {
            // Handle state change logic
        }

        @Override
        public void onReceivedPictureSize(int pictureWidth, int pictureHeight) {
            // Logic for processing size changes of received pictures
        }

        @Override
        public void onDataReceived(@NonNull VideoFrame vf) {
            if (cameraInfo.camera.getCameraStatus() != RtkCameraStatus.OPENED) {
                return;
            }

            // get image data
            byte[] imageData = vf.getJpegBuffer();

            if (imageData != null) {
                // Decoding Bitmap in the background thread
                Bitmap bitmap =
                        BitmapFactory.decodeByteArray(imageData, 0, imageData.length);
                runOnUiThread(() -> {
                    // Recycle old Bitmap to avoid memory leaks
                    if (cameraInfo.bitmap != null && !cameraInfo.bitmap.isRecycled()) {
                        cameraInfo.bitmap.recycle();
                    }
                    cameraInfo.bitmap = bitmap;
                    // update ImageView
                    cameraInfo.imageView.setImageBitmap(bitmap);
                });
            }
        }

        @Override
        public void onError(int error) {
            LogWrapper.printException(new Exception("Camera error code: " + error));
        }
    });
}

    private void closeCamera(@Nullable RtkCamera rtkCamera) {
        if (rtkCamera != null) {
            rtkCamera.close();
        }
    }

    /**
     * @author xuzengbao
     * @description release resources
     * @date 2024/11/15 10:10
     */
    @Override
    protected void onDestroy() {
        super.onDestroy();
        for (CameraInfo cameraInfo : mCameraInfos.values()) {
            if (cameraInfo.camera != null) {
                closeCamera(cameraInfo.camera);
            }
            if (cameraInfo.bitmap != null && !cameraInfo.bitmap.isRecycled()) {
                cameraInfo.bitmap.recycle();
                cameraInfo.bitmap = null;
            }
        }
        mCameraInfos.clear();
    }
}
```