# ALSTutorial

## ALS 项目分析

![](Image/001.png)

如上图所示，为 ALS 项目的表现效果

![](Image/002.png)

上图展示 ALS 项目的控制按钮和介绍

| 按键 | 作用 |
| --- | --- |
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

### 摄像机

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

#### Camera 的动画蓝图

![](Image/022.png)

每帧更新属性状态数值，在根据状态的不同， `Blend` 出各个 `Curve` 的数值

![](Image/023.png)

可以理解为，提前预设出很多套曲线数值模板，根据状态的不同，选择不同的曲线数值

![](Image/035.png)

以上图为例，根据 `RightShoulder` 是否是右键来走不同的 `Modify Curve`，这里 `Modify Curve` 使用的 `Apply Mode` 为 `Scale`，也就是将**乘以**

#### PlayerCameraManager 中的计算

`PlayerCameraManager` 中的计算分文两部分：**初始化** 和 **Tick更新**

在 `OnPossess` 的进行初始化操作

在 `ALS_PlayerCameraManager` 中，当绑定到对象后会触发 `OnPossess` 事件并将绑定的 `Pawn` 作为参数传递

为了方便使用，直接将其值传递给 `Camera` 的动画蓝图

![](Image/019.png)


在 `BlueprintUpdateCamera` 中进行每帧更新的计算操作

![](Image/020.png)

如果控制的角色含义 `ALG_Character` 的 `Tag` 表示这是使用 ALS 动画蓝图的对象，走 `Custom Camera Behavior` 计算，否则走父类的计算流程(也就是默认流程)

所以计算的核心还是在 `Custom Camera Behavior`

![](Image/021.png)

- `RootPoint`: 相机追踪目标(ViewTarget)的Root点，一般就是目标的根节点
- `PivotPoint`：枢纽点，相机会围绕这个点进行旋转
- `LooAtPoint`：看向点，相机会始终注视着这个位置

![](Image/011.png)

- 绿色球为 `RootPoint`，其坐标是 `head` 和 `root` 的中心点
- 红色球为 `Smmothed Pivot Target`，是通过根据 `RootPoint` 插值计算出的平滑点
- 蓝色球为 `Pivot Point` 即珍重的枢纽点，相机围绕该点旋转
- 相机坐标为 `Target Camera Location`，根据 `Pivot Point` 坐标和相机旋转计算出的坐标

其实 ALS 的作者注释写的很全面，在 `Custom Camera Behavior` 函数中分为 8 步

1. 通过接口初始化本次计算的数据
2. 计算相机平滑旋转：通过计算当前相机旋转和控制器旋转的插值
3. 计算相机平滑移动的坐标和朝向
4. 计算注视点
5. 计算相机最终的坐标
6. 通过射线检测，防止摄像机穿模
7. 绘制 Debug 球体
8. 计算最终返回值

> 上面的计算使用到了曲线的数值


![](Image/025.png)

通过接口获取 

- `Pivot Target` 其坐标是 `head` 和 `root` 骨骼坐标的中点、旋转是角色当前旋转角度
- `FPTarget` 是骨骼 `FP_Camera` 点的坐标
- `TPFOV` 和 `FPFOV` 是设定值，默认为 90

![](Image/026.png)

通过相机的朝向和玩家 `Controller` 的朝向插值计算出 `TargetCameraRotation`，插值计算能够让相机旋转更加平滑

`RotationLogSpeed` 是相机骨骼中定义的曲线，在相机的动画蓝图中通过 `Modify Curve` 来控制值，其值为 20，可以修改 `Interp Speed` 值为 1 查看效果变化

![](Image/027.png)

通过当前的 `Pivot Target` 的 `Transform` 和上一帧的 `Smoothed Pivot Target` 配合 `PivotLagSpeed_X`、`PivotLagSpeed_Y`、`PivotLagSpeed_Z` 插值计算出当前` Smoothed Pivot Target` 的 `Location`

![](Image/028.png)

这里需要注意的是 `Calculate Axis Independent Lag` 函数的实现，没太懂 `UnRotate Vector` 和 `Rotate Vector`，但是直接使用 `Current Location`、`Target Location`、`Log Speeds` 进行插值计算，也可以得到相似的效果

![](Image/029.png)

根据 `Smoothed Pivo Taret` 的 `Rotation` 获得当前朝向的前、右、上三个方向的向量，再根据 `PivotOffset_X`、`PivotOffset_Y`、`PivotOffset_Z` 分别计算各个轴的值，再叠加 `Smoothed Pivot Target` 的坐标，计算出真正的 `Pivot Location`

> X 轴是前后、Y 轴是左右、Z 轴是上下

![](Image/030.png)

根据 `Target Camera Rotation` 相机朝向计算出前、右、上三个方向的向量，再根据 `CameraOffset_X`、`CameraOffset_Y`、`CameraOffset_Z` 计算出偏移值，最后叠加到 `Pivot Location` 得到真正的相机坐标

![](Image/031.png)

![](Image/024.png)

如图上图所示完成了第6步，使用射线检测避免了相机穿模和相机在墙后的情况

### 角色动画蓝图

ALS 的动画蓝图是分层流水线结构

![](Image/032.jpg)

- 基础 Pose 层(Base Poses)：确定 **站立**或者**蹲伏**的基础状态
- 状态覆盖层(Overlay Layer)：根据状态选择动画叠加基础 Pose
- 基础动画层(Base Layer)：根据运动状态确定叠加动画

| Base Pose | Overlay Layer | Base Layer |
| --- | --- | --- |
| ![](Image/032.png) | ![](Image/033.png) | ![](Image/034.png) |
| 根据状态选择蹲或者站动画 | 根据 OverlayState 选择动画，按 Q 切换状态 | 根据角色移动信息选择动画 |

由于 `Base Poses` 和 `Overlay Layer` 结构比较简单，不做过多讨论。重点在于状态众多的 `Base Layer`

从输出开始

| Main Movement States | 状态机内容 |
| --- | --- |
| ![](Image/036.png) | ![](Image/037.png) |

#### Base Layer

##### Main Movement States

`Main Movement States` 中将状态分为两类：空中(Air)和地面(Ground)

- 空中
  - Fall：直接走下高点进入空中
  - Jump：通过跳跃进入空中
- 地面
  - Land：落地后如果没有移动、旋转等操作，进入 Land，根据下降速度播放落地动画
  - Land Movement：落地后如果有移动速度则会进入该状态，将落地动画与角色动画混合
  - Grounded: 地面动画，当角色站在地面上时，无论走跑都是该状态（翻滚不算，翻滚在 `RagDoll States` 中）

默认进入 `Grounded` 状态，当跳跃的时候 `Movement State` 会从 `Grounded` 进入 `InAir`，状态机也会进入 `InAir` 再根据是否是 `Jumped` 来选择进入 `Jump` 或者 `Fall`

![](Image/039.png)

当 `Jump` 的时候执行执行，在等待 0.1s 之后将 `Jumped` 设置为 `false` 因为状态机那个时候已经完成切换了

###### Fall

![](Image/040.png)

`Fall` 分为两大块：空中动画和落地动画

根据下降速度播放空中动画，`FallLoop`、`FallLoop_Fast`、`Flail`，然后叠加到 `Lean_Falling` 的偏移动画（前倾、后倾、左倾、右倾）得到最后的空中动画

根据下降速度将 `Land_heavy` 和 `Land_Light` 进行混合，得到落地动画

最后根据 `LandPrediction` 将空中对话与落地动画进行混合，并输出最后的结果

`LandPrediction` 的计算在 `Calculate Land Prediction` 函数中

![](Image/041.png)

![](Image/042.png)

如果下降速度小于 200(大于 -200)，则 `LandPrediction` 为0，表示使用 空中动画

如果下降速度大于 200(小于 -200)，则根据速度进行胶囊体检测，如果没有检测到物体，则 `LandPrediction` 为 0；如果检测到物体，则根据 `Time` 从 `Land Prediction Curve` 曲线中取值

`Hit Result` 中 `Time` 值为 **Time = Distance / WholeDistance**， `Distance` 为 `Trace Start` 到 `Location` 的距离， `WholeDistance` 为 `Trace Start` 到 `Trace End` 的距离

![](Image/043.png)

> `Land Prediction Curve` 是一个递减函数

射线检测到的点越近，则 `Time` **越小**，对应的曲线值**越大**，那么落地动画的权重**更大**

###### Jump

![](Image/044.png)

Jump 与 Fall 状态有很多地方相似，都是将空中与落地动画进行混合，重要的其实在 `Jump States` 状态机中

![](Image/045.png)

根据跳跃起步时**左脚**还是**右脚**，分别走左脚跳或者右脚跳，然后进入跳跃动画，最后进入下落动画循环

那么 ALS 是如何判断起步脚是左脚还是右脚呢？

| 动画曲线 | Condition |
| --- | --- |
| ![](Image/046.png) | ![](Image/047.png) |

以 `ALS_N_Run_F` 为例，动画中存在一个名为 `Feet_Position` 的曲线，当曲线值小于 0 为左脚，大于 0 为右脚

###### Grounded

![](Image/048.png)

直接使用了 `Main Grounded States` Pose

##### Main Grounded States

![](Image/038.png)

`Main Grounded States` 接受 `Main grounded States` 状态机的输出

![](Image/049.png)

状态切换逻辑很明显

`Entry` 是一个空白状态， 空白节点用于在输入时拾取姿态（基于权重最大的BasePose动画曲线）。它还使用固定条目状态枚举来输入条目覆盖状态，以从其他操作（如滚动）转换

根据进入 Entry 时 `GroundedEntryState` 的值选择 `From Roll` 或者 `Standing` 或者 `Crouching LF`

> `Crouching` 蹲伏

左边的 `Transition` 是站立到蹲伏，右边的 `Transition` 是蹲伏到站立

由于 `From Roll` 状态播放的 `ALS_N_LandRoll_F` 动画最后一帧是站立状态，所以需要通过 `N->CLF Transition` 从站立切换到蹲伏

![](Image/050.png)

那么核心的 `Standing` 和 `Crouching LF` 分别对应的就是上图的上半部分和下半部分

`Standing` 中将站立分为三个层次：基础状态、状态细节、动作循环

| 基础状态 | 状态细节 | 动作循环 |
| --- | --- | --- |
| ![](Image/051.png) | ![](Image/052.png) | ![](Image/053.png) |
| 将站立分为移动和未移动，其中未移动的时候可以进入旋转状态 | 在 `Moving` 时可以根据移动细节进行很多状态的划分，走、跑、开始走、开始跑、第一次反向跑、第二次反向跑 | 在 `Moving` 和 `Waling` 的时候动画是循环的，这个时候根据在状态选择动画并设置参数即可 |

`Crouching` 中将蹲伏分为两个层次：基础状态、动作循环

| 基础状态 | 动作循环 |
| --- | --- | 
| ![](Image/054.png) | ![](Image/055.png) |
| `Crouching` 的基础状态和 `Standing` 的状态细节内容相似 | 根据当前状态，选择移动动画 |

> 这里的一个细节在于八方向移动

| 动画循环 | 动画层 | Directional States |
| --- | --- | --- |
| ![](Image/057.png) | ![](Image/058.png) | ![](Image/056.png) |

#### 原地旋转

![](Gif/001.gif)

如上图 Gif 所示，但相机朝向与玩家朝向超过一定值之后，玩家会自动旋转到相机的方向

![](Image/059.png)

这个功能是在动画蓝图中 `Event Blueprint Update Animation` 中每帧执行的

[旋转讲解](https://zhuanlan.zhihu.com/p/454841949)

> 旋转部分后面的内容几乎都是对该文章的复制

##### Turn In Place

![](Image/060.png)

在 `ALS_AnimBP` 的 `UpdateGraph` 事件视图中，会每帧的区判断 `CanTurnInPlace` 也就是原地旋转

![](Image/061.png)

条件判断是 `Enable_Transition` 的值大于 0.99 以及当前是第三人称相机

至于 `Enable_Transition` 曲线值的设置，是在 `CLF Not Moving` 和 `N Not Moving` 状态，也就是站立且不移动和蹲下且不一定两个状态

| N Not Moving | CLF Not Moving |
| --- | --- |
| ![](Image/062.png) | ![](Image/063.png) |

当其他状态时，`Enable_Transition` 值为 0，也就是不能触发自动旋转

> `Enable_Transition` 之前是直接判断 >= 1，但是有误差，改成了 0.99

前面 `CanTurnInPlace` 是判断旋转的**前置条件**是否满足

后面 `Turn In Place Check` 是中进行具体数值的判断，判断旋转的**条件**是否满足

| 旋转角度检查 | 旋转时间检查 |
| --- | --- 
| ![](Image/064.png) | ![](Image/065.png) |

- `Aim Yaw Rate` 这一帧与上一帧 `Controller` 的 `Yaw` 轴旋转角度的偏差，当该值大于 `Aim Yaw Rate Limit` 值表示当前玩家仍然在高速移动相机，此时不用旋转，等待玩家确定相机朝向后再开始旋转
- `Aiming Angle` 就是 `Character` 和 `Controller` 角度差距，当 `Yaw` 轴偏移超过 `Turn Check Min Angle` 的时候才会触发后续旋转
- `Elapsed Delay Time` 延迟旋转时间，当符合旋转角度条件的时候，每帧加上 `Delta Time`，当该值大于指定值的时候，表示时间条件完成，开始旋转

这里注意 `Map Range Clamped` 函数的使用，函数参数 `InRange` 值为 `(45, 180)`， `OutRange` 值为 `(0.75, 0)`，也就是说 **旋转角度** `Aiming Angle` 越大，则函数返回值越小，**旋转角度** 越小则函数返回值越大

通过上面对 `Map Range Clamped` 的使用，实现了**旋转角度**越大则越快**触发旋转**的效果

最后在 `Turn In Place` 真正开始触发旋转

| 选择动画 | 播放动画 | 计算旋转速率 |
| --- | --- | --- |
| ![](Image/066.png) | ![](Image/067.png) | ![](Image/068.png) |

![](Image/069.png)

注意一下 `Turn In Place Asset Type` 结构体的内容，包括：动画资产、旋转角度、`Slot`槽名称、动画播放速率、是否修改旋转速率

以 `N_TurnIP_L90` 预设的属性值为例，这里的槽名称为 `(N) Turn/Rotate`，对应前面截图 `N Not Moving` 状态中使用的槽的名称

![](Image/070.png)

也就是当使用 `Play Slot Animation As Dynamic Montage` 节点播放动画时，会使用 `(N) Turn/Rotate` 槽，当角色触发 `Turn In Place` 旋转时，不会执行前面的 `Modify Curve`， 此时 `Enable_Transition` 曲线的值为 0，避免了 `Turn In Place` 的时候再次触发 `Turn In Place`

![](Image/071.png)

注意 `(N) Turn/Rotate` 槽后面的 `Modify Curve` 节点，将之前计算出来的 `Rotation Scale` 乘上 `Rotation Amount` 曲线

![](Image/072.png)

曲线配置在旋转动画上，使用曲线配置是为了能够在旋转的时候根据动画匹配旋转角度，而不是以一个固定的旋转角度去改变角色角度

至此，角色动画蓝图中操作和计算就完成了

在角色的蓝图 `ALS_Base_CharacterBP` 中的计算在 `Update Grouded Rotation` 里面，改函数在 `Tick` 中每帧调用

![](Image/073.png)

> 动画帧率为 30

#### Rotate In Place

这是在第一人称或者瞄准状态时，相机角度与角色朝向存在偏差时触发的旋转

在 `Rotate In Place` 的时候，根据角度偏差计算计算旋转方向和旋转速率 `Rotate Rate`

![](Image/074.png)

在 `Rotate In Place` 中计算出朝左、朝右旋转，然后根据 bool 值进入对应的状态

| 旋转状态 | 曲线、速率设置 |
| --- | --- |
| ![](Image/075.png) | ![](Image/076.png) |

最后依然触发角色蓝图 `ALS_Base_CharacterBP` 的 `Update Grouded Rotation` 函数

![](Image/077.png) 

> 这里与 `Turn In Place` 一样触发添加角色旋转，并且根据 `RotationAmount` 控制旋转速率

##### 瞄准偏移

![](Gif/002.gif)

在常见的射击游戏中，我们希望角色的头可以朝向相机所指的方向，而不是直直的看着前面

在 `ALS` 中使用动画混合 (Blend Space 1D) 来制作摇头动画

![](Image/078.png)

这个混合动画看着只有一个轴，来角色头的 `Pitch` 旋转，其实这个动画动起来的时候头会从左到右旋转，所以动画的时间轴可以确定角色头部的 `Yaw`

![](Image/079.png)

整个动画的时间是 1s，也就是说动画从 0 ~ 0.5s 是头从左到中，动画的 0.5s ~ 1s 是头从中到右，通过 `Map Range Clamped` 将 `Yaw` 角度从 -180 ~ 180 映射到 0 ~ 1 之间，最后得到的值就是 `ALS_N_Look` 动画轴的值

![](Image/080.png)

| | 左转 | 右转 | 向前看 |
| --- | --- | --- | --- |
| 条件 | ![](Image/081.png) | ![](Image/082.png) | ![](Image/086.png) |
| 节点内容 | ![](Image/083.png) | ![](Image/084.png) | ![](Image/085.png) |

当瞄准偏移角度超过 180° 的时候，会进入到 `SwitchSidesBlendPos` 状态

![](Image/087.png)

这个状态会强制将头朝中间

#### 分层叠加 LayerBlending

[ALSv4分层叠加系统](https://zhuanlan.zhihu.com/p/662774779)

![](Image/088.jpg)

