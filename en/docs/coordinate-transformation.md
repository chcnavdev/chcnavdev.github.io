# Coordinate Transformation

This section will introduce the setting functions related to coordinate system transformation, including setting source ellipsoid, target ellipsoid, projection parameters, datum transformation parameters, etc. Here we will introduce the usage of related functions through examples of forward and inverse calculations. For more usage, readers can refer to the related interfaces provided later.

## How to Implement Forward and Inverse Calculations through SDK

### Preparation

Whether it is forward or inverse calculation, coordinate transformation requires initialization and setting related transformation parameters. The specific steps and reference code are as follows:

1. Initialize
2. Set target ellipsoid
3. Set projection
4. Set datum transformation parameters

```java
private void init() throws RemoteException {
    // Step 1: Initialize, must be done first
    mCoordinate.init();
 
    // Step 2: Set the parameters for the coordinate system
    // Set the destination ellipsoid (e.g., Beijing 1954)
    mCoordinate.setDstEllipse(getBeijing54Elps()); 
 
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
private ElpseData getBeijing54Elps() {
    return new ElpseData(1, "BeiJing54(China)", 6378245.0, 298.3);
}
 
// Method to get the projection parameters
private ProjectionData getProjection() {
    return new ProjectionData(40, Math.toRadians(117), 0.0, 0.0, 1.0, 500000.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0, 0.0);
}
```

## Forward and Inverse Calculations

The initialization has been introduced above. Now we will provide the code for forward and inverse calculations respectively.

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
    double[] tgt54 = new double[3];
    
    // Perform forward transformation from source BLH to local NEH
    int errorcode1 = mCoordinate.transformSrcBlhToLocalNeh(src84, tgt54);
    
    // Update UI elements with transformed coordinates and error code
    mTvN.setText(String.valueOf(tgt54[0])); // Northing (N)
    mTvE.setText(String.valueOf(tgt54[1])); // Easting (E)
    mTvH.setText(String.valueOf(tgt54[2])); // Height (H)
    mTvErrorCode.setText(String.valueOf(errorcode1)); // Error code for transformation
}

/**
 * Method to perform inverse transformation (from local NEH to source BLH)
 */
private void antiTrans() throws RemoteException {
    // Initialize arrays for source (WGS84) and target (Beijing 54) coordinates
    double[] src84 = new double[3];
    double[] tgt54 = new double[3];
    
    // Set initial source coordinates (WGS84) to zero
    src84[0] = 0;
    src84[1] = 0;
    src84[2] = 0;
    
    // Set target coordinates (Beijing 54)
    tgt54[0] = 3431025.93297; // Northing (N)
    tgt54[1] = 499943.686606; // Easting (E)
    tgt54[2] = 88.423175; // Height (H)
    
    // Perform inverse transformation from local NEH to source BLH
    int errorcode2 = mCoordinate.transformLocalNehToSrcBlh(tgt54, src84);
    
    // Update UI elements with transformed coordinates and error code
    mTv84B.setText(String.valueOf(Math.toDegrees(src84[0]))); // Convert radians to degrees for latitude (B)
    mTv84L.setText(String.valueOf(Math.toDegrees(src84[1]))); // Convert radians to degrees for longitude (L)
    mTv84H.setText(String.valueOf(src84[2])); // Height (H)
    mTvAntiErrorCode.setText(String.valueOf(errorcode2)); // Error code for transformation
}
```

## Coordinate Transformation Interface

```java
/**
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
boolean setSrcEllipse(ElpseData var1) throws RemoteException;

/**
 * Sets the destination ellipsoid.
 *
 * @param elpse the destination ellipsoid.
 * @return whether the setting is successful
 */ 
boolean setDstEllipse(ElpseData var1) throws RemoteException;

/**
 * Sets the projection parameters.
 * 
 * @param projection the projection parameters
 * @return whether the setting is successful
 */ 
boolean setProjection(ProjectionData var1) throws RemoteException;

/**
 * Sets the datum transformation parameters.
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
boolean setHeightFitting(HeightFittingData hFitting) throws RemoteException;

/**
 * Sets the geoid model parameters.
 * 
 * @param geoidParas the geoid model parameters.
 * @return whether the setting is successful
 */ 
boolean setHeightFittingConvert(GeoidParamsData geoidParas) throws RemoteException;

/**
 * Curve surface fitting (8 parameters).
 * 
 * @param srcxy the source coordinate array
 * @param dsth the target point height array
 */
VerticalAdjResult calCurveSurfaceVerticalAdjParamsEightParams(double[] srcxy, double[] dsth);

/**
 * TGO adjustment.
 *
 * @param srcxy the source coordinate array
 * @param dsth the target point height array
 */
VerticalAdjResult calTgoVerticalAdjParams(double[] srcxy, double[] dsth);

/**
 * Fixed difference adjustment.
 *
 * @param dsth the target point height array
 */
VerticalAdjResult calFixedVerticalAdjParams(double[] dsth);

/**
 * Horizontal adjustment.
 * 
 * @param srcxy the source coordinate array
 * @param dstxy the target coordinate array
 */
HorizAdjResult calTgoHorizAdjParams(double[] srcxy, double[] dstxy);
```