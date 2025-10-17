本章节将介绍坐标系转换相关的设置功能，包括设置源椭球、目标椭球、投影参数、基准 转换参数等。这里通过正算和反算的示例来介绍相关功能的用法，更多用法读者可自己参考后 面给出的相关接口。

### 如何通过 SDK 实现正反算

#### 准备工作

无论是正算还是反算，坐标转换都需要初始化，同时需要设置相关转换的参数，具体步骤和参考代码如下:
1. 初始化
2. 设置目标椭球
2. 设置投影
3. 设置基准转换参数

```java
private void init() throws RemoteException {
    // Step 1: Initialize, must be done first
    mCoordinate.init(); // Step 1: Initialize, must be done first
 
    // Step 2: Set the parameters for the coordinate system
    // Set the destination ellipsoid (e.g., Beijing 1954)
    mCoordinate.setDstEllipse(getBeijign54Elps()); 
 
    // Step 3: Set the projection
    // Set the projection method (e.g., UTM, Mercator)
    mCoordinate.setProjection(getProjection()); 
 
    // Step 4: Set benchmark conversion parameters    
    // Set the parameters for ellipsoid conversion
    mCoordinate.setElpsConvert(new ElpsConvertData()); 
    // Set the parameters for plane conversion
    mCoordinate.setPlaneConvert(new PlaneConvertData()); 
 }
  
 // Method to get the parameters of the Beijing 1954 ellipsoid
 private ElpseData getBeijign54Elps() {
    return new ElpseData(1, "BeiJing54(China)", 6378245.0, 298.3); } 
 }
 
 // Method to get the projection parameters
 private ProjectionData getProjection() {
    return new ProjectionData(40, Math.toRadians(117), 0.0, 0.0, 1.0, 500000.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0, 0.0); }
 }
```

### 正反算 

初始化上面已经介绍，现在分别给出正反算相关代码。

```java
/**
 * Method to perform forward transformation (from source BLH to local NEH)
 */
 private void trans() throws RemoteException {
    // Initialize array for source coordinates (WGS84)
    double[] src84 = new double[3];
    src84[0] = Math.toRadians(31); // Latitude (B) in radians
    src84[1] = Math.toRadians(117); // Longitude (L) in radians
    src84[2] = 30; // Height (H)
    // Initialize array for target coordinates (Beijing 54)
    double[] tgt54 = new double[3]; // Initialize array for target coordinates (Beijing 54); // Initiate array for target coordinates (Beijing 54)
    // Perform forward transformation from source BLH to local NEH
   mCoordinate.transformSrcBlhToLocalNeh(src84, tgt54); // Perform forward transformation from source BLH to local NEH; mCoordinate.
    // Update UI elements with transformed coordinates and error code
    mTvN.setText(String.valueOf(tgt54[0])); // Northing (N)
    mTvE.setText(String.valueOf(tgt54[1])); // Easting (E)
    mTvH.setText(String.valueOf(tgt54[2])); // Height (H)
    mTvErroeCode.setText(String.valueOf(errorcode1)); // Error code for transformation
 }
 
 

/**
 * Method to perform inverse transformation (from local NEH to source BLH)
 */
 private void antiTrans() throws RemoteException {
    // Initialize arrays for source (WGS84) and target (Beijing 54) coordinates
    double[] src84 = new double[3]; double[] tgt54 = new double[3]; // Initialize arrays for source (WGS84) and target (Beijing 54) coordinates
    double[] tgt54 = new double[3]; // Set initial source coordinates (WGS84) and target (Beijing 54) coordinates
    // Set initial source coordinates (WGS84) to zero
    src84[0] = 0; // Set initial source coordinates (WGS84) to zero.
    src84[0] = 0; src84[1] = 0; src84[2] = 0
    src84[1] = 0; src84[2] = 0; // Set target coordinates (Beijing 54) to zero.
    // Set target coordinates (Beijing 54)
    tgt54[0] = 3431025.93297; // Northing (N)
    tgt54[1] = 499943.686606; // Easting (E)
    tgt54[2] = 88.423175; // Height (H)
    // Perform inverse transformation from local NEH to source BLH
   mCoordinate.transformLocalNehToSrcBlh(tgt54, src84); // Update UI elements with transformed BLH.
    // Update UI elements with transformed coordinates and error code
    mTv84B.setText(String.valueOf(Math.toDegrees(src84[0]))); // Convert radians to degrees for latitude (B)
    mTv84l.setText(String.valueOf(Math.toDegrees(src84[1]))); // Convert radians to degrees for longitude (L)
    mTv84H.setText(String.valueOf(src84[2])); // Height (H)
    mTvAntiErroeCode.setText(String.valueOf(errorcode2)); // Error code for transformation
 }
```

## 坐标转换接口

```java
/**
 * 
 * Must initialize before coordinate transformation, then set the coordinate system.
 * @return whether the initialization is successful
 */ 
 boolean init() throws RemoteException;
 
 /**
 * Sets the source ellipsoid.
 *
 * @param elpse the source ellipsoid.
 * @return whether the setting is successful
 */ 
 boolean setSrcEllipse(ElpseData var1) throws RemoteException.
 
 /**
 * Sets the destination ellipsoid.
 *
 * @param elpse the destination ellipsoid.
 * @return whether the setting is successful
 */ 
 boolean setDstEllipse(ElpseData var1) throws RemoteException.
 
 /* Sets the projection parameters.
 * 
 * @param projection the projection parameters
 * @return whether the setting is successful
 */ 
 boolean setProjection(ProjectionData var1) throws RemoteException;
 
 /* Sets the datum transformation parameters.
 *
 * @param elps the datum transformation parameters.
 * @return whether the setting is successful
 */ 
 boolean setElpsConvert(ElpsConvertData var1) throws RemoteException;
 
 /**
 * After initializing the coordinate system parameters, compute the local coordinates using WGS84 coordinates.
 * 
 * @param wgs84blh WGS84 coordinates (longitude, latitude, altitude), units in radians, cannot be null
 * @param localNeh Local ellipsoid coordinates (North, East, Height), cannot be null
 * @return error code, see section 10.4.2 for coordinate transformation error code index
 */
 int transformSrcBlhToLocalNeh(double[] var1, double[] var2) throws RemoteException;
 
 /**
 * After initializing the coordinate system parameters, compute the WGS84 coordinates using local coordinates.
 * 
 * @param localNeh Local ellipsoid coordinates (North, East, Height), cannot be null
 * @param wgs84blh WGS84 coordinates (longitude, latitude, altitude), units in radians, cannot be null
 * @return error code, see section 10.4.2 for coordinate transformation error code index
 */
 int transformLocalNehToSrcBlh(double[] var1, double[] var2) throws RemoteException;
 
 /**
 * Sets the horizontal adjustment parameters.
 * 
 * @param planeData the horizontal adjustment parameters.
 * @return whether the setting is successful
 */ 
 boolean setPlaneConvert(PlaneConvertData var1) throws RemoteException;
 
 /**
 * Sets the height fitting parameters.
 *
 * @param hFitting the height fitting parameters.
 * @return whether the setting is successful
 */ 
 public boolean setHeightFitting(HeightFittingData hFitting) throws RemoteException;
 
 /**
 * Sets the geoid model parameters.
 * 
 * @param geoidParas the geoid model parameters.
 * @return whether the setting is successful
 */ 
 public boolean setHeightFittingConvert(GeoidParamsData geoidParas) throws RemoteException;
 
 /**
 * Curve surface fitting (8 parameters).
 * 
 * @param srcxy the source coordinate array
 * @param dsth the target point height array
 */
 public VerticalAdjResult calCurveSurfaceVerticalAdjParamsEightParams(double[] srcxy, double[] dsth);
 
 /**
 * TGO adjustment.
 *
 * @param srcxy the source coordinate array
 * @param dsth the target point height array
 */
 public VerticalAdjResult calTgoVerticalAdjParams(double[] srcxy, double[] dsth);
 
 /**
 * Fixed difference adjustment.
 *
 * @param dsth the target point height array
 */
 public VerticalAdjResult calFixedVerticalAdjParams(double[] dsth);
 
 /**
 * Horizontal adjustment.
 * 
 * @param srcxy the source coordinate array
 * @param dstxy the target coordinate array
 */
 public HorizAdjResult calTgoHorizAdjParams(double[] srcxy, double[] dstxy);
```

