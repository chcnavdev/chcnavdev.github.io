# AR Stakeout

This section aims to help developers quickly integrate and use AR stakeout functionality. The core module that the SDK depends on is business. By integrating ArViewLayout and ArPointStakeoutController, developers can implement AR-related stakeout functions. This document will focus on how to configure, initialize, and use related APIs, while also covering view layout and UI component configuration.

## Layout

In the layout file, ArViewLayout is used to host the AR view. This layout will provide a screen-occupying area for the AR view, and subsequent AR operations will be displayed through ArViewLayout. The specific implementation is as follows:

```xml
<com.huace.gnssserver.business.api.ar.ArViewLayout
    android:id="@+id/viewAr"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1" />
```

## Initialize AR Functionality

The steps and code for creating and initializing ArPointStakeoutController are as follows:

1. In the onCreate method, first initialize ArViewLayout and create an ArPointStakeoutController instance
2. ArPointStakeoutController is used to control AR view operations and manage AR-related stakeout functions
3. The constructor of ArPointStakeoutController receives two parameters: Context and ArViewLayout, and binds IArModeListener to listen for AR mode status changes

```java
private void initAr() throws RemoteException {
    ArViewLayout arViewLayout = findViewById(R.id.viewAr);
    mArPointStakeoutController = new ArPointStakeoutController(
            this, 
            arViewLayout, 
            new IArModeListener() {
                @Override
                public void onBeforeEnter() {
                    // Callback before entering AR mode
                }

                @Override
                public void onAfterExit() {
                    // Callback after exiting AR mode
                }
            }
    );
}
```

## Setting AR Operations

Common AR operations and code are as follows:

1. Open/Close Camera: ArPointStakeoutController provides open() and close() methods for opening and closing the camera in the AR view respectively
2. Set Target Point: Using the setTargetPointOfSrcBlh() method, you can set the target point in the AR view. The target point is passed through Point3DMutable

```java
// Turn the camera on or off
findViewById(R.id.openCamera).setOnClickListener(view -> {
    mArPointStakeoutController.open();
});

// Close camera
findViewById(R.id.closeCamera).setOnClickListener(view -> {
    mArPointStakeoutController.close();
});
```

## Camera Switching

The ArSettings class is a configuration class used to store and retrieve distance settings related to AR mode, namely the automatic distance threshold for camera switching. This class implements global configuration through static variables and static methods, and these configurations can be used for operations in other parts of the project, such as automatic camera mode switching. Usage scenarios are as follows:

1. Automatic Camera Switching: During the AR stakeout process, when the distance between the user's device and the target point changes, ArSettings allows the system to automatically switch between the front camera and the ground camera to ensure optimal AR display effects. For example, when the device is far from the target, the system can automatically switch to the front camera
2. Debugging and Optimization: During development, camera switching can be conveniently debugged by adjusting these two distance thresholds

The following is sample code for convenient debugging during development:

```java
// Camera switch
findViewById(R.id.switchCamera).setOnClickListener(view -> {
    double dist = mArPointStakeoutController.getDist();
    if (dist > ArSettings.getFarDistOfAutoSwitchToArMode()) {
        ArSettings.setFarDistOfAutoSwitchToArMode(dist + 20);
        return;
    }
    if (dist > ArSettings.getGroundDistOfAutoSwitchToArMode()) {
        ArSettings.setGroundDistOfAutoSwitchToArMode(dist + 2);
    } else {
        ArSettings.setGroundDistOfAutoSwitchToArMode(dist - 2);
    }
});
```

The ArSettings class provides flexible automatic camera switching configuration for AR stakeout applications. By setting groundDistOfAutoSwitchToArMode and farDistOfAutoSwitchToArMode, developers can control when to switch to the ground camera or front camera, thereby providing a better user experience.

## Complete Code

```java
public class ArStakeoutActivity extends AppCompatActivity {

    private TextView tvB;
    private TextView tvL;
    private TextView tvH;
    private TextView tvPositionState;

    private ArPointStakeoutController mArPointStakeoutController;
    private Point3DMutable mPoint3DMutable;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_ar_stakeout);
        initAr();
        initView();
        initClickListener();
    }

    private void initAr() {
        ArViewLayout arViewLayout = findViewById(R.id.viewAr);
        mArPointStakeoutController =
                new ArPointStakeoutController(this, arViewLayout, new IArModeListener() {
                    @Override
                    public void onBeforeEnter() {
                    // Callback before entering AR mode
                    }

                    @Override
                    public void onAfterExit() {
                    // Callback after exiting AR mode
                    }
                });
    }

    private void initView() {
        tvB = (TextView) findViewById(R.id.tvB);
        tvL = (TextView) findViewById(R.id.tvL);
        tvH = (TextView) findViewById(R.id.tvH);
        tvPositionState = (TextView) findViewById(R.id.tvPositionState);
        try {
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
        } catch (RemoteException e) {
            throw new RuntimeException(e);
        }
    }

    private void initClickListener() {
        // Convenient debugging, switching between the ground camera and the front camera with one click
        findViewById(R.id.switchCamera).setOnClickListener(view -> {
            // Automatically switch to the front camera when the distance is between
            // ArSettings.getGroundDistOfAutoSwitchToArMode() and ArSettings.getFarDistOfAutoSwitchToArMode().
            // Automatically switch to the ground camera when the distance is less than
            // ArSettings.getGroundDistOfAutoSwitchToArMode().
            // The following logic ensures automatic switching between the ground camera and the front camera.
            double dist = mArPointStakeoutController.getDist();
            if (dist > ArSettings.getFarDistOfAutoSwitchToArMode()) {
                // Update the far distance threshold to allow further switching when the distance increases
                ArSettings.setFarDistOfAutoSwitchToArMode(dist + 20);
                return;
            }
            if (dist > ArSettings.getGroundDistOfAutoSwitchToArMode()) {
                // Slightly increase the ground distance threshold for smoother switching
                ArSettings.setGroundDistOfAutoSwitchToArMode(dist + 2);
            } else {
                // Slightly decrease the ground distance threshold for smoother switching
                ArSettings.setGroundDistOfAutoSwitchToArMode(dist - 2);
            }
        });

        findViewById(R.id.setTarget).setOnClickListener(view -> {
            if (mPoint3DMutable != null) {
                Point3DMutable target = new Point3DMutable(mPoint3DMutable);
                mArPointStakeoutController.setTargetPointOfSrcBlh(target);
            }
        });

        findViewById(R.id.openCamera).setOnClickListener(view -> {
            mArPointStakeoutController.open();
        });

        findViewById(R.id.closeCamera).setOnClickListener(view -> {
            mArPointStakeoutController.close();
        });
    }

    @Override
    protected void onResume() {
        super.onResume();
        mArPointStakeoutController.onResume();
    }

    @Override
    protected void onPause() {
        super.onPause();
        mArPointStakeoutController.onPause();
    }

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
}
```

