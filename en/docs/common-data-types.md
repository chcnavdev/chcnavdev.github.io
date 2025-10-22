# Common Data Types

## SatellitePosition

|      Type       |       Name       |                       Description                        |
| :-------------: | :--------------: | :------------------------------------------------------: |
|    Position     |    mPosition     | Contains position information x, y, z (double type radians, corresponding to BLH) |
| EnumSolveStatus | mEnumSolveStatus |                  Contains solution status information                   |

## EnumSolveStatus

|           Name            |  Description  |
| :-----------------------: | :-----------: |
|    SOLVE_STATUS_FIX       |     Fixed     |
|   SOLVE_STATUS_FLOAT      |     Float     |
|    SOLVE_STATUS_WIDE      |     Float     |
|    SOLVE_STATUS_COS       |     Float     |
|    SOLVE_STATUS_RTD       |     Float     |
|    SOLVE_STATUS_DIFF      |     Float     |
|    SOLVE_STATUS_NONE      |    Unknown    |
| SOLVE_STATUS_BASE_WRONG   |    Unknown    |
| SOLVE_STATUS_SEARCH_SAT   |    Unknown    |
| SOLVE_STATUS_STAR_TIMEOUT |   Abnormal    |
| SOLVE_STATUS_STAR_OUTAREA |   Abnormal    |
| SOLVE_STATUS_STAR_SERVER_ERR |  Abnormal  |

In addition to the statuses described in the above table, all other statuses can be defined as single point.

## SatellitePrecision

|  Type  |  Name  |      Description      |
| :----: | :----: | :-------------------: |
| double | mHpre  | Horizontal accuracy   |
| double | mVpre  | Vertical accuracy     |
| double | mXpre  | Geodetic latitude accuracy |
| double | mYpre  | Geodetic longitude accuracy |
| double |  mRms  |   Spatial accuracy    |

## NtripProtocolCode

|       Type        |          Name           |   Description   |
| :---------------: | :---------------------: | :-------------: |
| NtripProtocolCode |    NC_ICY_200_OK       | Login successful |
| NtripProtocolCode | ERROR_401_UNAUTHORIZED | Login failed    |

## EmPdaWorkModeStatus

|        Type         |   Name   |        Description        |
| :-----------------: | :------: | :-----------------------: |
| EmPdaWorkModeStatus | LOGINING |       Logging in          |
| EmPdaWorkModeStatus |  LOGOUT  |        Logout             |
| EmPdaWorkModeStatus | LOGINED  | Logged in/Login successful |

## NoneMagneticTiltInfo

|  Type  |     Name      |        Description        |
| :----: | :-----------: | :-----------------------: |
| double |    mVarE      | Compensation point longitude standard deviation |
| double |    mVarN      | Compensation point latitude standard deviation  |
| double |    mVarU      | Compensation point elevation standard deviation |
| double |   mVarTilt    | Vertical tilt angle standard deviation         |
| double |  mVarDirect   | Tilt direction angle standard deviation        |
| double |  mLatitude    |    Compensation point latitude      |
| double | mLontitude    |    Compensation point longitude     |
| double |   mHeight     |    Compensation point elevation     |
| double | mHeightOffset |     Elevation offset       |
| double |   mDiffAge    |     Differential age       |
| double |    mPitch     |      Pitch angle          |
| double |    mRoll      |      Roll angle           |
| double |   mHeading    |      Heading angle        |

## CoorKernal

|   Type   |   Name    |      Description      |
| :------: | :-------: | :-------------------: |
| Position |  rawBlh   | Receiver raw output   |
| Position |  wgsBlh   |     Longitude and latitude     |
| Position |  wgsXyz   |    Spatial coordinates    |
| Position | localBlh  |   Local longitude and latitude   |
| Position | localXyz  |   Local XYZ      |
| Position | localNeh  |  Local plane coordinates  |

## SurveyPointPositionInfo

|        Type         |        Name         |    Description    |
| :-----------------: | :-----------------: | :---------------: |
|     CoorKernal      |    mCoorKernal      | Point coordinate set |
|       GpsTime       |      mGpsTime       |   Satellite time   |
|        Time         |       mTime         |    Time    |
|  SatellitePrecision | mSatellitePrecision |  Precision information  |
|       double        |      diffAge        |  Differential age  |

## RTKCamera

|        Type         |        Name         |    Description    |
| :-----------------: | :-----------------: | :---------------: |
|     CoorKernal      |    mCoorKernal      | Point coordinate set |
|       GpsTime       |      mGpsTime       |   Satellite time   |
|        Time         |       mTime         |    Time    |
|  SatellitePrecision | mSatellitePrecision |  Precision information  |
|       double        |      diffAge        |  Differential age  |

## BaseWarning

|       Type       |       Name        |   Description   |
| :--------------: | :---------------: | :-------------: |
| EnumBaseWarning  | mEnumBaseWarning  | Offset status   |

## EnumBaseWarning

|          Type           |          Name           |      Description      |
| :---------------------: | :---------------------: | :-------------------: |
|   BASE_WARNING_NONE     |   BASE_WARNING_NONE     |     No offset         |
| BASE_WARNING_BASE_MOVE  | BASE_WARNING_BASE_MOVE  | Position offset occurred |