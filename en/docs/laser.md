# Laser Measurement

This section helps developers quickly integrate and use laser measurement. Laser-related classes live in the **`gnssserver`** module under package **`com.huace.gnssserver.sdk.laser`** (`LaserDeviceManager`, `LaserPositionListener`, `LaserPosition`, `EnumLaserStatus`). Your app must depend on an artifact that includes this SDK. These APIs provide real-time laser ranging and position updates. `LaserPositionListener` also reports **camera open progress** (`RtkCameraStatus`) and **video errors**. This document covers configuration, initialization, API usage, data models, and status codes.

## Prerequisites

Before using laser measurement, ensure the following:

1. **Receiver communication is healthy** — Laser data depends on a working link to the receiver. Keep the link available per your product requirements (you may call existing WiFi / data-link status APIs for pre-checks; the demo page `LaserDeviceActivity` currently only checks laser support).
2. **Receiver supports laser measurement** — Confirm the connected receiver hardware supports laser. Use `ReceiverUtils.isSupportLaser()`; if unsupported, `openLaser()` returns `false`.
3. **Receiver is fixed with position output** — Ranging results depend on the receiver’s solved pose. Without a fix you may see no data, unstable data, or accuracy below expectations.

```java
if (!ReceiverUtils.isSupportLaser()) {
    // Inform the user that this receiver does not support laser measurement
    return;
}
```

## Layout

A typical laser screen has control buttons (start / stop / get latest) and a data display area. Example layout:

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <Button
        android:id="@+id/btnStartLaser"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Start laser" />

    <Button
        android:id="@+id/btnStopLaser"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Stop laser" />

    <Button
        android:id="@+id/btnGetLatest"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Get latest data" />

    <TextView
        android:id="@+id/tvLaser"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="12dp"
        android:textSize="14sp" />

</LinearLayout>
```

## Initializing laser

`LaserDeviceManager` is a singleton; do not construct your own instance. Initialization steps:

1. Obtain the global instance via `LaserDeviceManager.getInstance()`;
2. Register a `LaserPositionListener` for position updates, video errors, and camera open status;
3. Call `openLaser()` to start laser measurement;
4. Call `ReceiverUtils.isSupportLaser()` (and any link / connection checks your app needs) before starting.

### Preview path and computation path

The current implementation **always opens two paths in sequence**: first the **preview path** (`FRONT_PREVIEW_LASER_CAMERA`), then after the preview path’s **`onDataReceived` delivers the first frame**, the **high-resolution computation path** (`FRONT_HIGH_QUALITY_LASER_CAMERA`). This avoids timing issues from relying only on `onStatusChanged` for readiness.

- Data from **`onLaserPositionChanged` / `getLatestLaserPosition()` always comes from the high-resolution computation path** via `onDataReceived`; the preview path does not deliver ranging results to callers.
- **`onLaserVideoError`**: Called when either the preview or computation path fires `onError`. **The underlying path is not distinguished**; only an error code is provided.
- **`onLaserOpenStatusChanged`** (aligned with `LaserPositionListener` / `LaserDeviceManager`; forwarding rules below):
    - After `openLaser()` successfully submits the open flow, **one** `RtkCameraStatus.OPENING` is dispatched (start flow submitted);
    - **Preview path** `onStatusChanged`: All states except **`OPENED`** are forwarded as-is (when the preview path itself reaches `OPENED`, it is **not** exposed through this callback);
    - **Computation path** `onStatusChanged`: Only **`OPENED`** is forwarded; other states during open (e.g. `OPENING`, `OPEN_FAILED`) are **not** exposed from the computation path via this method. Semantically, **`OPENED` means the preview path can produce frames and the high-resolution computation path is open** (the computation path starts only after the preview path’s **first `onDataReceived`**, so `OPENED` implies both paths are ready);
    - For computation-path failures, use `onLaserVideoError` plus your own timeout / retry logic.

```java
import com.huace.gnssserver.sdk.laser.LaserDeviceManager;
import com.huace.gnssserver.sdk.laser.LaserPosition;
import com.huace.gnssserver.sdk.laser.LaserPositionListener;
import com.huace.gnssserver.sdk.receiver.ReceiverUtils;
import com.huace.gnssserver.sdk.receiver.camera.RtkCameraStatus;

private void initLaser() {
    // 1. Check whether the device supports laser measurement
    if (!ReceiverUtils.isSupportLaser()) {
        // Device does not support laser; inform the user
        return;
    }

    // 2. Create listener (implement all three methods)
    LaserPositionListener listener = new LaserPositionListener() {
        @Override
        public void onLaserPositionChanged(@NonNull LaserPosition position) {
            // Called when laser position updates (from high-resolution computation path)
            // Callback thread matches the underlying delivery thread; switch to main thread for UI
            runOnUiThread(() -> updateUI(position));
        }

        @Override
        public void onLaserVideoError(int errorCode) {
            // Preview or computation path video error; path is not distinguished
            Log.w(TAG, "Laser video error code=" + errorCode);
        }

        @Override
        public void onLaserOpenStatusChanged(@NonNull RtkCameraStatus status) {
            // Overall open progress (see "Preview path and computation path" above)
            Log.d(TAG, "Laser open status: " + status);
        }
    };

    // 3. Register listener and start (duplicate addListener with the same instance is ignored)
    LaserDeviceManager.getInstance().addListener(listener);
    LaserDeviceManager.getInstance().openLaser();
}
```

## Laser operations

`LaserDeviceManager` exposes these core operations:

### Start / stop laser measurement

```java
// Start laser measurement
findViewById(R.id.btnStartLaser).setOnClickListener(v -> {
    if (!ReceiverUtils.isSupportLaser()) {
        Toast.makeText(this, "Device does not support laser measurement", Toast.LENGTH_SHORT).show();
        return;
    }
    LaserDeviceManager.getInstance().addListener(mListener);
    LaserDeviceManager.getInstance().openLaser();
});

// Stop laser measurement
findViewById(R.id.btnStopLaser).setOnClickListener(v -> {
    LaserDeviceManager.getInstance().removeListener(mListener);
    LaserDeviceManager.getInstance().closeLaser();
});
```

### Fetch latest laser data

```java
findViewById(R.id.btnGetLatest).setOnClickListener(v -> {
    LaserPosition position = LaserDeviceManager.getInstance().getLatestLaserPosition();
    if (position == null) {
        Log.d(TAG, "No laser data yet");
    } else {
        Log.d(TAG, "Latest laser data: " + position.toString());
    }
});
```

## Data model

`LaserPosition` holds a full laser measurement payload. Main fields:

| Field | Type | Description |
|------|------|------|
| `timestampMs` | `long` | Local timestamp (milliseconds) |
| `laserDistance` | `double` | Measured distance |
| `laserLat` | `double` | Laser point latitude |
| `laserLon` | `double` | Laser point longitude |
| `laserHgt` | `double` | Laser point height |
| `laserHardAccuracy` | `double` | Hard accuracy; values below 1.0 are often acceptable (tune per product) |
| `laserAcc` | `double` | Accuracy factor; **values below 0.6 are often acceptable** (empirical threshold; adjust per product) |
| `laserScaling` | `double` | Scale factor |
| `laserXPre` | `int` | Laser error in X |
| `laserYPre` | `int` | Laser error in Y |
| `laserZPre` | `int` | Laser error in Z |
| `laserXInImagePx` | `double` | Laser point X in image (pixels) |
| `laserYInImagePx` | `double` | Laser point Y in image (pixels) |
| `laserStatus` | `EnumLaserStatus` | Device status |
| `laserError` | `int` | Raw fault code |

## RtkCameraStatus

Defined in `com.huace.gnssserver.sdk.receiver.camera`, `RtkCameraStatus` describes **RTK camera device** state during the open flow. Values in `LaserPositionListener#onLaserOpenStatusChanged` come from **one manager-injected `OPENING`** plus **preview / computation path forwarding per “Preview path and computation path”** (not the full state stream of a single camera instance). Enum meanings match the underlying layer.

| Value | Description |
|--------|------|
| `UN_OPENED` | Device not open |
| `UN_SUPPORT` | Device not supported |
| `UN_CONNECTED` | Not connected to RTK |
| `CONNECTION_WAY_NOT_SUPPORT` | Current connection type not supported |
| `PARAMETER_GET_FAILED` | Failed to read camera parameters |
| `OPENING` | Camera opening |
| `OPEN_FAILED` | Open failed |
| `DEVICE_IS_OCCUPIED` | Camera in use (often another controller is connected to the receiver) |
| `OPENED` | Opened |

The SDK also provides `isOpeningOrOpened()`, which returns `true` for `OPENING` or `OPENED`, useful to detect “opening or already open”.

## Status codes

`EnumLaserStatus` defines laser device states. **Numeric codes** match the same-named enum on the LandStar8 side (different package); unknown raw codes map to `OTHER_UNKNOWN`.

| Value | Code | Description |
|--------|------|------|
| `NORMAL` | `0` | Normal |
| `DATA_ABNORMAL` | `1000` | Abnormal data |
| `SIGNAL_WEAK` | `1001` | Weak signal |
| `OVER_RANGE` | `1002` | Out of range |
| `SIGNAL_TOO_STRONG` | `1003` | Signal too strong |
| `LD_ERROR` | `1004` | Laser diode fault |
| `BACKGROUND_LIGHT_STRONG` | `1005` | Background light too strong |
| `HARDWARE_ERROR` | `1006` | Hardware error |
| `LIGHT_LEAKAGE` | `1007` | Light leakage |
| `REFLECTIVITY_ABNORMAL` | `1008` | Abnormal reflectivity |
| `TEMPERATURE_ABNORMAL` | `1009` | Abnormal temperature |
| `OTHER_UNKNOWN` | `1100` | Other / unknown error |

Use `position.getLaserStatus()` for the current status and `getLaserStatus().getCode()` for the raw code, or `position.getLaserError()` for the raw fault code.

## Lifecycle

`LaserDeviceManager` manages preview and high-resolution computation resources internally. Release them at the right lifecycle points:

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    LaserDeviceManager.getInstance().removeListener(mListener);
    LaserDeviceManager.getInstance().closeLaser();
}
```

## Complete example

The in-repo demo is `com.huace.gnsstest.laser.LaserDeviceActivity` (including `AlertDialog` when laser is unsupported and `strings.xml` copy). Below is the same core logic for reference or copy into your module.

```java
import android.app.AlertDialog;
import android.os.Bundle;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;

import com.huace.gnssserver.sdk.laser.LaserDeviceManager;
import com.huace.gnssserver.sdk.laser.LaserPosition;
import com.huace.gnssserver.sdk.laser.LaserPositionListener;
import com.huace.gnssserver.sdk.receiver.ReceiverUtils;
import com.huace.gnssserver.sdk.receiver.camera.RtkCameraStatus;
import com.huace.gnsstest.R;

public class LaserDeviceActivity extends AppCompatActivity {

    private final LaserPositionListener mListener = new LaserPositionListener() {
        @Override
        public void onLaserPositionChanged(@NonNull LaserPosition position) {
            renderPosition(position);
        }

        @Override
        public void onLaserVideoError(int errorCode) {
            renderText(getString(R.string.laser_error_video, errorCode));
        }

        @Override
        public void onLaserOpenStatusChanged(@NonNull RtkCameraStatus status) {
            renderText(status.name());
        }
    };

    private TextView mTvLaser;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_laser_survey);
        mTvLaser = findViewById(R.id.tvLaser);
        findViewById(R.id.btnStartLaser).setOnClickListener(v -> {
            if (!ReceiverUtils.isSupportLaser()) {
                new AlertDialog.Builder(this)
                        .setTitle(getString(R.string.laser_dialog_title))
                        .setMessage(getString(R.string.laser_not_supported))
                        .setPositiveButton(getString(R.string.laser_dialog_ok), null)
                        .show();
                return;
            }
            LaserDeviceManager.getInstance().addListener(mListener);
            LaserDeviceManager.getInstance().openLaser();
        });
        findViewById(R.id.btnStopLaser).setOnClickListener(v -> {
            LaserDeviceManager.getInstance().removeListener(mListener);
            LaserDeviceManager.getInstance().closeLaser();
            renderText(getString(R.string.laser_stopped));
        });
        findViewById(R.id.btnGetLatest).setOnClickListener(v -> {
            LaserPosition position = LaserDeviceManager.getInstance().getLatestLaserPosition();
            if (position == null) {
                renderText(getString(R.string.laser_latest_null));
            } else {
                renderPosition(position);
            }
        });
    }

    @Override
    protected void onDestroy() {
        LaserDeviceManager.getInstance().removeListener(mListener);
        LaserDeviceManager.getInstance().closeLaser();
        super.onDestroy();
    }

    private void renderPosition(@NonNull LaserPosition position) {
        String text = "timestampMs=" + position.getTimestampMs()
                + "\ndistance=" + position.getLaserDistance()
                + "\nlat=" + position.getLaserLat()
                + "\nlon=" + position.getLaserLon()
                + "\nh=" + position.getLaserHgt()
                + "\nhardAccuracy=" + position.getLaserHardAccuracy()
                + "\nacc=" + position.getLaserAcc()
                + "\nscaling=" + position.getLaserScaling()
                + "\nxPre=" + position.getLaserXPre()
                + "\nyPre=" + position.getLaserYPre()
                + "\nzPre=" + position.getLaserZPre()
                + "\nerror=" + position.getLaserError()
                + "\nlaserXInImagePx=" + position.getLaserXInImagePx()
                + "\nlaserYInImagePx=" + position.getLaserYInImagePx()
                + "\nstatus=" + position.getLaserStatus().name()
                + "(" + position.getLaserStatus().getCode() + ")";
        renderText(text);
    }

    private void renderText(@NonNull String text) {
        runOnUiThread(() -> mTvLaser.setText(text));
    }
}
```

## API reference

### LaserDeviceManager

| Method | Return | Description |
|------|--------|------|
| `getInstance()` | `LaserDeviceManager` | Singleton instance |
| `openLaser()` | `boolean` | Start laser: `false` if receiver does not support laser; **`true` if already running (idempotent)**; `true` when open flow is first submitted successfully |
| `closeLaser()` | `void` | Stop laser, close cameras, clear cached latest position; no-op if not started |
| `getLatestLaserPosition()` | `@Nullable LaserPosition` | Latest position (from high-resolution computation path); `null` if none yet or stopped |
| `addListener(LaserPositionListener)` | `void` | Register listener; **same instance is not added twice** |
| `removeListener(LaserPositionListener)` | `void` | Unregister listener |

### LaserPositionListener

| Method | Description |
|------|------|
| `onLaserPositionChanged(LaserPosition)` | Position update; data from high-resolution computation path |
| `onLaserVideoError(int errorCode)` | Preview or computation video error; **path not distinguished**, error code only |
| `onLaserOpenStatusChanged(RtkCameraStatus status)` | Overall open progress (see **“Preview path and computation path”**); enum meanings in **“RtkCameraStatus”** |

> **Note:** `onLaserPositionChanged`, `onLaserVideoError`, and `onLaserOpenStatusChanged` run on the same thread as the underlying camera / data delivery. Switch to the main thread yourself when updating UI.
