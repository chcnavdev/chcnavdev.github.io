## SatellitePosition

|      类型       |       名称       |                       描述                        |
| :-------------: | :--------------: | :-----------------------------------------------: |
|    Position     |    mPosition     | 包含位置信息 x、y、z（double 类型弧度，对应 BLH） |
| EnumSolveStatus | mEnumSolveStatus |                  包含解状态信息                   |

## EnumSolveStatus

|           名称            |  描述  |
| :-----------------------: | :----: |
|    SOLVE_STATUS_FIX     |  固定  |
|   SOLVE_STATUS_FLOAT    |  浮动  |
|    SOLVE_STATUS_WIDE    |  浮动  |
|    SOLVE_STATUS_COS     |  浮动  |
|    SOLVE_STATUS_RTD     |  浮动  |
|    SOLVE_STATUS_DIFF    |  浮动  |
|    SOLVE_STATUS_NONE    |  未知  |
| SOLVE_STATUS_BASE_WRONG |  未知  |
| SOLVE_STATUS_SEARCH_SAT |  未知  |
| SOLVE_STATUS_STAR_TIMEOUT |  异常  |
| SOLVE_STATUS_STAR_OUTAREA |  异常  |
| SOLVE_STATUS_STAR_SERVER_ERR |  异常  |

除上表所述外，其余所有状态，您可以定义为单点。

## SatellitePrecision

|  类型  |  名称  |      描述      |
| :----: | :----: | :------------: |
| double | mHpre  | 水平方向精度   |
| double | mVpre  | 垂直方向精度   |
| double | mXpre  | 大地纬度精度   |
| double | mYpre  | 大地精读精度   |
| double |  mRms  |   空间精度     |

## NtripProtocolCode

|       类型        |          名称           |   描述   |
| :---------------: | :---------------------: | :------: |
| NtripProtocolCode |    NC_ICY_200_OK       | 登录成功 |
| NtripProtocolCode | ERROR_401_UNAUTHORIZED | 登录失败 |

## EmPdaWorkModeStatus

|        类型         |   名称   |        描述        |
| :-----------------: | :------: | :----------------: |
| EmPdaWorkModeStatus | LOGINING |     正在登录       |
| EmPdaWorkModeStatus |  LOGOUT  |       注销         |
| EmPdaWorkModeStatus | LOGINED  | 已登陆/登录成功    |

## NoneMagneticTiltInfo

|  类型  |     名称      |        描述        |
| :----: | :-----------: | :----------------: |
| double |    mVarE      |  补偿点经度标准差  |
| double |    mVarN      |  补偿点纬度标准差  |
| double |    mVarU      |  补偿点高程标准差  |
| double |   mVarTilt    |  竖直倾角标准差    |
| double |  mVarDirect   |  倾斜方向角标准差  |
| double |  mLatitude    |    补偿点纬度      |
| double | mLontitude    |    补偿点经度      |
| double |   mHeight     |    补偿点高程      |
| double | mHeightOffset |     高程偏移       |
| double |   mDiffAge    |     差分龄期       |
| double |    mPitch     |      俯仰角        |
| double |    mRoll      |      横滚角        |
| double |   mHeading    |      航向角        |

## CoorKernal

|   类型   |   名称    |      描述      |
| :------: | :-------: | :------------: |
| Position |  rawBlh   | 接收机原始输出 |
| Position |  wgsBlh   |     经纬度     |
| Position |  wgsXyz   |    空间坐标    |
| Position | localBlh  |   本地经纬度   |
| Position | localXyz  |   本地XYZ      |
| Position | localNeh  |  本地平面坐标  |

## SurveyPointPositionInfo

|        类型         |        名称         |    描述    |
| :-----------------: | :-----------------: | :--------: |
|     CoorKernal      |    mCoorKernal      | 点坐标集合 |
|       GpsTime       |      mGpsTime       |   卫星时   |
|        Time         |       mTime         |    时间    |
|  SatellitePrecision | mSatellitePrecision |  精度信息  |
|       double        |      diffAge        |  差分龄期  |

## RTKCamera

|        类型         |        名称         |    描述    |
| :-----------------: | :-----------------: | :--------: |
|     CoorKernal      |    mCoorKernal      | 点坐标集合 |
|       GpsTime       |      mGpsTime       |   卫星时   |
|        Time         |       mTime         |    时间    |
|  SatellitePrecision | mSatellitePrecision |  精度信息  |
|       double        |      diffAge        |  差分龄期  |

## BaseWarning

|       类型       |       名称        |   描述   |
| :--------------: | :---------------: | :------: |
| EnumBaseWarning  | mEnumBaseWarning  | 偏移状态 |

## EnumBaseWarning

|          类型           |          名称           |      描述      |
| :---------------------: | :---------------------: | :------------: |
|   BASE_WARNING_NONE     |   BASE_WARNING_NONE     |     未偏移     |
| BASE_WARNING_BASE_MOVE  | BASE_WARNING_BASE_MOVE  | 位置发生偏移   |

