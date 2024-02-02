<a name="UEKIy"></a>
# 人的运动模式
<a name="usaSs"></a>
## 1.移动组件与玩家角色
角色的移动本质上就是合理的改变坐标位置，在UE里面角色移动的本质就是修改某个根组件的坐标位置。图2-1是我们常见的一个Character的组件构成情况，可以看到我们通常将CapsuleComponent(胶囊体)作为自己的根组件，而Character的坐标本质上就是其RootComponent的坐标，Mesh网格等其他组件都会跟随胶囊体而移动。移动组件在初始化的时候会把胶囊体设置为移动基础组件的UpdateComponent，随后的操作都是在计算UpdateComponent的位置。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668569350820-2df4133b-726d-414e-9d8a-a24027d690b7.png#averageHue=%232b2a2a&clientId=u19a77d12-fb1d-4&from=paste&height=221&id=uf839ce5a&originHeight=221&originWidth=445&originalType=binary&ratio=1&rotation=0&showTitle=false&size=23435&status=done&style=none&taskId=u2c5b4d6d-64a7-46de-a03d-075ad015c7a&title=&width=445)
<a name="K58YQ"></a>
## 2.运动组件继承树
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668569531580-8a95299b-0ff5-4577-82d5-3695fa42b960.png#averageHue=%23efefef&clientId=u19a77d12-fb1d-4&from=paste&height=925&id=u4cfe61ca&originHeight=925&originWidth=885&originalType=binary&ratio=1&rotation=0&showTitle=false&size=455030&status=done&style=none&taskId=ue3a0fffc-16eb-47b2-aa09-626c7a57fd9&title=&width=885)<br />UMovementComponent：移动组件的基类，实现了SafeMoveUpdatedComponent()的基本移动接口，可以调用UpdateComponent组件的接口函数来更新其位置<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668569994793-5861dc68-fbf3-4767-afc6-6f6c7e62961f.png#averageHue=%232d2c2c&clientId=u19a77d12-fb1d-4&from=paste&height=233&id=x18Fr&originHeight=233&originWidth=1276&originalType=binary&ratio=1&rotation=0&showTitle=false&size=31595&status=done&style=none&taskId=u40d410d7-0067-4e8e-ab36-352b70c0321&title=&width=1276)<br />UNavMovementComponent：该组件更多的是提供给AI寻路的能力，同时包括基本的移动状态，比如是否能游泳，是否能飞行等。<br />UPawnMovementComponent：组件开始变得可以和玩家交互了，前面都是基本的移动接口，不手动调用根本无法实现玩家操作，提供了AddInputVector()，可以实现接收玩家的输入并根据输入值修改所控制Pawn的位置。要注意的是，在UE中，Pawn是一个可控制的游戏角色（也可以是被AI控制），他的移动必须与特定的组件UPawnMovementComponent配合，玩家通过InputComponent组件绑定一个按键操作，然后在按键响应时调用Pawn的AddMovementInput接口，进而调用移动组件的AddInputVector()，调用结束后会通过ConsumeMovementInputVector()接口消耗掉该次操作的输入数值，完成一次移动操作<br />UCharacterMovementComponent：目前 人物 等使用的移动组件<br />在一个普通的三维空间里，最简单的移动就是直接修改角色的坐标。所以，我们的角色只要有一个包含坐标信息的组件，就可以通过基本的移动组件完成移动。但是随着游戏世界的复杂程度加深，我们在游戏里面添加了可行走的地面，可以探索的海洋。我们发现移动就变得复杂起来，玩家的脚下有地面才能行走，那就需要不停的检测地面碰撞信息（FFindFloorResult，FBasedMovementInfo）；玩家想进入水中游泳，那就需要检测到水的体积（GetPhysicsVolume()，Overlap事件，同样需要物理）；水中的速度与效果与陆地上差别很大，那就把两个状态分开写（PhysSwimming，PhysWalking）；移动的时候动画动作得匹配上啊，那就在更新位置的时候，更新动画（TickCharacterPose）；移动的时候碰到障碍物怎么办，被其他玩家推怎么处理（MoveAlongFloor里有相关处理）；游戏内容太少，想增加一些可以自己寻路的NPC，又需要设置导航网格（涉及到FNavAgentProperties）；一个玩家太无聊，那就让大家一起联机玩（模拟移动同步FRepMovement，客户端移动修正ClientUpdatePositionAfterServerUpdate）。
<a name="k4umy"></a>
### 3.移动细节
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668579511560-0eb9620a-2676-4957-ac2f-38c89c2c685e.png#averageHue=%23f8f8f8&clientId=u19a77d12-fb1d-4&from=paste&height=451&id=uc63608f7&originHeight=451&originWidth=899&originalType=binary&ratio=1&rotation=0&showTitle=false&size=110300&status=done&style=none&taskId=u90565301-07e8-4f06-b8ee-d1cf2b5a70e&title=&width=899)<br />首先肯定是从Tick开始，每帧都要进行状态的检测与处理，状态通过一个移动模式MovementMode来区分，在合适的时候修改为正确的移动模式。移动模式默认有6种，基本常用的模式有行走、游泳、下落、飞行四种，有一种给AI代理提供的行走模式，最后还有一个自定义移动模式。
<a name="QgNKn"></a>
#### 3.1 Walking
行走模式可以说是所有移动模式的基础，也是各个移动模式里面最为复杂的一个。为了模拟出出真实世界的移动效果，玩家的脚下必须要有一个可以支撑不会掉落的物理对象，就好像地面一样。在移动组件里面，这个地面通过成员变量FFindFloorResult CurrentFloor来记录。在游戏一开始的时候，移动组件就会根据配置设置默认的MovementMode，如果是Walking，就会通过FindFloor操作来找到当前的地面<br />FindFloor的流程：FindFloor本质上就是通过胶囊体的Sweep检测来找到脚下的地面，所以地面必须要有物理数据，而且通道类型要设置与玩家的Pawn有Block响应。这里还有一些小的细节，比如我们在寻找地面的时候，只考虑脚下位置附近的，而忽略掉腰部附近的物体；Sweep用的是胶囊体而不是射线检测，方便处理斜面移动，计算可站立半径等，HitResult里面的Normal与ImpactNormal在胶囊体Sweep检测时不一定相同）。另外，目前Character的移动是基于胶囊体实现的，所以一个不带胶囊体组件的Actor是无法正常使用UCharacterMovementComponent的<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668580809524-d32f12f4-b26f-4687-81d7-f2fa796a9d64.png#averageHue=%23f9f7f3&clientId=u19a77d12-fb1d-4&from=paste&height=693&id=ub40c1fc0&originHeight=693&originWidth=926&originalType=binary&ratio=1&rotation=0&showTitle=false&size=174372&status=done&style=none&taskId=ua14c3868-0acc-4707-b2e6-b35e46e5d00&title=&width=926)<br />找到地面玩家就可以站立住么？不一定。这又涉及到一个新的概念PerchRadiusThreshold，我称他为可栖息范围半径，也就是可站立半径。默认这个值为0，移动组件会忽略这个可站立半径的相关计算，一旦这个值大于0.15，就会做进一步的判断看看当前的地面空间是否足够让玩家站立在上面。<br />	前面的准备工作完成了，现在正式进入Walking的位移计算，这一段代码都是在PhysWalking里面计算的。为了表现的更为平滑流畅，UE4把一个Tick的移动分成了N段处理。在处理每段时，首先把当前的位置信息，地面信息记录下来。在TickComponent的时候根据玩家的按键时长，计算出当前的加速度。随后在CalcVelocity()根据加速度计算速度，同时还会考虑地面摩擦，是否在水中等情况。<br />算出速度之后，调用函数MoveAlongFloor()改变当前对象的坐标位置。在真正调用移动接口SafeMoveUpdatedComponent()前还会简单处理一种特殊的情况——玩家沿着斜面行走。正常在walking状态下，玩家只会前后左右移动，不会有Z方向的移动速度。如果遇到斜坡怎么办？如果这个斜坡可以行走，就会调用ComputeGroundMovementDelta()函数去根据当前的水平速度计算出一个新的平行与斜面的速度，这样可以简单模拟一个沿着斜面行走的效果，而且一般来说上坡的时候玩家的水平速度应该减小，通过设置bMaintainHorizontalGroundVelocity为false可以自动处理这种情况。我们在做载具的时候忽略了这种情况，默认不单独减小速度，通过摩擦力和坡度重力减速度来减小速度。
<a name="gC6Fh"></a>
#### 3.2 Falling
<a name="TXkaX"></a>
#### 3.3 Swiming
<a name="s0ofX"></a>
#### 3.4 Flying
<a name="V68s8"></a>
# 物理模拟状态划分
正常的话根据功能分为三个状态，PhysWalkingSim：模拟地面行走，PhysFallingSim：模拟载具坠落，PhysSliderSim：模拟坡道滑落，然后为了解决低速载具下大角度斜坡出现明显穿模问题添加了两个小状态，PhysMoveToFallingSim：正面从walk进入Falling的过渡状态,PhysMoveToFallingBackSim：倒车从walk进入Falling的过渡状态
<a name="Nd7RF"></a>
## 1.模拟正常地面行走 PhysWalkingSim
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1667823914184-c8441505-cd18-4158-a5e3-6671e6ff8eca.png#averageHue=%23222121&clientId=uda68e2a0-1a2d-4&from=paste&height=591&id=u4dcf283f&originHeight=591&originWidth=1363&originalType=binary&ratio=1&rotation=0&showTitle=false&size=359550&status=done&style=none&taskId=ub18767cc-9a3e-4a13-837b-4dac9374dfa&title=&width=1363)<br />载具的移动其实是基于胶囊体的移动，基础逻辑是存储载具自身的矢量速度（不包含Z轴的速度），然后在每一帧的 Tick 函数里计算位移，然后在根据当前坡面角度计算实际位移，然后再尝试移动，模拟函数的流程图如下：<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668759379685-47672b00-bd30-43da-bb56-2ff7155a91a9.png#averageHue=%23fefdfc&clientId=uaf2cb688-a680-4&from=paste&height=594&id=ua625a29d&originHeight=594&originWidth=371&originalType=binary&ratio=1&rotation=0&showTitle=false&size=22167&status=done&style=none&taskId=u30d8d0e3-8abc-4315-a2ac-8dc93efe5ac&title=&width=371)
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
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669030723545-e02b81f0-58a7-4824-9c3e-84c10ef7b8c9.png#averageHue=%23362a1a&clientId=ubc73ec77-f8b8-4&from=paste&height=353&id=TeGfk&originHeight=353&originWidth=892&originalType=binary&ratio=1&rotation=0&showTitle=false&size=240312&status=done&style=none&taskId=u12093129-03e3-4a33-aeb1-b491b2296d7&title=&width=892)<br />MoveUpdatedComponent：由于载具的主胶囊体在载具的前方，且只能包含住一部分载具，所以载具使用了三个胶囊体，主胶囊体和前胶囊体保持一致的位置和大小，主胶囊体带动运动和偏航角（Yaw）的转向，前胶囊体控制俯仰角（Pitch）和 翻滚角（roll），但是目前策划要求摩托不调整翻滚角的值，然后后边的胶囊体包含住车的尾巴在载具倒车的时候进行简单的检测，如果是倒车且倒车没有检测到不可行走墙面的时候，就尝试移动尾部胶囊体，然后根据实际移动的情况（OutHit->Time）来决定主胶囊体的移动距离 DeltaLocation，如果是前进的话就直接尝试位移主胶囊体，这个检测是为了避免倒车的时候车尾部和可碰撞物体产生的穿模问题，因为在实际运动过程中能判断碰撞的就只有根胶囊体.
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
<a name="SqJMA"></a>
### 1.1 环形倒立赛道
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669012240346-4d5af0f2-f9fd-43c8-a467-f03c376992fe.png#averageHue=%23d0b79f&clientId=ubc73ec77-f8b8-4&from=paste&height=652&id=uccbfd155&originHeight=652&originWidth=1857&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1893006&status=done&style=none&taskId=u0383927d-508c-4ed4-bc7a-9cc163c888b&title=&width=1857)<br />上图表现得就是一个环形倒立赛道，图示显示的是我们能够到达的最高处，在这一帧实际检测运动 FHitresult 的结果是 block，这个时候如果一直按着前进按键人的处理方式是会尝试再一次移动，如果还是 block 就会调用 SlideAlongSurface 接口沿着平面横向滑行，然后又由于人的移动速度相对较慢且人的身体是纵向的模型，横移的幅度也比较小看着也比较合理，但是应用到载具身上就非常不合理，首先就是载具是一个横向的模型直接偏移很奇怪，其次就是载具冲上来的速度很快，如果不对速度进行处理那么载具会以一个很快的速度横移，所以载具的处理方式是直接把速度清空，如果长按前进按键的话就会一直触发速度清空然后保持不动，松手的话就自动滑落。

<a name="UmCxH"></a>
### 1.2 直角坡临界面侧面滑落
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668844492792-a75be6cd-6fec-4f32-9176-bf0742e9e203.png#averageHue=%23cac1af&clientId=uaf2cb688-a680-4&from=paste&height=286&id=u6d0873ed&originHeight=310&originWidth=604&originalType=binary&ratio=1&rotation=0&showTitle=false&size=265215&status=done&style=none&taskId=u18c604c6-13fe-4d30-b915-ac5b7e90fa5&title=&width=557)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668844502551-eed54b1b-d140-4346-9288-ea0a94a366d1.png#averageHue=%23beae97&clientId=uaf2cb688-a680-4&from=paste&height=299&id=ua6f921bb&originHeight=318&originWidth=555&originalType=binary&ratio=1&rotation=0&showTitle=false&size=297127&status=done&style=none&taskId=udfd32ba3-28f7-4123-8a32-135d88840b4&title=&width=522)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668844517666-d9496843-6ab9-40c7-bb93-078120057f60.png#averageHue=%23949585&clientId=uaf2cb688-a680-4&from=paste&height=249&id=ue9b9e9ff&originHeight=249&originWidth=409&originalType=binary&ratio=1&rotation=0&showTitle=false&size=169716&status=done&style=none&taskId=ubfc30356-9465-47ec-adc1-2dfd84cb5a0&title=&width=409)<br />每次移动之后检测当前所处地面的行走状态，如果地面不可行走的话进入其他状态，但是上图所示的临界状态比较特殊，之前想的是这种地面不可行走那就直接进入 PhysFallingSim 状态，但是由于前胶囊体并没有完全脱离地面，向下打射线检测是否着陆的话必定是能打到碰撞物体，然后就会出现一种状态是 载具卡在那里 然后在 PhysWalkingSim  和 PhysFallingSim 之间不停的切换，然后就根据情况区分加入了一个新的状态   滑落状态（PhysSliderSim）,这个状态对应的人的状态其实是 PhysFalling，也就是人的坠落状态，但其实人的滑落走的也是坠落 Falling，当载具开到这里如果不能继续行走的话就会进入 飞出 或者 滑落状态，然后目前测试好像没出现之前的卡在那里不动得现象。
<a name="wkYxP"></a>
### 1.3 低速正面下直角坡（PhysMoveToFallingSim）
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669015904069-6bbd1513-76bc-4097-ab58-1910e98b48f9.png#averageHue=%23d4c4ad&clientId=ubc73ec77-f8b8-4&from=paste&height=410&id=SNTRN&originHeight=410&originWidth=1349&originalType=binary&ratio=1&rotation=0&showTitle=false&size=855615&status=done&style=none&taskId=u567cb9ff-91ee-461f-9c9d-ba3cc21bae4&title=&width=1349)<br />因为目前来说我们的根组件其实就是上图红框里的胶囊体，当载具在当前的位置继续向前前进的时候会根据速度出现种情况，第一种就是载具当时跑道这种地形的时候载具的速度比较高，然后呢这一段距离很快就位移过去了，基本没什么问题，另一个情况就是速度非常的低以一个很低的速度下直角坡，这个时候可以预见的问题是根胶囊体已经没有着陆点了进入了 PhysFallingSim 模拟下落状态，但是呢摩托车的胶囊体模型还在地面上，如果不进行处理的话就会出现下面的情况<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669016798346-c387e12b-bb10-4c61-b206-f8598945f90e.png#averageHue=%23a89379&clientId=ubc73ec77-f8b8-4&from=paste&height=256&id=ue8fa1da2&originHeight=362&originWidth=892&originalType=binary&ratio=1&rotation=0&showTitle=false&size=570778&status=done&style=none&taskId=uc893874e-9ab2-4de9-bd0a-a5d8971a4d9&title=&width=632)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669016816623-9b593922-52c0-4e7a-95e9-17988fd1db32.png#averageHue=%23a48e76&clientId=ubc73ec77-f8b8-4&from=paste&height=255&id=u42589852&originHeight=320&originWidth=814&originalType=binary&ratio=1&rotation=0&showTitle=false&size=456069&status=done&style=none&taskId=uc87990d7-bf51-4d94-8e29-96e6aa8d48c&title=&width=648)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669016836613-ac207f45-7651-4e6d-9432-172b2e5daa40.png#averageHue=%23ae9c89&clientId=ubc73ec77-f8b8-4&from=paste&height=278&id=u2accefb2&originHeight=278&originWidth=786&originalType=binary&ratio=1&rotation=0&showTitle=false&size=365441&status=done&style=none&taskId=u24595ac1-2417-4dc9-ae09-38b9861ee18&title=&width=786)<br />模型穿模就非常严重，然后这种情况目前的处理方式是 加入一个过渡状态 PhysMoveToFallingSim, 意思就是从 地面到空中的过渡状态，这个状态的基本逻辑是：判断是否能检测到地面，检测到地面的话就退出此状态进入PhysWalkingSim，检测不到的话就开始以前胶囊体向后打半个胶囊体高度的射线<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669020644137-910c45bf-edbd-4d88-8a56-17a9212bff4d.png#averageHue=%23b19c88&clientId=ubc73ec77-f8b8-4&from=paste&height=363&id=u1367e0a0&originHeight=363&originWidth=612&originalType=binary&ratio=1&rotation=0&showTitle=false&size=400083&status=done&style=none&taskId=u9300df24-0156-4e61-852e-87e51614764&title=&width=612)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669022430346-75e89506-d35b-4eba-a5c4-6a0c453839cc.png#averageHue=%232e2d2c&clientId=ubc73ec77-f8b8-4&from=paste&height=139&id=u54f5d974&originHeight=139&originWidth=629&originalType=binary&ratio=1&rotation=0&showTitle=false&size=13715&status=done&style=none&taskId=u4266525a-acad-4c8d-a5ba-ddf91451424&title=&width=629)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669022569341-95951ece-cb0e-4067-90e5-32b2d19d903e.png#averageHue=%23312f2f&clientId=ubc73ec77-f8b8-4&from=paste&height=41&id=u7d7c7a4d&originHeight=41&originWidth=1204&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12099&status=done&style=none&taskId=ub0540dd1-0d86-458a-9d75-8e47d3dcb6f&title=&width=1204)<br />分别检测能否打中地面 和 是否是载具的前半部分打的射线击中了地面，如果是前半部分击中了地面的话相当于模拟载具本身的重量卡在悬崖边的效果，避免车头出去一点就向下坠落显得比较假，还有一点就是这个状态的时候玩家还是可以按下后退按键后退的，然后如果车前半部分出了这个断崖面的话为了避免载具速度较小给了一个比较小的额外速度 MoveToFallExtraSpeed,同时使用特殊的重力参数避免载具下落速度过快，相当于尽量增加载具的滞空时间，然后在载具进入飞出的一瞬间记录根胶囊体的位置，之后在载具完全进入滞空状态之前的时候每帧获取根胶囊体的位置，通过这两个位置实时更新载具的俯仰角Pitch，进一步避免穿模的现象，但是不完善的地方是遇到倾斜角比较大的地方还是会有一些穿模。
<a name="pgaTi"></a>
### 1.4 低速背面下直角坡（PhysMoveToFallingBackSim）
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669023934889-e0e8674b-5be1-4326-8b0d-f67f232492db.png#averageHue=%23c0b09b&clientId=ubc73ec77-f8b8-4&from=paste&height=439&id=uda4b8864&originHeight=439&originWidth=1730&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1143359&status=done&style=none&taskId=uf6e5985d-fcc0-4d1b-b905-b7ca3834a48&title=&width=1730)<br />背面下这种坡度的地形的时候其实比正面下感觉处理的时候更加的奇怪就是逻辑上不复杂但是呢就是不符合载具之前的运动设定（前胶囊体带动俯仰角改变），正常来讲如果不处理的话他的运动结果应该是像下图一样：<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669024145131-2ef2464a-e6d3-41b9-803b-80dc12d89b86.png#averageHue=%23a28e75&clientId=ubc73ec77-f8b8-4&from=paste&height=254&id=u8f8cef5d&originHeight=267&originWidth=733&originalType=binary&ratio=1&rotation=0&showTitle=false&size=348642&status=done&style=none&taskId=ued8796df-f46e-4018-af2c-b53d18bf294&title=&width=696)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669024162903-a2fd5f05-9601-49f8-86ee-dc966a6078ff.png#averageHue=%23907d66&clientId=ubc73ec77-f8b8-4&from=paste&height=251&id=uf292e6cb&originHeight=251&originWidth=654&originalType=binary&ratio=1&rotation=0&showTitle=false&size=297388&status=done&style=none&taskId=uc9fe4f31-831d-46b3-a003-78e66749616&title=&width=654)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669030240760-2ca8a827-a059-4e3a-b4b2-6c1bc6742fe1.png#averageHue=%23958370&clientId=ubc73ec77-f8b8-4&from=paste&height=182&id=u73d8def4&originHeight=182&originWidth=420&originalType=binary&ratio=1&rotation=0&showTitle=false&size=146653&status=done&style=none&taskId=udd65aa6b-9c2b-4090-a8d8-d4bf848c0e1&title=&width=420)<br />就当根胶囊体离开地面之前的时候载具完全就悬空了等到根胶囊体开始下坠载具整体也跟着下坠，比较诡异，然后这个时候如果按照载具以前的逻辑调整根胶囊体的俯仰角就会出现载具车身中间部分卡在模型里面，所以就不能按照之前的俯仰角调整逻辑来处理这个事情，如下图分别单独调整前后胶囊体的俯仰角<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669029797282-e75ef7b4-02b2-44fb-bdba-225ea21e1f8d.png#averageHue=%23a38f79&clientId=ubc73ec77-f8b8-4&from=paste&height=215&id=u5472e7de&originHeight=215&originWidth=407&originalType=binary&ratio=1&rotation=0&showTitle=false&size=160187&status=done&style=none&taskId=u5d63310d-832a-4d0e-9bab-9f83e3b4e79&title=&width=407)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669029851084-6da04ed8-f906-463f-9cc1-7e4d66ef9ae0.png#averageHue=%23ab9c88&clientId=ubc73ec77-f8b8-4&from=paste&height=214&id=u9f765188&originHeight=225&originWidth=497&originalType=binary&ratio=1&rotation=0&showTitle=false&size=195869&status=done&style=none&taskId=uca96570f-f92f-494f-b89b-72d9f7e414d&title=&width=472)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669029882161-c3748ec9-b493-46a5-bc80-81c1c38d2b00.png#averageHue=%23ac9c8a&clientId=ubc73ec77-f8b8-4&from=paste&height=214&id=ub34fc428&originHeight=234&originWidth=482&originalType=binary&ratio=1&rotation=0&showTitle=false&size=204858&status=done&style=none&taskId=u45ae74fa-a33e-426d-a795-4738f8cb2c0&title=&width=441)<br />第一张图是只调整了前胶囊体的俯仰角的效果，可以看到有一部分模型穿模了，第二张图只调整了后胶囊体的俯仰角可以看到的是载具模型悬空了，第三张图是同时调整了前后胶囊体相对于父节点的相对偏移，效果互相补充看着就还好，也有一部分原因是摆的位置和修改的角度参数比较契合的原因，在实际的运行效果上，由于也一个后退的额外速度参数来控制，所以看起来的效果基本上就是有一个模型后仰的趋势动作，但不会后仰太多车子就进入坠落状态然后恢复正常的角度。
<a name="e1zvj"></a>
### 1.5 载具俯仰角计算逻辑
<a name="MXv2Z"></a>
#### 1.51穿模问题
目前摩托车的俯仰角计算是通过前后胶囊体向下打 capsule 射线确定两点然后通过这两点来确定俯仰角，但是有的时候会有一些特殊的情况，就比如之前遇到的一种情况：![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669085433203-898c59ce-5587-4f0f-8317-bbf6b1b2208d.png#averageHue=%23a88d6c&clientId=u6cbb5685-2bd6-4&from=paste&height=202&id=u5f15c1fa&originHeight=198&originWidth=373&originalType=binary&ratio=1&rotation=0&showTitle=false&size=145370&status=done&style=none&taskId=udeb84d6c-0209-4885-875e-d4771e9ecd0&title=&width=380)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669085853106-a6d6a20b-77ac-4240-81e0-9899d75bf8c0.png#averageHue=%23a18360&clientId=u6cbb5685-2bd6-4&from=paste&height=202&id=ucfec3787&originHeight=248&originWidth=370&originalType=binary&ratio=1&rotation=0&showTitle=false&size=182408&status=done&style=none&taskId=ue72437ae-a9c6-4279-8c0e-2bdcd188785&title=&width=301)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669085875871-826a14cf-bfb6-4f76-b0c3-ee917fd89877.png#averageHue=%23a2865f&clientId=u6cbb5685-2bd6-4&from=paste&height=203&id=u2aa95180&originHeight=203&originWidth=336&originalType=binary&ratio=1&rotation=0&showTitle=false&size=147390&status=done&style=none&taskId=u014f826a-7f61-4c4d-87f3-87f9e62b7ca&title=&width=336)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669086266711-d3d4da33-fbf1-4c41-8027-ee971932b3ea.png#averageHue=%23bfa98f&clientId=u6cbb5685-2bd6-4&from=paste&height=204&id=ue8579692&originHeight=180&originWidth=325&originalType=binary&ratio=1&rotation=0&showTitle=false&size=104861&status=done&style=none&taskId=u25d0f1ec-a91c-4666-9381-4869a1e6ee7&title=&width=368)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669086912976-48ddff72-5fd9-4621-870c-d993de5b14ae.png#averageHue=%23bca791&clientId=u6cbb5685-2bd6-4&from=paste&height=249&id=MtVF5&originHeight=287&originWidth=429&originalType=binary&ratio=1&rotation=0&showTitle=false&size=208926&status=done&style=none&taskId=u8e08142a-b795-4e72-b2dd-0e0c7256ebc&title=&width=372)<br />就是在上图状态的时候，之前还没做倒车下坡的状态，所以如果以一种极低的速度通过的时候就可能产生这种情况，图一后轮胶囊体其实在悬空的时候检测不到地面然后就不能通过前后点来确定俯仰角，就只能用前轮胶囊体的射线法线来检测，但是呢它这个法线也不能直接使用，图二和图三就是直接使用法线导致的问题，原因是它的碰撞可能在图四所示的侧面，然后当时排除了这种情况后，有补充了一种额外保底的计算方式，就是图五这样，从载具中心打几个射线取 法线 的和的法线用来计算（图示 我调整了下打射线的位置，因为现在基本复现不出来这种情况，为了方便截图把射线起点放到了后边），然后如果算的值还有问题的话就保持当前角度不改变。<br />下图是现在的实际情况，在坡度比较低的时候会通过调整车身角度 和 载具本身的尾部减震结构让轮胎尽量往下压到策划配置的最大值，然后如果高度再高点的话就会进入 PhysMoveToFallingBackSim 状态，走它的逻辑。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669085281648-f10ae111-ed33-4506-b037-f6bf77e55e10.png#averageHue=%23b2aa9f&clientId=u6cbb5685-2bd6-4&from=paste&height=361&id=u43eb5be8&originHeight=361&originWidth=650&originalType=binary&ratio=1&rotation=0&showTitle=false&size=349087&status=done&style=none&taskId=ud0d532ea-a719-4f87-8d0a-a924de10ac6&title=&width=650)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669085349373-f3e1a81a-73c1-4d8e-ae2b-f1fdb2d1c15c.png#averageHue=%23b3aba0&clientId=u6cbb5685-2bd6-4&from=paste&height=362&id=ue9b6ad93&originHeight=370&originWidth=616&originalType=binary&ratio=1&rotation=0&showTitle=false&size=319816&status=done&style=none&taskId=u5c99e660-3097-4e10-aece-c5a2d9f389f&title=&width=603)
<a name="ulkAA"></a>
#### 1.52 数值应用问题
还有一个问题就是实时计算出来的载具俯仰角的过渡问题，如果直接使用计算结果非常明显的问题就是载具的俯仰角变换过于的剧烈，表现出来的效果就是非常的抖，但是如果不使用实时计算出来的数据的话在地形变化比较剧烈的地面上行驶的时候载具就会穿模，这个问题感觉没什么能完美解决的方法，现在采用的方式是 区分单次俯仰角变化 差值，就是比如：单次角度变化 0-5 用的一条曲线的数据，5-10 用一条曲线的数据，10以上再用一条曲线的数据，然后曲线 的 X 轴是载具速度，Y轴是插值系数，这样的话在载具速度比较小的时候策划配置一个相对较大的 插值 系数看起来不会穿模，然后因为速度相对较慢跨越的地形距离也比较短抖动也相对还好，然后在速度比较快的时候策划配置一个相对比较小的系数，这样跑起来的时候看起来抖动比较小，然后也因为速度相对较快，运动较快也能很快离开会穿模的地形。<br />然后就是 前进 和 后退的逻辑其实也是有区别的，前进的时候用的就上边的逻辑插值计算，这是因为胶囊体在载具最前方，前进的时候能永远保持前轮模型在地面上，但是后退的时候不一样，后退的时候只能采用实时的计算结果，不然倒退遇到骤变的坡度的时候尾部如果采用插值计算的话会明显穿模，而且倒车速度比较低，所以采用实时计算的角度也没什么问题。 
<a name="hONmY"></a>
### 1.6 贴墙转向离开
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669105562270-bdf51b87-37ce-4900-b072-700c11abebe1.png#averageHue=%23af9878&clientId=u6cbb5685-2bd6-4&from=paste&height=246&id=xPmZd&originHeight=231&originWidth=374&originalType=binary&ratio=1&rotation=0&showTitle=false&size=177797&status=done&style=none&taskId=u98b082b8-79c9-401a-9652-b92dc6817bf&title=&width=398)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669104638401-ca41ea65-4a08-428e-87f5-dded5d0fab04.png#averageHue=%23596d5c&clientId=u6cbb5685-2bd6-4&from=paste&height=246&id=ucb88bb7d&originHeight=246&originWidth=406&originalType=binary&ratio=1&rotation=0&showTitle=false&size=173441&status=done&style=none&taskId=u753b4144-e306-4e49-baa1-3d4d5e6472f&title=&width=406)![贴墙转向.gif](https://cdn.nlark.com/yuque/0/2022/gif/26747865/1669108219783-370382c4-bbe4-4262-b8e7-2a83e034829c.gif#averageHue=%23a09e84&clientId=u6cbb5685-2bd6-4&from=paste&height=246&id=u3d7782bf&originHeight=763&originWidth=1246&originalType=binary&ratio=1&rotation=0&showTitle=false&size=3300875&status=done&style=none&taskId=uc43e5a5d-64ff-4356-a368-d1f378957c2&title=&width=401)<br />现在这个贴墙处理的方案还是不是很好，之后估计策划还得重新定方案或者向其他方法来完善逻辑。<br />现在载具的贴墙处理逻辑是当载具从图一所示的状态一直按着前进按键载具会自动贴墙偏转，而且最大也只能是水平于墙面，类似图二的效果，那么这个时候再按着前进按键载具会贴墙前进，但是玩家的操作可能是按着右转向按键想要转出，但是按照我们正常的运动逻辑是没办法做到的(和正常的物理载具不一样，物理载具运动的方向是载具车头的方向，但是我们的运动完全是被胶囊体带着运动的，没办法让胶囊体跟随车头动画动态转向)所以就只能用其他的方式，现在的处理方式是补充了额外逻辑如果贴墙的角度和速度同时小于策划的配置值且玩家按着远离墙面的方向按键的时候,会给载具一个正右方向的额外的速度，这个速度比较小，帮助载具胶囊体稍微离开墙面的同时也能减少转向的时候尾部穿模的问题，并且给了一个额外的速度转向角把速度的方向向离开墙面的方向进行额外转向，然后再计算单次位移，基本能模拟出载具的转向位移效果。
<a name="X2bVe"></a>
## 2.模拟正面从 PhysWalkingSim到坠落PhysFallingSim 即 PhysMoveToFallingSim
详见 1.3
<a name="NkJUW"></a>
## 3.模拟背面从 PhysWalkingSim到坠落PhysFallingSim 即 PhysMoveToFallingBackSim
详见 1.4
<a name="vlhgg"></a>
## 4.模拟坠落 PhysFallingSim
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669347849344-9054dba6-fca4-412b-bc77-d0da6cdf2edf.png#averageHue=%23fefdfc&clientId=u5d3a4b09-191a-4&from=paste&height=450&id=uc62b6d43&originHeight=450&originWidth=248&originalType=binary&ratio=1&rotation=0&showTitle=false&size=12859&status=done&style=none&taskId=u65de7f60-fa76-4880-9719-8ca02231b09&title=&width=248)<br />坠落的时候和行走的时候不一样不需要计算加速度，在空中的时候速度 只有Z轴速度会跟随滞空时间变化，玩家在空中的操作只能向左或者向右摇摆车头，在CalcVehicleFlyData里计算，在空中的时候只修改载具的朝向不修改载具的速度方向，保持载具原先的运行轨迹，在载具落地的时候开始速度向载具朝向回归模拟落地滑车的效果，然后在载具完全滞空之前还需要一直检测载具的车身碰撞调整载具的俯仰角，等到载具完全滞空后，开始自动调整载具的俯仰角（Pitch）和翻滚角（Roll）到策划配置的数据。<br />处理落地的时候遇到一个问题，就是载具尾部先落地，但是此时根胶囊体还是没有落地的，这样就会导致尾部穿模，现在处理方式的每帧检测前后轮是否落地，如果后轮先落地的话就在落地的瞬间修改载具的俯仰角不让其穿模。
<a name="ivS11"></a>
## 5.模拟滑落 PhysSliderSim
这个状态的实现逻辑基本和人物坠落 PhysFalling 一致，暂时只是用来处理一些特殊状态，比如下图滑落的时候。这个没啥特别的。有兴趣的可以直接看人的坠落逻辑。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1668844517666-d9496843-6ab9-40c7-bb93-078120057f60.png#averageHue=%23949585&clientId=uaf2cb688-a680-4&from=paste&height=249&id=sXLwe&originHeight=249&originWidth=409&originalType=binary&ratio=1&rotation=0&showTitle=false&size=169716&status=done&style=none&taskId=ubfc30356-9465-47ec-adc1-2dfd84cb5a0&title=&width=409)
<a name="ylCqT"></a>
# TS部分逻辑
<a name="oYmxu"></a>
## 1.寻路
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669362002311-d752bede-2052-454c-9715-c0cd1680cd7e.png#averageHue=%23403e3e&clientId=u5d3a4b09-191a-4&from=paste&height=250&id=ud814a642&originHeight=250&originWidth=505&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15898&status=done&style=none&taskId=u5d5c5455-c05a-4c64-86cb-03fa3b40c98&title=&width=505)<br />寻路策划要求可以自动避障走到上下车点，但是对于动态物体来说 Nav寻路没办法判断，所以如果在路上有动态物体挡在路上寻路就会被卡着，现在的方法是在载具周围摆几个上车点，然后统计所有通过周围的过渡点能够到达上车点的路线，再选取最短的路线过去。
<a name="eZONU"></a>
## 2.特效挂点
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669366673130-2c716696-aba1-466b-acb1-1671b8e93dac.png#averageHue=%23171816&clientId=u5d3a4b09-191a-4&from=paste&height=224&id=u2db0b641&originHeight=224&originWidth=501&originalType=binary&ratio=1&rotation=0&showTitle=false&size=127239&status=done&style=none&taskId=ua0583970-305e-4429-a8f6-c09b26a83aa&title=&width=501)![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1669366683096-12d75c98-0383-4886-a650-f8ac7628d12d.png#averageHue=%23181a17&clientId=u5d3a4b09-191a-4&from=paste&height=223&id=u9b5d218f&originHeight=240&originWidth=523&originalType=binary&ratio=1&rotation=0&showTitle=false&size=136168&status=done&style=none&taskId=u3ff2ef9b-dae1-4466-b23a-0b15e1d6352&title=&width=486)<br />有的特效需要挂载在轮胎底部和地面接触的地方，但是因为没办法保证载具模型的后轮胎一直在地面上，如果直接把特效组件挂载在模型上就会出现特效组件在地面或者悬在空中穿帮的情况，所以添加了几个 SceneComponent 从轮胎骨骼向下打射线，取 HitLocation 向上偏移一小段距离的位置作为轮胎和地面的接触位置，然后把特效组件挂载在在这个组件上就可以保证特效不会断开（比如车胎痕迹）
<a name="h7ipQ"></a>
## 3.特效切换
在进入不同的材质地面上切换不同的特效效果，直接切换特效的话会导致之前的特效直接消失，比较突兀，可以新建一个特效组件挂载，只要之前的组件被 deactive 等到特效消失之后组件就会自己被回收，之前不知道这个特性，费了很大劲感觉效果还不是很好，换这种方式的话会方便的多。
<a name="wBgaU"></a>
# 类逻辑
<a name="NBKzy"></a>
## 基础继承关系
![](https://cdn.nlark.com/yuque/0/2023/jpeg/26747865/1675932971323-333600b0-c4bd-45ae-8a70-35feb8e0d4fd.jpeg)
<a name="RpAtX"></a>
## 程序逻辑
玩家：<br />添加一个载具交互组件  TS_CallVehicleComponent 继承自 DriveSystemComponent 拥有属性如下图，目前上车请求只能 上自己拥有的载具（暂时没有具体上车需求）根据 OwnVehicleID 获取载具，然后本地检测可以到达的上车点位置并往其位移，然后通过人物的移动同步实现玩家同步到上车点，到达上车点之后发送请求载具上车协议  CSRideVehicle(vehicleId: BigInt, loc: number, bIsGetIn: boolean) 然后服务器广播玩家状态变更协议 客户端会根据收到的协议带来的  上车点位置信息 决定播放哪个动作，这个协议还需要根据需求完善，需要加上 请求的玩家ID（现在服务器默认只能上自己召唤的车所以没加请求的玩家ID），然后最终上哪个载具需要由服务器决定，现在是本地检测距离最近的座位，然后控制权由座位决定，现在是谁拥有谁可以控制(需要具体需求)<br />上下车动画：保存在 TS_CallVehicleComponent 组件中，根据 玩家sn_载具sn 读取配置设置<br />直接挂载玩家到载具上(无动画过程)：AttachToVehicle  适用于上线的时候 玩家已经在 载具上了 + 载具进入视野重新创建的时候<br />AttachToVehicle：本地查找空闲座位 + 判断是否可以由本上车点上去(走配置) + 挂载玩家到载具上<br />交互上车(有动画过程)：玩家跟载具互动上下车<br />OnDealVehicleInteract：根据上下车决定用哪个动画(这个需要梳理下映射关系) + 播放蒙太奇 + 蒙太奇播放结束处理镜头和控制权（控制权暂时只给了载具的召唤者）（需要去掉 本地保存的骑乘玩家引用，根据座位控制权从座位来那objid获取actor）<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/26747865/1675934136112-d74b952a-8ffc-4ac0-8c51-b5ff503003b0.png#averageHue=%232e2d2c&clientId=u2a553488-0ba0-4&from=paste&height=426&id=u10300550&originHeight=426&originWidth=414&originalType=binary&ratio=1&rotation=0&showTitle=false&size=38308&status=done&style=none&taskId=u70bcebca-6bea-4151-a0be-d5584810600&title=&width=414)

通用参数结构：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/26747865/1675933617953-186a62e6-6986-4b0e-8780-d463581b816c.png#averageHue=%232d2c2b&clientId=u2a553488-0ba0-4&from=paste&height=633&id=u90cac3a6&originHeight=633&originWidth=774&originalType=binary&ratio=1&rotation=0&showTitle=false&size=97440&status=done&style=none&taskId=uab3a5ca7-9349-4335-9200-7b17231a34a&title=&width=774)![image.png](https://cdn.nlark.com/yuque/0/2023/png/26747865/1675933699999-1ae47059-1ed9-4c61-85ee-3bb47d03860f.png#averageHue=%232d2c2b&clientId=u2a553488-0ba0-4&from=paste&height=509&id=u3ef839fe&originHeight=509&originWidth=681&originalType=binary&ratio=1&rotation=0&showTitle=false&size=66729&status=done&style=none&taskId=u23e3f86f-6b3b-44f2-bbec-abcce00696e&title=&width=681)
