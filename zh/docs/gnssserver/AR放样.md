本节旨在帮助开发者快速集成并使用AR放样功能。SDK依赖的核心模块是business，通过集成ArViewLayout和ArPointStakeoutController，开发者能够实现与AR相关的放样功能。文档将重点介绍如何配置、初始化以及使用相关API，同时也会涉及到视图的布局和UI组件的配置。

## 布局

在布局文件中，ArViewLayout用于承载AR视图。该布局将为AR视图提供一个占据屏幕的区域，后续的AR操作将通过ArViewLayout展示。具体如下：

```java
<com.huace.gnssserver.business.api.ar.ArViewLayout
    android:id="@+id/viewAr"
    android:layout_width="match_parent"
    android:layout_height="0dp"
    android:layout_weight="1" />
```

## 初始化AR功能

创建并初始化ArPointStakeoutController的步骤和代码如下：

1. 在onCreate方法中，首先初始化ArViewLayout并创建ArPointStakeoutController实例；
2. ArPointStakeoutController用于控制AR视图的操作，并管理与AR相关的放样功能；
3. ArPointStakeoutController的构造函数接收两个参数：Context和ArViewLayout，并绑定了IArModeListener，以便监听AR模式的状态变化；

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

## 设置AR操作

关于AR常见的操作和代码如下：

1. 打开/关闭相机：ArPointStakeoutController提供了open()和close()方法，分别用于打开和关闭AR视图中的相机；
2. 设置目标点：使用setTargetPointOfSrcBlh()方法，可以设置AR视图中的目标点。目标点通过Point3DMutable传递；

```java
// Turn the camera on or off
findViewById(R.id.openCamera).setOnClickListener(view -> {
    mArPointStakeoutController.open();
});

// Set target points
findViewById(R.id.closeCamera).setOnClickListener(view -> {
    mArPointStakeoutController.close();
});
```

## 相机切换

ArSettings 类是一个配置类，用于存储和获取与AR模式相关的距离设置，即摄像头切换的自动距离阈值。这个类通过静态变量和静态方法来实现全局配置，可以在项目中的其他地方使用这些配置进行操作，比如自动切换相机模式等，使用场景如下：

1. 自动相机切换：在AR放样过程中，当用户的设备与目标点的距离发生变化时，ArSettings 允许系统自动切换前摄相机和下摄相机，以确保AR显示效果的最佳化。例如，当设备距离目标较远时，系统可以自动切换到前
2. 调试与优化：在开发过程中，可以通过调整这两个距离阈值来便捷调试相机的切换；

如下为开发过程中便捷调试的示例代码：

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

ArSettings类为AR放样应用提供了灵活的自动切换相机的配置。通过设置groundDistOfAutoSwitchToArMode和farDistOfAutoSwitchToArMode，开发者可以控制何时切换到下摄相机或前摄相机，从而提供更好的用户体验。

## 完整代码

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

