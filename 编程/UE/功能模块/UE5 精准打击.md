## 特效挂载
```cpp
UParticleSystemComponent * psc =  UGameplayStatics::SpawnEmitterAttached(InteractParticle, GetMesh(), boneName
, Location, Rotator, FVector(1,1,1), EAttachLocation::KeepWorldPosition, true
, EPSCPoolMethod::AutoRelease);
```
特效会挂载在人物受击的具体点，且跟随人物移动，原理是生成一个 UParticleSystemComponent  组件并在运行期间挂载到 Actor 的 RootComponent 上，该组件会在特效生命周期结束后 很短的时间内自行进行垃圾回收（特殊情况是，如果这个特效是个持久的特效，那么组件将一直挂载在人物身上，除非手动结束特效）
## 逻辑原理

1. 建立动画通知类，在攻击动画里发出信号
```cpp
#pragma once
#include "CoreMinimal.h"
#include "Animation/AnimNotifies/AnimNotifyState.h"
#include "AnimNotify_AttackTracer.generated.h"

/**
* 动画通知头文件
*/
UCLASS()
    class PRECISIONATTACK_API UAnimNotify_AttackTracer : public UAnimNotifyState
    {
    GENERATED_BODY()
    
    public:
    virtual void NotifyTick(USkeletalMeshComponent MeshComp, UAnimSequenceBase Animation, float FrameDeltaTime) override;
    virtual void NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation) override;
    
    virtual  void NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation, float TotalDuration) override;
    
    TArray<FHitResult>				HitResult;
    TArray<AActor*>					HitActors;
    AController*					EventInstigator;
    TSubclassOf<UDamageType>		DamageTypeClass;
    FVector							LastLocation1;
    FVector							LastLocation2;
    class APrecisionAttackCharacter*Player;
    class USkeletalMeshComponent*	Weapon;
    TArray<AActor*>					ActorsToIgnore;
};
```

```cpp
#include "AnimNotify_AttackTracer.h"
#include "Components/SkeletalMeshComponent.h"
#include "Kismet/KismetSystemLibrary.h"
#include "Kismet/GameplayStatics.h"
#include "PrecisionAttackCharacter.h"
#include "WeaponBase.h"
#include "CharacterBase.h"
#include "GameFramework/Controller.h"

void UAnimNotify_AttackTracer::NotifyTick(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
                                          float FrameDeltaTime)
{
    Super::NotifyTick(MeshComp, Animation, FrameDeltaTime);
    
    if (Player)
    {
        // 插槽1
        // 盒体检测
        // UKismetSystemLibrary::BoxTraceMulti(Player->GetWorld(), LastLocation1, Weapon->GetSocketLocation("Trace1"), FVector(5, 30, 50)
        // 	, Weapon->GetComponentRotation(), ETraceTypeQuery::TraceTypeQuery4, false, ActorsToIgnore
        // 	, EDrawDebugTrace::ForDuration, HitResult, true);
        
        //射线检测
        UKismetSystemLibrary::LineTraceMulti(Player->GetWorld(), Weapon->GetSocketLocation("Trace2"), Weapon->GetSocketLocation("Trace1")
                                             , ETraceTypeQuery::TraceTypeQuery4, false, ActorsToIgnore, EDrawDebugTrace::ForDuration, HitResult, true);
        // 	, EDrawDebugTrace::ForDuration, HitResult, true)
        
        for (int i=0; i<HitResult.Num(); i++)
        {
            AActor* HitActor = HitResult[i].GetActor();
            if (HitActor && !HitActors.Contains(HitActor))
            {
                HitActors.Add(HitActor);
                UGameplayStatics::ApplyDamage(HitActor, 10.0f, EventInstigator, Player, DamageTypeClass);
                
                ACharacterBase* enemy = Cast<ACharacterBase>(HitActor);
                if (enemy)
                {
                    enemy->PlayHurtedParticle(HitResult[i].Location);
                    GEngine->AddOnScreenDebugMessage(FMath::RandRange(0, 100), 1, FColor::Blue, FString("Enemy"));
                }
                GEngine->AddOnScreenDebugMessage(FMath::RandRange(0, 100), 0.5, FColor::Green, HitActor->GetName());
            }
        }
        
        
        LastLocation1 = Weapon->GetSocketLocation("Trace1");
        LastLocation2 = Weapon->GetSocketLocation("Trace2");
    }
}

void UAnimNotify_AttackTracer::NotifyEnd(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation)
{
    Super::NotifyEnd(MeshComp, Animation);
    
    HitActors.Empty();
}

void UAnimNotify_AttackTracer::NotifyBegin(USkeletalMeshComponent* MeshComp, UAnimSequenceBase* Animation,
                                           float TotalDuration)
{
    Super::NotifyBegin(MeshComp, Animation, TotalDuration);
    
    Player = Cast<APrecisionAttackCharacter>(MeshComp->GetOwner());
    if (Player)
    {
        Weapon = Player->Weapon->GetWeaponMesh();
        ActorsToIgnore = { MeshComp->GetOwner() };
        LastLocation1 = Weapon->GetSocketLocation("Trace1");
        LastLocation2 = Weapon->GetSocketLocation("Trace2");
    }
}
```

## 武器类型区分
### 长剑类型武器
在武器首位两处添加骨骼点，以此两骨骼点做射线进行射线检测
![[编程/UE/功能模块/Image/UE5 精准打击/2dfeba0da94c0bd0e26f3bee03af1ea9_MD5.png]]
![[编程/UE/功能模块/Image/UE5 精准打击/e068b2167ae8f6e0139bc3713f406b21_MD5.png]]
### 枪械类型武器

