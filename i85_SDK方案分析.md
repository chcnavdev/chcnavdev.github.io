i85激光测量SDK封装方案

LandStar中激光相关业务流程

![](https://blog-1256273063.cos.ap-nanjing.myqcloud.com/202511251549641.png)

![](https://blog-1256273063.cos.ap-nanjing.myqcloud.com/202511251555514.png)





## 暴露的接口

### gnssserver层

**LaserDeviceManager.aidl**

	- **openLaser()** # 打开激光，对用户隐藏预览路和计算路的概念
	- **closeLaser()** # 关闭激光

**LaserDeviceListener**

	- onInitStatus() # 初始化状态
	- onLaserPointInfo() # 获取激光点位置信息（含激光距离、坐标(px&pos)、精度、错误码等）

### business层（可选）

**LaserViewLayout** ：视频流+激光十字丝视图

