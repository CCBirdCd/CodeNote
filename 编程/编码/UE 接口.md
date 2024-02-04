<a name="4ed9ed9b"></a>
## 功能点实现接口
<a name="vRofo"></a>
### C++打印字符串到屏幕
```cpp
#include"Engine.h" 
GEngine->AddOnScreenDebugMessage(-1,20, FColor::Red,FString(TEXT("test1")));

#include"Kismet/KismetSystemLibrary.h"UKismetSystemLibrary::PrintString(this,TEXT("Successed Produce NPC:%d"),Num);

//ProductIntroduce为FString类型，需要用*转换成从const char*才能用
UE_LOG(LogTemp, Warning, TEXT("read by traversal --- introduct:%s"), *(pRow->ProductIntroduce));
```

<a name="He5mA"></a>
### FString
> - FString::FromInt()  Int 转换为 String
> 
 

<a name="FMath"></a>
### FMath

- FMath::Clamp(a, 1.0, 2.0): 确保a在 1.0 -2.0 之间 a<1.0 返回 1.0, a>2.0 返回2.0
<a name="lCjIj"></a>
### 插值算法
> - FMath::Lerp():  Lerp 插值函数 在两个数据中间计算出 中间数据 使得过度较为平滑
> - FMath::FInterpTo( T1 Current, T2 Target, T3 DeltaTime, T4 InterpSpeed )
>    - 百分比插值 插值结果如下，插值效果先快后满
>    - const RetType DeltaMove = Dist * FMath::Clamp<RetType>(DeltaTime * InterpSpeed, 0.f, 1.f);<br />return Current + DeltaMove
> - 弹簧插值

```cpp
	/**
	 * Uses a simple spring model to interpolate a float from Current to Target.
	 *
	 * @param Current               Current value 当前值
	 * @param Target                Target value 目标值
	 * @param SpringState           Data related to spring model (velocity, error, etc..) - Create a unique variable per spring
	 * @param Stiffness             How stiff the spring model is (more stiffness means more oscillation around the target value)
	 * 								弹簧刚度，影响插值过程中弹簧的反应速度。值越大，插值过程越快
     * @param CriticalDampingFactor How much damping to apply to the spring (0 means no damping, 1 means critically damped which means no oscillation)
	 *								阻尼系数，用于控制弹簧摆动的速度衰减。值越大，摆动衰减得越快，物体越快达到静止状态 
     * @param DeltaTime             Time difference since the last update
	 * @param Mass                  Multiplier that acts like mass on a spring
	 * @param TargetVelocityAmount  If 1 then the target velocity will be calculated and used, which results following the target more closely/without lag. Values down to zero (recommended when using this to smooth data) will progressively disable this effect.
	 * @param bClamp                Whether to use the Min/Max values to clamp the motion
	 * @param MinValue              Clamps the minimum output value and cancels the velocity if it reaches this limit
	 * @param MaxValue              Clamps the maximum output value and cancels the velocity if it reaches this limit
	 * @param bInitializeFromTarget If set then the current value will be set from the target on the first update
	 */
	UFUNCTION(BlueprintCallable, Category = "Math|Interpolation", meta=(AdvancedDisplay = "8"))
	static float FloatSpringInterp(float Current, float Target, UPARAM(ref) FFloatSpringState& SpringState,
	                               float Stiffness, float CriticalDampingFactor, float DeltaTime,
	                               float Mass = 1.f, float TargetVelocityAmount = 1.f, 
	                               bool bClamp = false, float MinValue = -1.f, float MaxValue = 1.f,
	                               bool bInitializeFromTarget = false);
```
>  

<a name="6d0d5d0a"></a>
### 宏
> - UPROPERTY(EditAnywhere, Category="Count")  // 公开变量到编辑器 EditAnywhere表示该属性可以在原型和实例上的属性窗口中编辑, Category 用于设置展开设置的节点名称
> - UFUNCTION(BlueprintNativeEvent)  // 公开函数到蓝图

<a name="32f26205"></a>
### 3D场景大小缩放
```cpp
// 思路:按住按键开始缩放 松开还原 
// 网格体放大缩小
{
    float CurrentScale = StaticMeshComp->GetComponentScale().X;
    if  (bGrowing)
    {
        CurrentScale += DeltaTime;
    }
    else
    {
        CurrentScale -= (DeltaTime * 0.5);
    }
    CurrentScale = FMath::Clamp<float>(CurrentScale, 1.0, 5.0);
    StaticMeshComp->SetWorldScale3D(FVector(CurrentScale));
	}
```

<a name="dd3082ac"></a>
### 绑定动作 和 偏移轴(多用于输入事件)
```cpp
PlayerInputComponent->BindAction("Grow", IE_Released, this, &APawnWithCamera::StopGrow);
PlayerInputComponent->BindAxis("MoveForward", this, &APawnWithCamera::MoveForward);
```

<a name="2ff728e2"></a>
### 文本显示
```cpp
// 初始化文本显示倒计时
UTextRenderComponent* CountDownText = CreateDefaultSubobject<UTextRenderComponent>(TEXT("CountDownNumber"));
CountDownText->SetHorizontalAlignment(EHTA_Center);
CountDownText->SetWorldSize(150.0f);
CountDownText->SetText(FString::FromInt(FMath::Max(CountDownTime, 0)));
```

<a name="fec3d6e4"></a>
### 定时器
```cpp
// 创建定时器句柄并绑定
FTimerHandle CountDownTimeHandle;
GetWorldTimerManager().SetTimer(CountDownTimeHandle, this, &ACountDown::AdvanceTimer, 1.0f, true
                               );
//删除定时器句柄
GetWorldTimerManager().ClearTimer(CountDownTimeHandle);
```

<a name="4fcc4a59"></a>
### 将函数公开到蓝图
```cpp
.h
UFUNCTION(BlueprintNativeEvent)     // 声明
void CountDownHasFinished();        // 定义无需实现
virtual void CountDownHasFinished_Implementation();  //具体实现函数
.cpp
void ACountDown::CountDownHasFinished_Implementation()
{
    CountDownText->SetText(TEXT("Countdown BP"));
}
调用:
CountDownHasFinished();
//调用的话调用 CountDownHasFinished() 不调用 CountDownHasFinished_Implementation()
//将函数公开给蓝图 UFUNCTION() 宏负责处理将C++函数公开给反射系统。BlueprintCallable 选项将其公开给蓝图虚拟机。每一个公开给蓝图的函数都需要一个与之关联的类别，这样右键点击快捷菜单的功能才能正确生效
UFUNCTION(BlueprintCallable, Category="Damage")
    void CalculateValues();
```

<a name="57c2062e"></a>
### 事件(定义、分发、绑定)
> - 蓝图实现:<br />① 定义事件：在蓝图编辑器左下角 Event Dispatchers 点击 + 添加事件，添加完成后拖拽该事件到UI上选择 call 在合适时机调用<br />![](media/image_10.png)<br />![](media/image_11.png)<br />② 绑定事件:从节点中拖出一条引线，然后搜索并选择 Bind Event to XXXXX<br />在 细节（Details） 面板上，点击 变量类型（Variable Type） 下拉菜单，然后搜索并选择 BP_BossDied 的 对象引用（Object Reference）。勾选 实例可编辑（Instance Editable） 复选框。<br />![](media/image_29.png)<br />![](media/image_33.png)


<a name="J6eZf"></a>
## 蓝图
<a name="AGZ4N"></a>
### 通用蓝图节点

1. 判断某个按键是否按下 : GetPlayControl -> IsInputKeyDwon
2. 动画状态机判断某个动画播放剩余时间：GetRelevantAnimTimeRemaining(动画状态机名称)
3. 设置actor是否可以碰撞：SetActorEnableCollision
<a name="wl5Da"></a>
### 蓝图函数
1.传值 和 传引用  区别在于 输入参数是否勾选  Pass-by-Reference， 如果想要在蓝图函数内改变<br />传入的变量，则需要勾选该项，勾选后节点变成菱形标识为引用，修改参数的节点为  set by ref <br />![蓝图函数.PNG](media/蓝图函数.PNG.png)

<a name="r7ftU"></a>
### 射线检测
LineTraceByChannel<br />![射线检测.PNG](media/射线检测.PNG.png)
<a name="kKjrA"></a>
## c++接口
<a name="eMe6d"></a>
### 移动:MoveTemp
类似于 std:move

<a name="OCgEe"></a>
### 添加 Component
![image.png](media/image-15.png)

<a name="U58W7"></a>
### 加载 资源、类、蓝图
UParticleSystem* commonParticleAsset = LoadObject<UParticleSystem>(NULL, TEXT("'/Game/StarterContent/Particles/P_Fire.P_Fire'"));<br />if (commonParticleAsset)<br />{<br />   ProjectileParticleCmpt->SetTemplate(commonParticleAsset);<br />}

![image.png](media/image-16.png)
<a name="JCL8q"></a>
#### 代码方式如何Spawn蓝图类
第一种：/
```cpp
//SpawnActor用法
FString sPath = "/Game/Blueprints/Actor/RuntimeActor/RuntimeCameraBP.RuntimeCameraBP_C";
 
FVector vDir = GGameInstance->Player->GetActorForwardVector();
//vDir.Z = 0;
FVector vLocation = GGameInstance->Player->GetActorLocation() + vDir * 1000;
FActorSpawnParameters params;
params.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AdjustIfPossibleButAlwaysSpawn;
UObject* pObject = UUtilsLibrary::CreateAsset(sPath);
UBlueprintGeneratedClass* BP = Cast<UBlueprintGeneratedClass>(pObject);
	
auto pActor = GWorld->SpawnActor<ASPRuntimeCamera>(BP, FTransform(vLocation), params);
```
第二种：
```cpp
/** Pivot Actor bp calss, in the blueprint set value. */
UPROPERTY(Category = "Dynamic Data (General Settings)", EditAnywhere, BlueprintReadWrite)
TSubclassOf<ASPPivotMeshActor> PivotActorBP;
 
//How to Spawn?
SPPivotMeshActor = GWorld->SpawnActor<ASPPivotMeshActor>(PivotActorBP, FTransform(CenterPosition));
```
<a name="hi5cU"></a>
#### 通过构造加载方式
```cpp
//静态方法, 加载uasset的资源，比如UI贴图等。建议写到继承自UBlueprintFunctionLibrary的类中
UFUNCTION(BlueprintCallable, Category = "Utils")
static UObject* CreateAsset(const FString& AssetPath);
 
//实现
UObject* UUtilsLibrary::CreateAsset(const FString& AssetPath)
{
	FStringAssetReference ref = AssetPath;
	UObject* uoTmp = ref.ResolveObject();
	if (uoTmp == nullptr)
	{
		UE_LOG(LogTemp, Log, TEXT("CreateAsset path = %s"), *AssetPath);
		FStreamableManager& EKAssetLoader = GGameInstance->GetStreamableManager();
		uoTmp = EKAssetLoader.LoadSynchronous(ref, true);
	}
 
	return uoTmp;
}
 
//使用案例1， 为UMG上面的UButton和UImage设置图片(直接代码写中文以及中文图片的命名方式的习惯不好，不要学我)
//UButton dynamic change normal image
UObject* pHovered_ContentAttach = UUtilsLibrary::CreateAsset(TEXT("Texture2D'/Game/Blueprints/UITextures/风险挂接-选中.风险挂接-选中'"));
Button_BottomType_Mode->WidgetStyle.Normal.SetResourceObject(pHovered_ContentAttach);
 
//UImage dynamic change normal image
FString sImagePath = TEXT("Texture2D'/Game/Blueprints/UITextures/添加摄像头.添加摄像头'");
UObject* pImage = UUtilsLibrary::CreateAsset(sImagePath);
RETURN_IF_NULL(pImage);
Image_CameraAdd->Brush.SetResourceObject(pImage);
 
 
 
//使用案例2， 结合CreateAsset在代码中创建一个UMG蓝图
 
// 对于ScrollBox……等类似列表容器可先使用该种方式, 创建然后再做添加
UFUNCTION()
UUserWidget* CreateFromAsset(const FStringAssetReference& StringRef, ESPPreviewType emUIType, bool bAddToViewPort, int32 nZorder = 0);
 
//实现
UUserWidget* USPUIManager::CreateFromAsset(const FStringAssetReference& StringRef, ESPPreviewType emUIType, bool bAddToViewPort, int32 nZorder)
{
	UUserWidget* pUI = nullptr;
	UObject* inObject = UUtilsLibrary::CreateAsset(StringRef.ToString());
	if (inObject)
	{
		pUI = CreateWidget<UUserWidget>(GGameInstance, Cast<UClass>(inObject));
		if (pUI)
		{
			if (bAddToViewPort)
			{
				pUI->AddToViewport(nZorder);
				m_WidgetCache.Add(pUI);
			}
 
			if (Cast<USPUserWidget>(pUI))
			{
				Cast<USPUserWidget>(pUI)->PreviewTypeArray.Add(emUIType);
			}
		}
	}
 
	return pUI;
}
 
//HowTo Use Sample
UUserWidget* pUW = GGameInstance->UIManager()->CreateFromAsset(StringRef, emUIType, false);
if(pUW) UGridSlot* pGSlot = SPGridPanel_Left->AddChildToGrid(pUW, row, column);
```


<a name="Z7wfz"></a>
#### 通过构造加载方式2
```cpp
// set default pawn class to our Blueprinted character 构造函数中加载
static ConstructorHelpers::FClassFinder<APawn> PlayerPawnBPClass(TEXT("/Game/ThirdPersonCPP/Blueprints/ThirdPersonCharacter"));
if (PlayerPawnBPClass.Class != NULL)
{
	DefaultPawnClass = PlayerPawnBPClass.Class;
}
```

<a name="VjKye"></a>
#### 通过构造加载方式3 LoadClass以及ConstructorHelpers::FClassFinder

```cpp
//.h中声明一下 加载一个蓝图类
UPROPERTY()
TSubclassOf<class AActor> BP_1;
 
//构造函数中实现, 加载一个蓝图类
BP_1 = LoadClass<AActor>(NULL, TEXT("Blueprint'/Game/BP/MeshBP_1.MeshBP_C'"));
 
 
//.h中声明一下 加载一个UMG
UPROPERTY()
TSubclassOf<UUserWidget> UIMain;
 
//构造函数中实现, 加载一个UMG, 这里可以注意并没有写_C
static ConstructorHelpers::FClassFinder<UUserWidget> MYWidget(TEXT("/Game/UMG/UI_Main"));
UIMain_Instance = MYWidget.Class;
```
<a name="hfypD"></a>
#### 通过构造函数内Load资源 进行资源加载  LoadObject
 
```cpp
// 红.h中声明一下
UPROPERTY()
UMaterialInstance* MaterialInstance_Level1;
 
 
// Sets default values 构造函数中通过LoadObject加载
UWorldActorsManager::UWorldActorsManager()
	: m_pSelectedActor(nullptr)
{
	mapMaterials.Reset();
 
	/*MaterialInstance_Level1 = LoadObject<UMaterialInstance>(NULL, TEXT("/Game/Material/ColorMatreial_Inst_1.ColorMatreial_Inst_1"), NULL, LOAD_None, NULL);
	MaterialInstance_Level2 = LoadObject<UMaterialInstance>(NULL, TEXT("/Game/Material/ColorMatreial_Inst_2.ColorMatreial_Inst_2"), NULL, LOAD_None, NULL);
	MaterialInstance_Level3 = LoadObject<UMaterialInstance>(NULL, TEXT("/Game/Material/ColorMatreial_Inst_3.ColorMatreial_Inst_3"), NULL, LOAD_None, NULL);
	MaterialInstance_Level4 = LoadObject<UMaterialInstance>(NULL, TEXT("/Game/Material/ColorMatreial_Inst_4.ColorMatreial_Inst_4"), NULL, LOAD_None, NULL);*/
}
```

<a name="QXFPf"></a>
#### 如何加载一张磁盘上的Png、jpg、jpeg、bmp到UMG上
```cpp
//如何加载一张磁盘上的Png、jpg、jpeg、bmp到UMG上 头文件
UFUNCTION(BlueprintCallable, Category = "Utils")
static UTexture2D* LoadTexture(FString fullPath);
 
//具体实现
UTexture2D* UUtilsLibrary::LoadTexture(FString fullPath)
{
	RETURN_NULL_IF_FALSE(FPaths::FileExists(fullPath));
 
	TArray<uint8> RawFileData;
	UTexture2D* MyTexture = NULL;
	RETURN_NULL_IF_FALSE(FFileHelper::LoadFileToArray(RawFileData, *fullPath));
	IImageWrapperModule& ImageWrapperModule = FModuleManager::LoadModuleChecked<IImageWrapperModule>(FName("ImageWrapper"));
	// Note: PNG format.
	EImageFormat format = EImageFormat::Invalid;
	if (fullPath.EndsWith(TEXT(".png")))
	{
		format = EImageFormat::PNG;
	}
	else if (fullPath.EndsWith(TEXT(".jpg")) || fullPath.EndsWith(TEXT(".jpeg")))
	{
		format = EImageFormat::JPEG;
	}
	else if (fullPath.EndsWith(TEXT(".bmp")))
	{
		format = EImageFormat::BMP;
	}
	TSharedPtr<IImageWrapper> ImageWrapper = ImageWrapperModule.CreateImageWrapper(format);
	RETURN_NULL_IF_FALSE(ImageWrapper.IsValid());
	RETURN_NULL_IF_FALSE(ImageWrapper->SetCompressed(RawFileData.GetData(), RawFileData.Num()));
	TArray<uint8> UncompressedBGRA;
	RETURN_NULL_IF_FALSE(ImageWrapper->GetRaw(ERGBFormat::BGRA, 8, UncompressedBGRA));
 
	// Create the UTexture for rendering
	UTexture2D* result = UTexture2D::CreateTransient(ImageWrapper->GetWidth(), ImageWrapper->GetHeight(), PF_B8G8R8A8);
 
	// Fill in the source data from the file
	void* TextureData = result->PlatformData->Mips[0].BulkData.Lock(LOCK_READ_WRITE);
	FMemory::Memcpy(TextureData, UncompressedBGRA.GetData(), UncompressedBGRA.Num());
	result->PlatformData->Mips[0].BulkData.Unlock();
 
	// Update the rendering resource from data.
	result->UpdateResource();
 
	return result;
 
}
 
//使用案例 注意是Full路径，绝对路径
UTexture2D* pTexture2D = UUtilsLibrary::LoadTexture(m_pRuntimeHotPoint->GetObjInfo()->GetBindMoviePath() + TEXT(".jpg"));
```

