# 人的运动模式
## 1.移动组件与玩家角色
角色的移动本质上就是合理的改变坐标位置，在UE里面角色移动的本质就是修改某个根组件的坐标位置。图2-1是我们常见的一个Character的组件构成情况，可以看到我们通常将CapsuleComponent(胶囊体)作为自己的根组件，而Character的坐标本质上就是其RootComponent的坐标，Mesh网格等其他组件都会跟随胶囊体而移动。移动组件在初始化的时候会把胶囊体设置为移动基础组件的UpdateComponent，随后的操作都是在计算UpdateComponent的位置。
![[Image/UE基于胶囊体移动的物理载具模拟/bcc217f116c6a0b9ec974488a09ebb39_MD5.png]]
## 2.运动组件继承树
![[Image/UE基于胶囊体移动的物理载具模拟/b36032a8e22b70b61b698668c9107dc2_MD5.png]]
UMovementComponent：移动组件的基类，实现了SafeMoveUpdatedComponent()的基本移动接口，可以调用UpdateComponent组件的接口函数来更新其位置
![[Image/UE基于胶囊体移动的物理载具模拟/4bde2c9756893d05b88ff6d882d848c7_MD5.png]]
UNavMovementComponent：该组件更多的是提供给AI寻路的能力，同时包括基本的移动状态，比如是否能游泳，是否能飞行等。
UPawnMovementComponent：组件开始变得可以和玩家交互了，前面都是基本的移动接口，不手动调用根本无法实现玩家操作，提供了AddInputVector()，可以实现接收玩家的输入并根据输入值修改所控制Pawn的位置。要注意的是，在UE中，Pawn是一个可控制的游戏角色（也可以是被AI控制），他的移动必须与特定的组件UPawnMovementComponent配合，玩家通过InputComponent组件绑定一个按键操作，然后在按键响应时调用Pawn的AddMovementInput接口，进而调用移动组件的AddInputVector()，调用结束后会通过ConsumeMovementInputVector()接口消耗掉该次操作的输入数值，完成一次移动操作
UCharacterMovementComponent：目前 人物 等使用的移动组件
在一个普通的三维空间里，最简单的移动就是直接修改角色的坐标。所以，我们的角色只要有一个包含坐标信息的组件，就可以通过基本的移动组件完成移动。但是随着游戏世界的复杂程度加深，我们在游戏里面添加了可行走的地面，可以探索的海洋。我们发现移动就变得复杂起来，玩家的脚下有地面才能行走，那就需要不停的检测地面碰撞信息（FFindFloorResult，FBasedMovementInfo）；玩家想进入水中游泳，那就需要检测到水的体积（GetPhysicsVolume()，Overlap事件，同样需要物理）；水中的速度与效果与陆地上差别很大，那就把两个状态分开写（PhysSwimming，PhysWalking）；移动的时候动画动作得匹配上啊，那就在更新位置的时候，更新动画（TickCharacterPose）；移动的时候碰到障碍物怎么办，被其他玩家推怎么处理（MoveAlongFloor里有相关处理）；游戏内容太少，想增加一些可以自己寻路的NPC，又需要设置导航网格（涉及到FNavAgentProperties）；一个玩家太无聊，那就让大家一起联机玩（模拟移动同步FRepMovement，客户端移动修正ClientUpdatePositionAfterServerUpdate）。
### 3.移动细节
![[Image/UE基于胶囊体移动的物理载具模拟/488dc2da2b0ce5b42123911ccc11a9c7_MD5.png]]
首先肯定是从Tick开始，每帧都要进行状态的检测与处理，状态通过一个移动模式MovementMode来区分，在合适的时候修改为正确的移动模式。移动模式默认有6种，基本常用的模式有行走、游泳、下落、飞行四种，有一种给AI代理提供的行走模式，最后还有一个自定义移动模式。
#### 3.1 Walking
行走模式可以说是所有移动模式的基础，也是各个移动模式里面最为复杂的一个。为了模拟出出真实世界的移动效果，玩家的脚下必须要有一个可以支撑不会掉落的物理对象，就好像地面一样。在移动组件里面，这个地面通过成员变量FFindFloorResult CurrentFloor来记录。在游戏一开始的时候，移动组件就会根据配置设置默认的MovementMode，如果是Walking，就会通过FindFloor操作来找到当前的地面
FindFloor的流程：FindFloor本质上就是通过胶囊体的Sweep检测来找到脚下的地面，所以地面必须要有物理数据，而且通道类型要设置与玩家的Pawn有Block响应。这里还有一些小的细节，比如我们在寻找地面的时候，只考虑脚下位置附近的，而忽略掉腰部附近的物体；Sweep用的是胶囊体而不是射线检测，方便处理斜面移动，计算可站立半径等，HitResult里面的Normal与ImpactNormal在胶囊体Sweep检测时不一定相同）。另外，目前Character的移动是基于胶囊体实现的，所以一个不带胶囊体组件的Actor是无法正常使用UCharacterMovementComponent的
![[Image/UE基于胶囊体移动的物理载具模拟/84d5bb6de0502f7c8082020625c4b339_MD5.png]]
找到地面玩家就可以站立住么？不一定。这又涉及到一个新的概念PerchRadiusThreshold，我称他为可栖息范围半径，也就是可站立半径。默认这个值为0，移动组件会忽略这个可站立半径的相关计算，一旦这个值大于0.15，就会做进一步的判断看看当前的地面空间是否足够让玩家站立在上面。
	前面的准备工作完成了，现在正式进入Walking的位移计算，这一段代码都是在PhysWalking里面计算的。为了表现的更为平滑流畅，UE4把一个Tick的移动分成了N段处理。在处理每段时，首先把当前的位置信息，地面信息记录下来。在TickComponent的时候根据玩家的按键时长，计算出当前的加速度。随后在CalcVelocity()根据加速度计算速度，同时还会考虑地面摩擦，是否在水中等情况。
算出速度之后，调用函数MoveAlongFloor()改变当前对象的坐标位置。在真正调用移动接口SafeMoveUpdatedComponent()前还会简单处理一种特殊的情况——玩家沿着斜面行走。正常在walking状态下，玩家只会前后左右移动，不会有Z方向的移动速度。如果遇到斜坡怎么办？如果这个斜坡可以行走，就会调用ComputeGroundMovementDelta()函数去根据当前的水平速度计算出一个新的平行与斜面的速度，这样可以简单模拟一个沿着斜面行走的效果，而且一般来说上坡的时候玩家的水平速度应该减小，通过设置bMaintainHorizontalGroundVelocity为false可以自动处理这种情况。我们在做载具的时候忽略了这种情况，默认不单独减小速度，通过摩擦力和坡度重力减速度来减小速度。
#### 3.2 Falling
#### 3.3 Swiming
#### 3.4 Flying
# 物理模拟状态划分
正常的话根据功能分为三个状态，PhysWalkingSim：模拟地面行走，PhysFallingSim：模拟载具坠落，PhysSliderSim：模拟坡道滑落，然后为了解决低速载具下大角度斜坡出现明显穿模问题添加了两个小状态，PhysMoveToFallingSim：正面从walk进入Falling的过渡状态,PhysMoveToFallingBackSim：倒车从walk进入Falling的过渡状态
## 1.模拟正常地面行走 PhysWalkingSim
![[Image/UE基于胶囊体移动的物理载具模拟/e2222c31eb76b400a22ff76fbfc7b531_MD5.png]]
载具的移动其实是基于胶囊体的移动，基础逻辑是存储载具自身的矢量速度（不包含Z轴的速度），然后在每一帧的 Tick 函数里计算位移，然后在根据当前坡面角度计算实际位移，然后再尝试移动，模拟函数的流程图如下：
![[Image/UE基于胶囊体移动的物理载具模拟/16455474d1525137052091a671a7b18a_MD5.png]]
```cpp
// 如果施加了加速度则添加加速度
const bool IsAccel = AccelDirection.SizeSquared() > SMALL_NUMBER;
const FVector OldAccelNormal = AccelDirection.GetSafeNormal();
if (IsAccel && SpeedParam.CurveAccelerationCoeff)
{
    float CurSpeedCoeff = SpeedParam.CurveAccelerationCoeff->GetFloatValue(LLVehicleData::GetVehicleSpeedPerHour(Velocity));
    const float SlopAccelCoeff = FMath::ClampAngle(CurSpeedCoeff, 0.f, 1.f);
    AccelDirection = AccelDirection.GetSafeNormal() * SpeedParam.MaxAcceleration * SlopAccelCoeff;
}

if (SpeedParam.CurveSlopDeAccelCoeff)
{
    const float Pitch = IsMovingOnGround() ? CalcVehiclePitch() : GetActorRotation().Pitch;
    const FVector SlopDeAccel = SpeedParam.GravAcceleration.Z * GetActorForwardVector().GetSafeNormal() * SpeedParam.CurveSlopDeAccelCoeff->GetFloatValue(FMath::Abs(Pitch));
    AccelDirection += (Pitch > 0.f ? 1.0f : -1.0f) * SlopDeAccel;
}

VehicleRunParam.BIsForwardAccel = LLVehicleData::IsNearVecOneWithVecTwo2D(OldAccelNormal, AccelDirection);

AccelDirection.X = FMath::IsFinite(AccelDirection.X) ? AccelDirection.X : int(AccelDirection.X);
AccelDirection.Y = FMath::IsFinite(AccelDirection.Y) ? AccelDirection.Y : int(AccelDirection.Y);
AccelDirection.Z = 0;

VehicleRunParam.CurAccelOfMaxAccel = AccelDirection.Length() / SpeedParam.MaxAcceleration;
```
```cpp
if (!GetOverlapCheckWalkable()) return false;

FVector OldLocationTemp = VehicleOwner->GetActorLocation();
FVector VelocityTemp = (Velocity + AccelDirection * deltaTime).Length() * (VehicleStatusWS == RunForward ? GetActorForwardVector() : GetActorBackVector());
FVector ForcastLocation = VehicleOwner->GetActorLocation() +  VelocityTemp * deltaTime;

FVector FCapLoc = RootCapssuleMeshFront->K2_GetComponentLocation();
FVector BCapLoc = RootCapssuleMeshBack->K2_GetComponentLocation();
FVector End = FCapLoc + GetActorForwardVector() * FVector::Distance(FCapLoc, BCapLoc);
FHitResult Hit(1.f);
const bool CantFly = SweepCheckCapsule(RootCapssuleMesh, ForcastLocation,ForcastLocation + FVector(0.f, 0.f, -FlyParam.StartFlyHeight), Hit);
if (!CantFly && !CommonLineTrace(FCapLoc, End, Hit))
{
	VehicleOwner->SetActorLocation(ForcastLocation);
	VehicleRunParam.StartFlyLocation = OldLocationTemp;
	return true;
}

return false;
```
```cpp
// 确保只有水平速度
	Velocity.Z = 0.f;
	AccelDirection.Z = 0.f;
	
	// 应用刹车或者减速
	const bool bZeroAcceleration = (VehicleAccel == EVehicleRunStatus::AccelNone);
	bool bVelocityOverMax = Velocity.SizeSquared() > FMath::Square(FMath::Max(0.f, CurrentMaxSpeed));
	bVelocityOverMax = GetOverlapCheckWalkable() ? bVelocityOverMax : (Velocity.SizeSquared() > FMath::Square(FMath::Max(0.f, SpeedParam.MaxSpeedHitWall)));
	
	// 计算减速 (bZeroAcceleration || bVelocityOverMax || GetIsDeAcceleration() || VehicleRunParam.BIsDrift) && 
	if (!Velocity.IsZero() || !VelocityExtra.IsZero())
	{
		ApplyVelocityBraking(timeTick);
	}

	// 应用加速度
	const bool bIsOverFriction = AccelDirection.Length() > GetVehicleCurFriction();
	if (!AccelDirection.IsZero() &&!bVelocityOverMax && VehicleStatusWS != EVehicleRunStatus::HandBrake &&
		 CurrentFloor.IsWalkableFloor() && bIsOverFriction)
	{
		Velocity += AccelDirection * timeTick;
		if (Velocity.Length() > CurrentMaxSpeed)
		{
			Velocity = Velocity.GetSafeNormal() * CurrentMaxSpeed;
		}
	}
	
	// 计算车身倾斜角
	if (AnimParam.CurveVehicleRotateDegree)
	{
		VehicleRunParam.VehicleBodyDegree = AnimParam.CurveVehicleRotateDegree->GetFloatValue(LLVehicleData::GetVehicleSpeedPerHour(Velocity));
	}
	
	// 计算载具运动方向
	CalcVehicleRunDirection();
	
	// 计算载具当前可行驶最大速度
	if (VehicleStatusWS == EVehicleRunStatus::RunBack || (VehicleStatusWS == EVehicleRunStatus::Stop && !VehicleRunParam.BIsForwardAccel))
	{
		CurrentMaxSpeed = SpeedParam.MaxSpeedBack;
	}
	else
	{
		float SpeedCoeff = 1.0f;
		if (SpeedParam.CurveSlopeSpeedCoeff)
		{
			SpeedCoeff = FMath::Clamp(SpeedParam.CurveSlopeSpeedCoeff->GetFloatValue(VehicleOwner->GetActorRotation().Pitch), 0.f, 1.0f);
		}
		
		CurrentMaxSpeed = SpeedParam.MaxSpeed * SpeedCoeff;
	}
```
```cpp
if (!CurrentFloor.IsWalkableFloor())
{
	PrintDebugInfo(205);
	return;
}
FVector OldLocation = RootCapssuleMesh->GetComponentLocation();
FVector Delta = FVector(InVelocity.X, InVelocity.Y, 0.f) * DeltaSeconds;
if (!VelocityExtra.IsZero())
{
	Delta += FVector(VelocityExtra.X, VelocityExtra.Y, 0.f) * DeltaSeconds;
}
FHitResult Hit(1.f);
FVector RampVector = ComputeGroundMovementDelta(Delta, CurrentFloor.HitResult, CurrentFloor.bLineTrace);
SafeMoveUpdatedComponent(RampVector, RootCapssuleMesh->GetComponentQuat(), true, Hit);
float LastMoveTimeSlice = DeltaSeconds;
if (Hit.bStartPenetrating)
{
	// Allow this hit to be used as an impact we can deflect off, otherwise we do nothing the rest of the update and appear to hitch.
	SlideAlongSurface(Delta, 1.f, Hit.Normal, Hit, true);
	
	if (Hit.bStartPenetrating)
	{
		bJustTeleported = true;
	}
}
else if (Hit.IsValidBlockingHit())
{
	// 载具前面撞击到墙体
	if (!GetOverlapCheckWalkable())
	{
		if (VehicleStatusWS == EVehicleRunStatus::RunForward)
		{
			DealVehicleHitWall(DeltaSeconds);
		}
		else
		{
			CalcVehicleHitAngle(Hit);
			ClearVehicleVelocity();
		}
	}
	else
	{
		// We impacted something (most likely another ramp, but possibly a barrier).
		float PercentTimeApplied = Hit.Time;
		if ((Hit.Time > 0.f) && (Hit.Normal.Z > KINDA_SMALL_NUMBER) && IsWalkable(Hit))
		{
			// Another walkable ramp.
			const float InitialPercentRemaining = 1.f - PercentTimeApplied;
			RampVector = ComputeGroundMovementDelta(Delta * InitialPercentRemaining, Hit, false);
			LastMoveTimeSlice = InitialPercentRemaining * LastMoveTimeSlice;
			SafeMoveUpdatedComponent(RampVector, RootCapssuleMesh->GetComponentQuat(), true, Hit);
		
			const float SecondHitPercent = Hit.Time * InitialPercentRemaining;
			PercentTimeApplied = FMath::Clamp(PercentTimeApplied + SecondHitPercent, 0.f, 1.f);
		}
		
		if (Hit.IsValidBlockingHit())
		{
			if (CanStepUp(Hit) ||  Hit.HitObjectHandle == VehicleOwner)
			{
				// hit a barrier, try to step up
				const FVector PreStepUpLocation = RootCapssuleMesh->GetComponentLocation();
				const FVector GravDir(0.f, 0.f, -1.f);
				if (!StepUp(GravDir, Delta * (1.f - PercentTimeApplied), Hit, OutStepDownResult))
				{					
					ClearVehicleVelocity();
				}
			}
			else if ( Hit.Component.IsValid() && !Hit.Component.Get()->CanCharacterStepUp(VehicleOwner) )
			{
				SlideAlongSurface(Delta, 1.f - PercentTimeApplied, Hit.Normal, Hit, true);
			}
		}
	}
}

// 射线检测载具和墙面
LineTraceOnWall();
```
![[Image/UE基于胶囊体移动的物理载具模拟/9000135b2de5270294fe9a5bf7ae2a11_MD5.png]]
MoveUpdatedComponent：由于载具的主胶囊体在载具的前方，且只能包含住一部分载具，所以载具使用了三个胶囊体，主胶囊体和前胶囊体保持一致的位置和大小，主胶囊体带动运动和偏航角（Yaw）的转向，前胶囊体控制俯仰角（Pitch）和 翻滚角（roll），但是目前策划要求摩托不调整翻滚角的值，然后后边的胶囊体包含住车的尾巴在载具倒车的时候进行简单的检测，如果是倒车且倒车没有检测到不可行走墙面的时候，就尝试移动尾部胶囊体，然后根据实际移动的情况（OutHit->Time）来决定主胶囊体的移动距离 DeltaLocation，如果是前进的话就直接尝试位移主胶囊体，这个检测是为了避免倒车的时候车尾部和可碰撞物体产生的穿模问题，因为在实际运动过程中能判断碰撞的就只有根胶囊体.
```cpp
if (VehicleStatusWS == EVehicleRunStatus::RunBack && IsMovingOnGround() && !GetOverlapCheckWalkable())
{
	FVector BackLocation = RootCapssuleMeshBack->GetComponentLocation();
	RootCapssuleMeshBack->MoveComponent(Delta, NewRotation, bSweep, OutHit, MoveComponentFlags, Teleport);
	RootCapssuleMeshBack->SetRelativeTransform(TransBackRelToFront);
	if (OutHit && OutHit->Time < 1.f)
	{
		return RootCapssuleMesh->MoveComponent(OutHit->Location - BackLocation, NewRotation, false);
	}
	else
	{
		return RootCapssuleMesh->MoveComponent(Delta, NewRotation, false);
	}
}
const bool IsSuccess = RootCapssuleMesh->MoveComponent(Delta, NewRotation, bSweep, OutHit, MoveComponentFlags, Teleport);
return IsSuccess;
```
移动方面遇到问题最多的就是各种碰撞问题，因为是纯模拟的碰撞逻辑处理所以需要对遇到的各种极限的地形进行对应的处理
### 1.1 环形倒立赛道
![[Image/UE基于胶囊体移动的物理载具模拟/05ddb6ff875ca5870ef6e5a278becfc8_MD5.png]]
上图表现得就是一个环形倒立赛道，图示显示的是我们能够到达的最高处，在这一帧实际检测运动 FHitresult 的结果是 block，这个时候如果一直按着前进按键人的处理方式是会尝试再一次移动，如果还是 block 就会调用 SlideAlongSurface 接口沿着平面横向滑行，然后又由于人的移动速度相对较慢且人的身体是纵向的模型，横移的幅度也比较小看着也比较合理，但是应用到载具身上就非常不合理，首先就是载具是一个横向的模型直接偏移很奇怪，其次就是载具冲上来的速度很快，如果不对速度进行处理那么载具会以一个很快的速度横移，所以载具的处理方式是直接把速度清空，如果长按前进按键的话就会一直触发速度清空然后保持不动，松手的话就自动滑落。

### 1.2 直角坡临界面侧面滑落
![[Image/UE基于胶囊体移动的物理载具模拟/e983de62ee50ab8f893c4a2939874c20_MD5.png]]
每次移动之后检测当前所处地面的行走状态，如果地面不可行走的话进入其他状态，但是上图所示的临界状态比较特殊，之前想的是这种地面不可行走那就直接进入 PhysFallingSim 状态，但是由于前胶囊体并没有完全脱离地面，向下打射线检测是否着陆的话必定是能打到碰撞物体，然后就会出现一种状态是 载具卡在那里 然后在 PhysWalkingSim  和 PhysFallingSim 之间不停的切换，然后就根据情况区分加入了一个新的状态   滑落状态（PhysSliderSim）,这个状态对应的人的状态其实是 PhysFalling，也就是人的坠落状态，但其实人的滑落走的也是坠落 Falling，当载具开到这里如果不能继续行走的话就会进入 飞出 或者 滑落状态，然后目前测试好像没出现之前的卡在那里不动得现象。
### 1.3 低速正面下直角坡（PhysMoveToFallingSim）
![[Image/UE基于胶囊体移动的物理载具模拟/aba8a98a201d5572ed316992838f71c4_MD5.png]]
因为目前来说我们的根组件其实就是上图红框里的胶囊体，当载具在当前的位置继续向前前进的时候会根据速度出现种情况，第一种就是载具当时跑道这种地形的时候载具的速度比较高，然后呢这一段距离很快就位移过去了，基本没什么问题，另一个情况就是速度非常的低以一个很低的速度下直角坡，这个时候可以预见的问题是根胶囊体已经没有着陆点了进入了 PhysFallingSim 模拟下落状态，但是呢摩托车的胶囊体模型还在地面上，如果不进行处理的话就会出现下面的情况
![[Image/UE基于胶囊体移动的物理载具模拟/58d81329cc05b972a110d56f0952425f_MD5.png]]
模型穿模就非常严重，然后这种情况目前的处理方式是 加入一个过渡状态 PhysMoveToFallingSim, 意思就是从 地面到空中的过渡状态，这个状态的基本逻辑是：判断是否能检测到地面，检测到地面的话就退出此状态进入PhysWalkingSim，检测不到的话就开始以前胶囊体向后打半个胶囊体高度的射线
![[Image/UE基于胶囊体移动的物理载具模拟/6b8d59694dc214e588df29d9757661bb_MD5.png]]
![[Image/UE基于胶囊体移动的物理载具模拟/a2b8bb22fcbff8356ab51c4c2d00fa88_MD5.png]]
分别检测能否打中地面 和 是否是载具的前半部分打的射线击中了地面，如果是前半部分击中了地面的话相当于模拟载具本身的重量卡在悬崖边的效果，避免车头出去一点就向下坠落显得比较假，还有一点就是这个状态的时候玩家还是可以按下后退按键后退的，然后如果车前半部分出了这个断崖面的话为了避免载具速度较小给了一个比较小的额外速度 MoveToFallExtraSpeed,同时使用特殊的重力参数避免载具下落速度过快，相当于尽量增加载具的滞空时间，然后在载具进入飞出的一瞬间记录根胶囊体的位置，之后在载具完全进入滞空状态之前的时候每帧获取根胶囊体的位置，通过这两个位置实时更新载具的俯仰角Pitch，进一步避免穿模的现象，但是不完善的地方是遇到倾斜角比较大的地方还是会有一些穿模。
### 1.4 低速背面下直角坡（PhysMoveToFallingBackSim）
![[Image/UE基于胶囊体移动的物理载具模拟/1d01ab71461c5377086f7cc900d5b2e4_MD5.png]]
背面下这种坡度的地形的时候其实比正面下感觉处理的时候更加的奇怪就是逻辑上不复杂但是呢就是不符合载具之前的运动设定（前胶囊体带动俯仰角改变），正常来讲如果不处理的话他的运动结果应该是像下图一样：
![[Image/UE基于胶囊体移动的物理载具模拟/337f7ef18716d2f162c7ccd8ce6154ac_MD5.png]]
![[Image/UE基于胶囊体移动的物理载具模拟/17c40996ae51ecb5e31510e1cff3b466_MD5.png]]
就当根胶囊体离开地面之前的时候载具完全就悬空了等到根胶囊体开始下坠载具整体也跟着下坠，比较诡异，然后这个时候如果按照载具以前的逻辑调整根胶囊体的俯仰角就会出现载具车身中间部分卡在模型里面，所以就不能按照之前的俯仰角调整逻辑来处理这个事情，如下图分别单独调整前后胶囊体的俯仰角
![[Image/UE基于胶囊体移动的物理载具模拟/0cba016c4d2396aedca538cb307cb7df_MD5.png]]
第一张图是只调整了前胶囊体的俯仰角的效果，可以看到有一部分模型穿模了，第二张图只调整了后胶囊体的俯仰角可以看到的是载具模型悬空了，第三张图是同时调整了前后胶囊体相对于父节点的相对偏移，效果互相补充看着就还好，也有一部分原因是摆的位置和修改的角度参数比较契合的原因，在实际的运行效果上，由于也一个后退的额外速度参数来控制，所以看起来的效果基本上就是有一个模型后仰的趋势动作，但不会后仰太多车子就进入坠落状态然后恢复正常的角度。
### 1.5 载具俯仰角计算逻辑
#### 1.51穿模问题
目前摩托车的俯仰角计算是通过前后胶囊体向下打 capsule 射线确定两点然后通过这两点来确定俯仰角，但是有的时候会有一些特殊的情况，就比如之前遇到的一种情况：![[Image/UE基于胶囊体移动的物理载具模拟/7093108ba53faa6c92af1025d2d5fb3f_MD5.png]]
![[Image/UE基于胶囊体移动的物理载具模拟/d5c5f8d2e7f4d69ec39538d6285ffc7d_MD5.png]]
就是在上图状态的时候，之前还没做倒车下坡的状态，所以如果以一种极低的速度通过的时候就可能产生这种情况，图一后轮胶囊体其实在悬空的时候检测不到地面然后就不能通过前后点来确定俯仰角，就只能用前轮胶囊体的射线法线来检测，但是呢它这个法线也不能直接使用，图二和图三就是直接使用法线导致的问题，原因是它的碰撞可能在图四所示的侧面，然后当时排除了这种情况后，有补充了一种额外保底的计算方式，就是图五这样，从载具中心打几个射线取 法线 的和的法线用来计算（图示 我调整了下打射线的位置，因为现在基本复现不出来这种情况，为了方便截图把射线起点放到了后边），然后如果算的值还有问题的话就保持当前角度不改变。
下图是现在的实际情况，在坡度比较低的时候会通过调整车身角度 和 载具本身的尾部减震结构让轮胎尽量往下压到策划配置的最大值，然后如果高度再高点的话就会进入 PhysMoveToFallingBackSim 状态，走它的逻辑。
![[Image/UE基于胶囊体移动的物理载具模拟/fc8f923d83151a9429282678cc8a2b0d_MD5.png]]
#### 1.52 数值应用问题
还有一个问题就是实时计算出来的载具俯仰角的过渡问题，如果直接使用计算结果非常明显的问题就是载具的俯仰角变换过于的剧烈，表现出来的效果就是非常的抖，但是如果不使用实时计算出来的数据的话在地形变化比较剧烈的地面上行驶的时候载具就会穿模，这个问题感觉没什么能完美解决的方法，现在采用的方式是 区分单次俯仰角变化 差值，就是比如：单次角度变化 0-5 用的一条曲线的数据，5-10 用一条曲线的数据，10以上再用一条曲线的数据，然后曲线 的 X 轴是载具速度，Y轴是插值系数，这样的话在载具速度比较小的时候策划配置一个相对较大的 插值 系数看起来不会穿模，然后因为速度相对较慢跨越的地形距离也比较短抖动也相对还好，然后在速度比较快的时候策划配置一个相对比较小的系数，这样跑起来的时候看起来抖动比较小，然后也因为速度相对较快，运动较快也能很快离开会穿模的地形。
然后就是 前进 和 后退的逻辑其实也是有区别的，前进的时候用的就上边的逻辑插值计算，这是因为胶囊体在载具最前方，前进的时候能永远保持前轮模型在地面上，但是后退的时候不一样，后退的时候只能采用实时的计算结果，不然倒退遇到骤变的坡度的时候尾部如果采用插值计算的话会明显穿模，而且倒车速度比较低，所以采用实时计算的角度也没什么问题。 
### 1.6 贴墙转向离开
![[Image/UE基于胶囊体移动的物理载具模拟/ea3c85e7c835482a6309f3ab49087a02_MD5.png]]
现在这个贴墙处理的方案还是不是很好，之后估计策划还得重新定方案或者向其他方法来完善逻辑。
现在载具的贴墙处理逻辑是当载具从图一所示的状态一直按着前进按键载具会自动贴墙偏转，而且最大也只能是水平于墙面，类似图二的效果，那么这个时候再按着前进按键载具会贴墙前进，但是玩家的操作可能是按着右转向按键想要转出，但是按照我们正常的运动逻辑是没办法做到的(和正常的物理载具不一样，物理载具运动的方向是载具车头的方向，但是我们的运动完全是被胶囊体带着运动的，没办法让胶囊体跟随车头动画动态转向)所以就只能用其他的方式，现在的处理方式是补充了额外逻辑如果贴墙的角度和速度同时小于策划的配置值且玩家按着远离墙面的方向按键的时候,会给载具一个正右方向的额外的速度，这个速度比较小，帮助载具胶囊体稍微离开墙面的同时也能减少转向的时候尾部穿模的问题，并且给了一个额外的速度转向角把速度的方向向离开墙面的方向进行额外转向，然后再计算单次位移，基本能模拟出载具的转向位移效果。
## 2.模拟正面从 PhysWalkingSim到坠落PhysFallingSim 即 PhysMoveToFallingSim
详见 1.3
## 3.模拟背面从 PhysWalkingSim到坠落PhysFallingSim 即 PhysMoveToFallingBackSim
详见 1.4
## 4.模拟坠落 PhysFallingSim
![[Image/UE基于胶囊体移动的物理载具模拟/49ded6184c6fc03f8d11d998b5beb288_MD5.png]]
坠落的时候和行走的时候不一样不需要计算加速度，在空中的时候速度 只有Z轴速度会跟随滞空时间变化，玩家在空中的操作只能向左或者向右摇摆车头，在CalcVehicleFlyData里计算，在空中的时候只修改载具的朝向不修改载具的速度方向，保持载具原先的运行轨迹，在载具落地的时候开始速度向载具朝向回归模拟落地滑车的效果，然后在载具完全滞空之前还需要一直检测载具的车身碰撞调整载具的俯仰角，等到载具完全滞空后，开始自动调整载具的俯仰角（Pitch）和翻滚角（Roll）到策划配置的数据。
处理落地的时候遇到一个问题，就是载具尾部先落地，但是此时根胶囊体还是没有落地的，这样就会导致尾部穿模，现在处理方式的每帧检测前后轮是否落地，如果后轮先落地的话就在落地的瞬间修改载具的俯仰角不让其穿模。
## 5.模拟滑落 PhysSliderSim
这个状态的实现逻辑基本和人物坠落 PhysFalling 一致，暂时只是用来处理一些特殊状态，比如下图滑落的时候。这个没啥特别的。有兴趣的可以直接看人的坠落逻辑。
![[Image/UE基于胶囊体移动的物理载具模拟/bde8eab241b43ff51895006c94db8415_MD5.png]]
# TS部分逻辑
## 1.寻路
![[Image/UE基于胶囊体移动的物理载具模拟/08def7ce1b977e6b5eb3868ce16efaa1_MD5.png]]
寻路策划要求可以自动避障走到上下车点，但是对于动态物体来说 Nav寻路没办法判断，所以如果在路上有动态物体挡在路上寻路就会被卡着，现在的方法是在载具周围摆几个上车点，然后统计所有通过周围的过渡点能够到达上车点的路线，再选取最短的路线过去。
## 2.特效挂点
![[Image/UE基于胶囊体移动的物理载具模拟/6f546859d8753f2e561b419024062b32_MD5.png]]
有的特效需要挂载在轮胎底部和地面接触的地方，但是因为没办法保证载具模型的后轮胎一直在地面上，如果直接把特效组件挂载在模型上就会出现特效组件在地面或者悬在空中穿帮的情况，所以添加了几个 SceneComponent 从轮胎骨骼向下打射线，取 HitLocation 向上偏移一小段距离的位置作为轮胎和地面的接触位置，然后把特效组件挂载在在这个组件上就可以保证特效不会断开（比如车胎痕迹）
## 3.特效切换
在进入不同的材质地面上切换不同的特效效果，直接切换特效的话会导致之前的特效直接消失，比较突兀，可以新建一个特效组件挂载，只要之前的组件被 deactive 等到特效消失之后组件就会自己被回收，之前不知道这个特性，费了很大劲感觉效果还不是很好，换这种方式的话会方便的多。
# 类逻辑
## 基础继承关系
![[Image/UE基于胶囊体移动的物理载具模拟/b2ab639cdc70302d8efd3dd34479da17_MD5.jpg]]
## 程序逻辑
玩家：
添加一个载具交互组件  TS_CallVehicleComponent 继承自 DriveSystemComponent 拥有属性如下图，目前上车请求只能 上自己拥有的载具（暂时没有具体上车需求）根据 OwnVehicleID 获取载具，然后本地检测可以到达的上车点位置并往其位移，然后通过人物的移动同步实现玩家同步到上车点，到达上车点之后发送请求载具上车协议  CSRideVehicle(vehicleId: BigInt, loc: number, bIsGetIn: boolean) 然后服务器广播玩家状态变更协议 客户端会根据收到的协议带来的  上车点位置信息 决定播放哪个动作，这个协议还需要根据需求完善，需要加上 请求的玩家ID（现在服务器默认只能上自己召唤的车所以没加请求的玩家ID），然后最终上哪个载具需要由服务器决定，现在是本地检测距离最近的座位，然后控制权由座位决定，现在是谁拥有谁可以控制(需要具体需求)
上下车动画：保存在 TS_CallVehicleComponent 组件中，根据 玩家sn_载具sn 读取配置设置
直接挂载玩家到载具上(无动画过程)：AttachToVehicle  适用于上线的时候 玩家已经在 载具上了 + 载具进入视野重新创建的时候
AttachToVehicle：本地查找空闲座位 + 判断是否可以由本上车点上去(走配置) + 挂载玩家到载具上
交互上车(有动画过程)：玩家跟载具互动上下车
OnDealVehicleInteract：根据上下车决定用哪个动画(这个需要梳理下映射关系) + 播放蒙太奇 + 蒙太奇播放结束处理镜头和控制权（控制权暂时只给了载具的召唤者）（需要去掉 本地保存的骑乘玩家引用，根据座位控制权从座位来那objid获取actor）
![[Image/UE基于胶囊体移动的物理载具模拟/23b9a5bc1e5400b91739c28a3b6126b3_MD5.png]]

通用参数结构：
![[Image/UE基于胶囊体移动的物理载具模拟/c1ab1893a4e90ada9f002a2c63a2742c_MD5.png]]
