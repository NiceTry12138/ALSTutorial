# ALSTutorial

![](Image/001.png)

如上图所示，为 ALS 项目的表现效果

![](Image/002.png)

上图展示 ALS 项目的控制按钮和介绍

| 按键 | 作用 |
| 1 | 切换为速度方向模式 |
| 2 | 切换为相机控制方向模式（默认） |
| Z | 慢动作 |
| V | 固定相机方向，方便查看角色状态 |
| T | 显示射线检测 |
| Y | 显示碰撞和方向（角色朝向、速度方向、加速度方向） |
| U | 显示身体各个部分的混合动画程度 |
| I | 展示细节属性面板 |

下面的就是角色移动控制

新建 `GameMode`、`CharacterBase`、`PlayerController`

设置 `CharacterBase` 的碰撞预设为 `ALS_Character`(ALS自带的，直接从ALS项目复制)，胶囊体高度设置为 90、半径设置为 30

![](Image/003.png)

> `MovementComponent` 如上修改
> **空气控制**：指的是在空中对角色移动的控制（即空中控制前进后退），地面值为1

继承 `CharacterBase` 创建 `ALS_Character` 的子类用作玩家，设置 Tag 为 `ALS_Character`；关闭 **控制器旋转Yaw**

## 摄像机

3C：`Camera`、`Character`、`Controller`，即相机、角色和控制器

`Camera` 摄像机需要关注两个点：相机坐标、相机朝向。通过设置这坐标和朝向可以决定我们最终得到的表现效果

`ALS` 项目中使用曲线和玩家骨骼坐标来定位相机坐标，实现相机平滑移动的效果

之前在角色的 `Class Default` 中关闭了 **控制器旋转Yaw** 是为了后面能够单独控制相机旋转

继承 `PlayerCameraManager` 创建 `ALS_PlayerCameraManager`，并添加 `ALS` 项目中自带的 `Camera` 骨骼网格体

使用 `Camera` 骨骼网格体是为了方便的使用骨骼中定义的曲线

![](Image/004.png)

> 上图为 ALS 项目中 Camere 骨骼定义的曲线

最后将 `ALS_PlayerCameraManager` 设置到 `ALS_PlayerController` 中

![](Image/005.png)

摄像机系统利用**骨骼空间**计算出某一个位置，将这个位置坐标转换到**世界坐空间**上

新建 `ALS_Camera_BPI` 用于定义相机接口，由 `BaseCharacter` 来实现这些接口

> 子类可以重写这些接口的实现

```cpp
// 函数名 参数 => 返回类型

// 获取相机的参数 TP_FOV 第三人称的 FOV，FP_FOV 第一人称的 FOV，bRightShoulder 是否在右肩
BPI_Get_CameraParameters() => float TP_FOV, float FP_FOV, float bRightShoulder

// 获取相机朝向目标 ReturnValue 目标坐标
BPI_Get_FP_CameraTarget() => FVector ReturnValue

// 获取第三人称锚点目标的 Transform
BPI_Get_3P_PivotTarget() => FTransform ReturnValue

// 获取第三人称的检测参数 
BPI_Get_3P_TraceParams() => FVector TraceOrigin, float TraceRadius, ETraceType TraceChannel
```
| 函数名 | 实现 |
| --- | --- |
| BPI_Get_CameraParameters | ![](Image/010.png) |
| BPI_Get_FP_CameraTarget | ![](Image/008.png) |
| BPI_Get_3P_PivotTarget | ![](Image/009.png) |
| BPI_Get_3P_TraceParams | ![](Image/007.png) | 

在 `ALS_Character` 中对上述函数进行重写

| 函数名 | 函数体 |
| --- | --- |
| BPI_Get_3P_PivotTarget | ![](Image/012.png) |
| BPI_Get_FP_CameraTarget | ![](Image/013.png) |
| BPI_Get_3P_TraceParams | ![](Image/014.png) |

![](Image/006.png)

`ALS` 的添加了一些虚拟骨骼用于绑定和定位

上面这些接口都是方便计算 `Camera` 相机的坐标和朝向的

曲线有多种添加方式：可以新建 Curve 曲线、可以在动画资产中添加曲线、可以在骨骼中添加曲线

`ALS` 的相机在骨骼中添加曲线，在 `Curves` 框中直接右键就可以添加曲线

![](Image/015.png)

这里定义的曲线，在动画蓝图中可以直接获取并对值进行 `Blend`

![](Image/016.png)

![](Image/017.png)

> 在 `Modify Curve` 节点上右键可以添加曲线和设置曲线的值

![](Image/018.png)

> 在蓝图中可以通过 `Get Curve Value` 来获取曲线的值

**综上所述，在 `Camera` 的骨骼中定义曲线名称，在动画蓝图中计算曲线对应的值，在 `PlayerCameraManager` 中根据曲线的值和目标坐标进行相机的坐标和朝向计算**

### 坐标计算

