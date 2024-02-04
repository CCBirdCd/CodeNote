# 效果实现分类
## Sprite
Sprite 是指在游戏引擎中，使用一个 2D 图片来渲染 3D 场景中的物体。Sprite 可以用于渲染各种 2D 图像，比如角色、道具、UI 等等。
在 Unreal Engine 5 中，Sprite 可以通过创建 Sprite 粒子系统来使用。在粒子系统中，Sprite 可以用来表示粒子的外观，比如火花、爆炸碎片、雨滴等等。
Sprite 可以通过材质实现多种效果，比如混合、发光、透明等等。在 Unreal Engine 5 中，Sprite 的材质可以使用 Material Editor 进行创建和编辑，可以通过修改材质参数来调整 Sprite 的效果。
## Beam
在UE5中，Beam是指一种效果，通常用于模拟激光、电力等粒子发射效果，表现为一条从起点到终点的可视化线条，其宽度、颜色、透明度等属性可以自由调节。Beam可以用Niagara模块实现。在Niagara模块中，Beam通过Emitter产生粒子发射，并且使用Beam Renderer渲染器进行可视化呈现。可以使用Beam的属性控制Beam的外观、位置、方向和其他特性。Beam Renderer还支持多种排序模式，以确保Beam的正确呈现顺序。
当 **Spawn Count** 的数量小于2时，由于粒子之间没有连线，因此看不到 **Beam** 效果。**Beam** 需要至少两个粒子才能形成线段。当 **Spawn Count** 数量小于2时，可以考虑通过增加 **Spawn Rate** 来提高粒子数量以展示 **Beam** 效果。
# Render
## Sorting
Sprite Render 的 Sorting（排序）是指决定在屏幕上渲染的精灵（Sprite）的显示顺序。在 UE4 中，Sprite Render 可以用于 2D 游戏中的图像渲染，而 Sorting 的作用就是决定哪些精灵应该在前面，哪些精灵应该在后面。
# Spawn Modules（生成模块）
[https://zhuanlan.zhihu.com/p/400113639?utm_id=0](https://zhuanlan.zhihu.com/p/400113639?utm_id=0)
--这个模块主要是用于粒子的生成，包含了一次性生成，每帧生成，每单位生成，生成速率。
![[Image/Niagara基础属性介绍/590d2a5ad5c1b915eb9c645fb81d5be7_MD5.png]]
## Spawn Rate（生成频率）
![[Image/Niagara基础属性介绍/1a942ef68855db1f22281f5319e93c1b_MD5.png]]
SpawnRate：生成速率：每秒生成的粒子的数量
Spawn Probablity：生成概率：每个粒子的生成概率
SpawnGroup：生成组：相同ID（组）就可以驱动相同ID(组）的属性（位置偏移，颜色，age等）。比如
![[Image/Niagara基础属性介绍/fb79b5ba8f73bb62c1fa7ee81d781215_MD5.png]]
![[Image/Niagara基础属性介绍/7778d6f89b946ca4deb04c4e1c3e9a9c_MD5.png]]
![[Image/Niagara基础属性介绍/4c06f5191dcca1582761dc81317227f3_MD5.png]]
![[Image/Niagara基础属性介绍/6a7e3da4c46d38cb088ae30acd62257d_MD5.png]]
![[Image/Niagara基础属性介绍/78a9f96721121b8f0593c140030b8222_MD5.png]]
得到的效果
![[Image/Niagara基础属性介绍/08023c4856091daf1ec4caaad765f88d_MD5.png]]
## Spawn Burst Instantaneous（一次性生成）
![[Image/Niagara基础属性介绍/046ddcc425a4fb71ed30045ef1358861_MD5.png]]
功能：一次性生成粒子
SpawnCount：生成粒子的数量
SpawnTime：生成粒子的延迟时间(时间需要小于Emitter State的LoopDuration 循环持续时间)
## Initalize Particle(粒子初始化)
### Point Attributes(点属性)
-- LifeTime Mode：生命周期模式，分为DirectSet(直接设置) 和 Randmom(范围随机)
-- Color Mode：
Unset
Direct Set(直接设置指定颜色)
Rnadom Range(指定颜色范围随机 )
ColorChannelMode：
![[Image/Niagara基础属性介绍/651717677ad260657cbe41bbea266722_MD5.png]]
Link RGBA：一个单独的随机值被用作线性选取最大值和最小值之间的颜色包括Alpha值，随机值受rgba四个参数的影响( 如果从(0,0,0,1) 随机到(1,1,1,1) 那么只能随机到百色，原因是 alpha只能从 1 随机到 1)  
随机结果=差值*百分比数值+最小值
Link RGBA/Link A：两个参数决定效果，一个参数在rgb中随机决定颜色随机值，另一个参数在alpha中随机决定头透明度 随机出两个百分比数值，RGB随机结果=差值*第一个百分比数值+最小值，Alpha随机
    结果=差值*第二个百分比数值+最小值 
Random Individual Channels：每个通道参数(R G B A)都在最大最小值范围内随机不受其他参数的影响，随机独立通道：各数值通过独立随机获得 作者：Winter惜曦
-- Position Mode：位置模式(UnSet、Direct Set、Simulation Range)
Simulation Position：模拟位置，返回当前世界坐标或本地位置
-- Mass Mode：质量模式(UnSet、Direct Set、Random)
Unset/Mass of 1：不设置/1质量：实际效果为忽略质量对速度的影响
### Sprite Attributes(图片属性)
-- Sprite Size Mode（图片大小设置包括轴向XY，设置不统一大小时需要设置两个轴向的参数来控制大小）
![[Image/Niagara基础属性介绍/a970bbbc04edecf4693923305915d7eb_MD5.png]]
UnSet：不设置
Uniform：统一：根据设置数值统一对图片的长和宽进行缩放
Random Unifoem：随机统一：根据随机数统一对图片的长和宽进行缩放 作
Non-Unifoem：不统一：根据设置数值分别缩放图片的长和宽 
Random Non-Unifoem：随机不统一：根据随机数分别缩放图片的长和宽
-- Sprite Rotation Mode（图片旋转模式设置）
![[Image/Niagara基础属性介绍/49c0060f30cd88d0cd591609d1c08988_MD5.png]]
UnSet：不设置
Random：随机朝向
Direct Angle(Degree)：直接设置旋转度
Direct Normalized Angle(0-1)：直接设置角度，数据标准化 和 上面的参数对应的有转换规则（1对应旋转360度）
-- Sprite UV Mode（图片纹理模式）
![[Image/Niagara基础属性介绍/8cc12670e8e8aa09a2e4f348c16a7225_MD5.png]]
UnSet：
RandomX：
RandomY：
RamdomX/Y：
Custom：
### Mesh Attributes（网格属性）
-- Mesh Scale Mode：网格体缩放模式
![[Image/Niagara基础属性介绍/50b1a24cabfd39f2cc3ff902744a317f_MD5.png]]
-- Mesh Renderer Array Visibility：网格体渲染器数组可见性模式
![[Image/Niagara基础属性介绍/993fae67fe64df433c676aadf3486086_MD5.png]]
### Ribbon Attributes（条带属性）
![[Image/Niagara基础属性介绍/8047560c336981f44c3248f4b20bc251_MD5.png]]
Ribbon Attributes 是 Initialized Particle 模块中的一个子模块，专门用于初始化 Ribbon 粒子的属性
**-- Ribbon Width Mode：**条带宽度模式
**-- Ribbon Facing Mode：**条带朝向模式
**-- Rbbon Twist  Moe：**条带扭曲模式

## Spawn Beam(生成光束)
Spawn Beam是在Niagara粒子系统中生成beam（光束）的模块之一。它可以将beam实例化为一个Niagara Actor，用于渲染效果。在使用Spawn Beam模块时，您需要指定beam的起点、终点、颜色、粗细等参数，从而定义beam的外观。您可以在Niagara编辑器中使用Spawn Beam模块，并且可以通过链接到其他Niagara模块来控制beam的属性和行为。
### Spline Position Error Threshold(曲线位置错误检查精度)
在 Unreal Engine 5 中，Spline Position Error Threshold 是一个设置 Beam Emitter 的参数，用于控制 Beam 在 Spline 上的位置精度。
在 Beam Emitter 中，当使用 Spline 作为路径时，Beam 会沿着 Spline 路径生成。Spline Position Error Threshold 参数控制 Beam 在 Spline 上的精度，以避免出现过多或过少的节点。
这个参数表示 Spline 上相邻两个控制点之间的最大距离，超过这个距离 Beam 就会添加一个新的控制点。当 Beam 的 Spline 路径设置不够平滑时，增加这个参数可以让 Beam 更加平滑。反之，如果路径已经很平滑了，可以降低这个参数，以减少控制点的数量。
需要注意的是，降低这个参数可能会影响性能，因为生成的控制点数量越多，需要处理的数据也就越多。因此，在实际应用中需要平衡精度和性能的关系。
# Emitter Update
## Beam Emitter Setup
在 UE5 中，Beam Emitter Setup 是一种用于创建和配置 Beam 特效的工具。它可以在 Niagara 粒子系统中使用，用于创建类似于激光、电力、火焰等效果的 Beam 特效。
通过 Beam Emitter Setup，可以调整 Beam 的各种属性
粒子沿着 beam 的运动轨迹，是因为它们受到了 beam 的影响。在 UE5 的 Beam 系统中，beam 是由多个顶点连接而成的，每个顶点可以施加不同的属性和影响，比如位置、缩放、旋转、颜色等。当粒子在运动时，如果它们与 beam 相交，就会受到 beam 所施加的影响，例如被拉伸、旋转、改变颜色等，从而沿着 beam 的路径运动。这就是 Beam 系统的基本工作原理。
Spline Position Error Threshold 参数的默认值是0.000001，这个值控制了在粒子沿着spline路径移动时，Niagara是否应该重新计算它们在spline上的位置。如果Niagara检测到粒子当前的位置与在spline上的位置之间的距离大于或等于这个值，则它将重新计算粒子在spline上的位置。如果距离小于这个值，则Niagara假定粒子仍然在spline上并继续其运动。
这个默认值0.000001是相对较小的值，因为对于大多数情况下的粒子运动，需要精确地跟踪它们在spline上的位置，特别是在路径被弯曲或变化的地方，因为那些地方的误差可能会更大。如果您的场景中的粒子在spline上移动的速度非常慢，那么您可以考虑适当增加Spline Position Error Threshold的值，以提高性能，但是这可能会影响粒子的准确性。
 **-- Beam Start**：Beam Start（Beam 起点）是指 Beam 效果的起始点。在 UE5 中，Beam Start 可以通过 Beam Emitter 的属性进行设置。通常情况下，Beam Start 默认为 Spawn Point（生成点），也就是粒子生成的位置。
 **-- Absolute Beam Start：**在 UE5 的 Beam Emitter 中，Absolute Beam Start 是一个用于设置 Beam 开始位置是否为绝对位置的选项。如果选中了 Absolute Beam Start，那么 Beam 的起点将会被设置为在世界坐标系中的绝对位置。如果没有选中，Beam 的起点将会被设置为在发射该 Beam 的粒子相对于粒子系统的本地位置。
换句话说，Absolute Beam Start 的作用是决定 Beam 起点的位置是相对于粒子系统本地坐标系还是世界坐标系。如果选择了绝对位置，那么无论粒子系统在哪里，Beam 的起点都会在同一个位置，否则 Beam 的起点会随着粒子系统的移动而改变位置。
实际效果：勾选完成后粒子将始终从发射器原点位置开始发射，但是移动后已发射的光束不会随之移动
**-- Absolute Beam End：**Absolute Beam End是一个bool类型的参数，勾选后表示Beam的结束点位置使用的是绝对世界坐标，而不是相对于Emitter的位置偏移。当勾选后，Beam的结束点会一直保持在指定的世界坐标位置，无论Emitter的位置偏移如何。
如果你设置了Absolute Beam End参数并且在发射器移动时Beam的结束点仍然保持在原来的位置，那么很可能是其他设置出现了问题，比如Spline Position Error Threshold设置不合适，或者Beam的Spline Point没被正确设置。
 -- Use Beam Tangents："Use Beam Tangents" 是 Beam Emitter 中的一个选项，用于控制 Beam 的朝向是否随着发射器的移动而发生改变。当这个选项被勾选时，Beam 会根据发射器的运动方向自动更新其方向，因此 Beam 的朝向会随着发射器的移动而发生改变。如果未勾选这个选项，则 Beam 的方向将始终指向它的目标点，不受发射器的影响。需要注意的是，如果启用了此选项，则 Beam 的方向将始终指向 Beam 的目标点，而不是发射器的目标点。这意味着如果您需要对 Beam 的方向进行更细粒度的控制，可能需要关闭此选项并手动控制 Beam 的方向。
# Particle Update（粒子更新）
## Sprite Facing And Alignment
粒子朝向和对齐设置：默认粒子图片朝向是朝向着摄像机的位置，需要更改为 Custom Facing Vector(自定义朝向向量才可以生效)
![[Image/Niagara基础属性介绍/97c55e757bfab410fbebdb120c02171a_MD5.png]]
## Sprite Size Scale（图片大小缩放）
![[Image/Niagara基础属性介绍/ed6fbd105eedb7fbbfe52b6b7cb61ce1_MD5.png]]
Uniform：统一，使用统一系数缩放图片的长和宽
Uniform Curve：统一曲线，使用同一曲线数值缩放图片的X和Y轴
Non-Uniform：不统一，使用不同的数值缩放图片的X和Y轴
Non-Uniform Curve：不统一曲线，分别从两条曲线获得数值缩放图片的X轴和Y轴
Uniform Curve Index：统一曲线的X轴坐标
Uniform Curve Scale：统一曲线缩放数值 ，图片实际大小=图片初始大小*曲线浮点*曲线缩放数值
# Location Modules（位置分布模块）
## Sphere Location（球型分布）
![[Image/Niagara基础属性介绍/5cbfca941183af03d79e6ffb7c7cc0ad_MD5.png]]
-- Shape（形状）
![[Image/Niagara基础属性介绍/32d05fa7f69a1c65e58c0fcf5c997a9b_MD5.png]]
Shape Primitive：分布形状选择，根据选择不同的分布形状可以配置不同的参数，例如选择Sphere分布则配置Sphere Radius，选择Box则配置矩形
Sphere Primitive：球体半径
Apply To Particle Position：应用到粒子位置
-- Distribution（分布参数配置，根据Shape Primitive切换不同配置参数）
Sphere Surface Distribution：球体分布模式
Random：随机分布
![[Image/Niagara基础属性介绍/15f5104d4291ad7d59d483351d5adbbd_MD5.png]]
Direct：直接分布
![[Image/Niagara基础属性介绍/b0fff35e95ef022eb0028a46da4387c0_MD5.png]]
Uniform：统一分布
![[Image/Niagara基础属性介绍/2024fd97efb324f2b7c175485cb4948f_MD5.png]]
  
# Velocity Modules（速度模块）
## Add Velocity（施加力）
Velocity Mode:速度模式 Linear：线性模式 	 From Point：从点发散速度	In Cone：圆锥发散
### Linear（线性模式 ）
![[Image/Niagara基础属性介绍/ba3e31d0ea5426d0ffd6f51ef60cab23_MD5.png]]
线性添加速度，按照指定的速度向量方向给 生成的粒子添加速度
### From Point（从指定点到粒子原点发散速度）
![[Image/Niagara基础属性介绍/90330e1d92458da9d973064f65e48791_MD5.png]]
在控件中定义任意一点（通过 OriginOffset来定下位置，如果OriginOffset为零则随机定义速度方向），通过该点和粒子原点 定义速度矢量吗，如果粒子位置和速度原点相互重叠或者十分接近，模块将注入随机速度，为了获得最准确的效果最好将此模块放在位置模块之后，以便粒子位置初始化
-- Constrain To Radius：限制粒子收到的速度作用的半径范围如果 Constrain To Radius 被设置为一个非零的值，则只有距离 FromPoint 点的距离小于等于该值的粒子才会受到速度的作用，而距离大于该值的粒子则不会受到速度的作用。这个参数的主要作用是控制粒子受到速度作用的范围，从而实现更加精细的粒子控制
Radius Falloff Near/Far：指定速度影响的衰减范围，对于距离 **FromPoint** 的近处粒子，它们会受到更大的速度影响，而距离远的粒子速度影响则更小。具体来说，**Falloff Near** 控制粒子从 **FromPoint** 到半径范围内的粒子位置之间的速度影响的衰减程度，而 **Falloff Far** 控制半径范围之外的粒子受到的速度影响的衰减程度。一般来说，这两个参数可以用来调整速度影响的平滑程度，使其看起来更自然和真实。
Radius Falloff Exponent：是控制速度随距离变化的指数，如果该值大于 1，则速度随距离增加而减小得更快；如果该值小于 1，则速度随距离增加而减小得更慢。具体来说，如果 Radius Falloff Near 和 Radius Falloff Far 分别表示离 FromPoint 最近和最远的距离，当前粒子与 FromPoint 的距离为 d，那么粒子的速度系数将根据以下公式计算：
f = clamp((d - Radius) / (Radius Falloff Near - Radius), 0, 1) velocityFactor = pow(1 - f, Radius Falloff Exponent)
      其中，clamp 函数将速度系数限制在 0 和 1 之间，pow 函数将速度系数指数化。通过这个公式，可以控制粒子在距离 FromPoint 不同的范围内，速度变化的快慢程度。
-- Offset：表示发射速度时在粒子的位置上添加一个偏移量，从偏移位置到原点位置的方向就是速度矢量的方向，以产生发射点的变化，从而使粒子在发射时具有随机性。默认情况下，offset 为 (0, 0, 0)，表示粒子在发射时的位置是粒子自身的位置，而没有任何偏移。例如，如果将Origin Offset 设置为 (50, 0, 0)，则从每个粒子的右侧发射速度。这可以用于模拟从侧面喷射出来的火焰、烟雾等效果。offset 参数可以是任意的向量，具体取决于所需的效果。
Origin Offset：起始点偏移 ，全部为 0 的时候粒子发散
### In Cone（圆锥发散）
![[Image/Niagara基础属性介绍/608e8ce5455347a7a3370f0e679cbfc0_MD5.png]]
## Vortex Velcity(涡量速度)
![[Image/Niagara基础属性介绍/d5f4f233e98cc713db2b211b7a3547cb_MD5.png]]
## Solve Forces and Velocity(解算力和速度)
![[Image/Niagara基础属性介绍/f941dab8d5f6d5489fe62f4a24bf41e0_MD5.png]]
解算更新粒子的实时速度和力，把计算的力结果融合进入物理系统，通过引擎解算计算粒子的最新速度，输出粒子的最新速度和位置
Speed Limit：速度限制
Acceleration Limit：加速度限制
Manually Enable Rotational Solver：手动开启转动结算
# Force Modules（施加力模块)
## Drag（阻力）
![[Image/Niagara基础属性介绍/8b2a710086b8eb7d18d74ac899f9e5f3_MD5.png]]
阻力效果：
Drag：线性减少每个粒子的速度，也可以切换其他模式，点击 下 箭头可选择
       Rotational Drag：减少粒子的旋转速度
## Curl Noise Force(旋转噪波力)
![[Image/Niagara基础属性介绍/d833cfcca2ab3748e5bddfaf0430f2e4_MD5.png]]
Write to Intrinsic Parmeters：写入内部参数，如果为true则旋转噪波力将附加到临时物理力
Noise Strength：噪波强度，缩放采样的旋转噪波力强度
Noise Frequency：噪波频率，调整位置以增加旋转噪波的采样率
Noise Quality/Cost：噪音波质量/成本
Pan Noise Field：泛噪声场的添加能让力的效果更加随机化，力场会让它带有动画扭动效果。值越大扭动频率越高
RandomizationVector：随机轴向向量，最终表现是随机轴向向量值*随机种子，其结果的改变能让波浪效果更加丰富
Mask Contribution：遮罩
CurlNoiseCooneMashAngle(0-180)：改变会形成一个遮罩
CurlNoiseConeMaskFalloffAnale：改变遮罩边缘的软硬过渡
CurlNoiseConeMaskAxis：改变粒子朝向
## Point Attraction Force(点吸引力)
![[Image/Niagara基础属性介绍/fe62008597f5c2b81f3daf1bd8a3e807_MD5.png]]
Point Attraction Force（点引力）是 Niagara 粒子系统中的一个模块，它可以模拟物体之间的引力关系，常用于吸引或排斥粒子，制造物体之间的交互效果。该模块会计算每个粒子到指定点的向量，并按照一定的公式计算引力，从而改变粒子的运动方向和速度。
Attraction Strength：吸引力强度 
Attraction Radius：吸引力半径
Falloff Exponent：衰减指数，默认参数0.5，用作平方反比
Attractor Position Offset：吸引力位置偏移
Kill Radius：杀伤半径，检查粒子是否在吸引力位置的半径范围内，不在则杀死粒子，可以防止反弹回高吸引速度
