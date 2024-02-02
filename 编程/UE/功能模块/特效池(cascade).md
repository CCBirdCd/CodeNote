<a name="hnYDO"></a>
## 一、特效池是干嘛的
举个粒子解释一下池子：<br />比如你是弓箭手，你会射箭，你会从地上（内存）捡树枝打造弓箭（NewObject）。
<a name="mX5U3"></a>
### - 如果没有箭袋
那每次想射箭，都需要从地上捡树枝打造弓箭，这个过程想想就很麻烦，所以你的效率很低
<a name="xFhoq"></a>
### - 如果背上背了箭袋
那么你可以在从地上捡树枝打造弓箭，并射出去之后，把弓箭捡回来，插到箭袋里，下次想射箭，如果箭袋里有弓箭，那直接拿出来射就行，不需要重新打造<br />以上是个人理解，有问题可以讨论~<br />值得注意的是：

1. 能这样做的基础是，每次射出去的弓箭，最后都会捡回来，除非箭袋都没了（即弓箭的生命应该完全由箭袋管理），不捡回来的箭袋就没有意义了，还得背着。。
2. 箭袋是有大小的，放了 100 支箭之后，就放不下第 101 支了（至于为什么会有第 101 支箭，是因为不是每次射箭之后都有时间把那支箭拿回来，可能一次要放五支箭出去，然后又射了三支，过一会再把这八支一起拿回来，所以在这个过程中，地上的箭和箭袋中的箭加起来，可能超过了箭袋的容量，那么，捡的时候捡满了就不捡了）
3. 从箭袋中拿箭，把箭放回箭袋的操作都不麻烦，至少一定要比从地上拣树枝子打造弓箭要容易，不然每次弄新的不就好了。
<a name="YZhNw"></a>
## 1.1 使用特效池的目的
ParticleSystem（俗称粒子特效）的释放最终调用的都是 CreateParticleSystem 函数，如下所示：
```cpp
UParticleSystemComponent* CreateParticleSystem(
    UParticleSystem* EmitterTemplate, 
    UWorld* World, AActor* Actor, 
    bool bAutoDestroy, 
    EPSCPoolMethod PoolingMethod)
{
    //Defaulting to creating systems from a pool. Can be disabled via fx.ParticleSystemPool.Enable 0
    UParticleSystemComponent* PSC = nullptr;
    if (FApp::CanEverRender() && World && !World->IsNetMode(NM_DedicatedServer))
    {
        if (PoolingMethod != EPSCPoolMethod::None)
        {
            //If system is set to auto destroy the we should be safe to automatically allocate from a the world pool.
            PSC = World->GetPSCPool().CreateWorldParticleSystem(EmitterTemplate, World, PoolingMethod);
        }
        else
        {
            PSC = NewObject<UParticleSystemComponent>((Actor ? Actor : (UObject*)World));
            /// PSC->xxx = xx 等一些初始化操作 blablabla...
        }
    }
    return PSC;
}
```
核心逻辑就是，如果 PoolMethod 不是 None，则从池子中取；如果是 None，则会 NewObject，即每次释放一个特效，都会创建新的。<br />NiagaraSystem（俗称奶瓜特效），也是一样，接口是 CreateNiagaraSystem，也是每次 NewObject。<br />**_Note：_** 值得注意的是，两种特效都进行了 World && !World->IsNetMode(NM_DedicatedServer) 的判断，即特效在服务器上是都不会创建的，所以不需要关心服务器上的特效会 NewObject，但是如果是 Actor 上挂载的ParticleSystemComponent（不管是从代码，还是从蓝图资源），是会在服务器创建 Component 的，这点需要注意。<br />由于 NewObject 会进行一系列操作，所以对 CPU（GameThread）肯定是有消耗的（虽然一个可能不多，但是架不住数量多啊），所以如果能够利用池子进行缓存，每次不新创建，而是从池子里取，则能对 CPU 性能有所帮助（利用 **空间换时间**）。
<a name="t6MMX"></a>
### 1.2 UE 中的特效池
由于 ParticleSystem 和 NiagaraSystem 是两种完全不同的特效，所以在对这两种特效的支持（释放，池化等）都是两套完全独立的代码，但是逻辑大体相似。<br />ParticleSystem 的池子叫 FWorldPSCPool，NiagaraSystem 的池子叫 UNiagaraComponentPool，下边会详细总结。
<a name="IRZzP"></a>
## 二、特效池使用
这里只记录 ParticleSystem 的特效池使用方法。<br />首先需要知道的是，每一个特效资源，在代码中就是一个 UParticleSystem*，每一个实际出现效果的（不管是火焰，爆炸，闪光，烟雾，粒子等等），都是用 UParticleSystemComponent 来实现的，即 UParticleSystem 是 **_数据_**，UParticleSystemComponent 是 **_实体_** 的感觉（Niagara 完全同理）。<br />比如一刀看下去，三个人身上冒血，那么是三个 UParticleSystemComponent 同时在播放，但是使用的是同一个 UParticleSystem 作为 **_数据源_**，理解了这一点，下边就都好说了。<br />特效池的目的就是同一个数据源，创建出的多个实体进行缓存，从而达到虽然一共播放了 1000 次冒血特效，但是只 New 了 3 个 Object 的效果。
<a name="XJbE4"></a>
## 2.1 关键数据结构
详见 Engine\Source\Runtime\Engine\Classes\Particles\WorldPSCPool.h。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1654483157906-0a150873-f5fc-428c-93eb-28045b102c46.png#averageHue=%23201f1f&clientId=u7ed93926-fb70-4&from=paste&id=ub5383dfe&originHeight=673&originWidth=617&originalType=url&ratio=1&rotation=0&showTitle=false&size=280310&status=done&style=none&taskId=u0390f32d-b4c6-4909-a609-15a48416cef&title=)<br />WorldPSCPool.h 总览
<a name="yK8i4"></a>
### **1. EPSCPoolMethod**
引擎一共 提供了三种池化操作（其实 EPSCPoolMethod 枚举类型由五个值，但是只关注前三个即可，后两种是内部逻辑使用）：

1. **None**

即不放入池子，每一次都是重新 Create 新的<br />2. **AutoRelease**<br />自动分配入池子，并且自动回收回池子中。适用于一次性效果的特效（one-shot fx），不需要考虑存起来（reference），只需要放就完事了。但是由于会自动回收，所以如果想修改这个 PSC 的属性，可能会不安全（所以默认肯定不能给这个值，因为不知道用户会不会接收释放特效的接口的返回值，并进行什么操作）。<br />3. **ManualRelease**<br />需要自己手动调用 ReleaseToPool() 才能回收（AutoDestroy 选项失效），适用于需要自己控制时长的 **"永久"** 特效（因为这种特效必须手动回收，否则会内存泄漏，所以肯定也不能是默认值）。<br />综上，引擎默认是不把特效放入池子的，但是如果想用，只需要把 SpawnEmitter 的接口的默认参数改为需要的即可（建议是一次性效果，如一次爆炸特效，用 AutoRelease；持续性类似 Buff 的效果，如身上的灼烧火焰效果，用 ManualRelease，并在火焰时间到，灼烧效果消失的时候，手动 ReleaseToPool()）。<br />特效的池子很简单，每一个粒子特效（UParticleSystem*，即特效资源），对应一个数组（ TArray<FPSCPoolElem> FreeElements），这个结构也很简单，如下：
```cpp
USTRUCT()
struct FPSCPoolElem
{
    GENERATED_BODY()

    UPROPERTY(transient)
    UParticleSystemComponent* PSC;

    float LastUsedTime;

    // 还有两个构造函数
};
```
<a name="GFiqZ"></a>
### 2. **FWorldPSCPool**
```cpp
USTRUCT()
struct ENGINE_API FWorldPSCPool
{
    GENERATED_BODY()

private:
    UPROPERTY()
    TMap<UParticleSystem*, FPSCPool> WorldParticleSystemPools;

    float LastParticleSytemPoolCleanTime;

    /** Cached world time last tick just to avoid us needing the world when reclaiming systems. */
    float CachedWorldTime;
public:

    FWorldPSCPool();
    ~FWorldPSCPool();

    void Cleanup();

    UParticleSystemComponent* CreateWorldParticleSystem(UParticleSystem* Template, UWorld* World, EPSCPoolMethod PoolingMethod);

    /** Called when an in-use particle component is finished and wishes to be returned to the pool. */
    void ReclaimWorldParticleSystem(UParticleSystemComponent* PSC);

    /** Call if you want to halt & reclaim all active particle systems and return them to their respective pools. */
    void ReclaimActiveParticleSystems();
	
    /** Dumps the current state of the pool to the log. */
    void Dump();
};
```
这个就是 UE 默认提供的粒子特效的池子，叫做 **FWorldPSCPool**（Niagara 的池子叫 **UNiagaraComponentPool**），但是如果使用的是 UGameplayStatics 里的释放粒子特效的接口（不管是 SpawnEmitterAtLocation 还是 SpawnEmitterAttached），都是默认不把特效放入池子的（即 EPSCPoolMethod::None，原因可能就是在于，引擎不知道应该给什么样的默认逻辑）。<br />**FWorldPSCPool **的生命周期可以认为由 World 管理，在 World 中有一个变量：
```cpp
UPROPERTY() 
FWorldPSCPool PSCPool; 
```
在 World 的 UWorld::CleanupWorldInternal 会调用 PSCPool.Cleanup() 清理特效池。<br />在 World 析构时会调用 FWorldPSCPool 的析构，会执行 Cleanup()。<br />在 CreateParticleSystem 的时候会调用 World->GetPSCPool().CreateWorldParticleSystem。<br />FWorldPSCPool 里最重要的就是：
```cpp
TMap<UParticleSystem*, FPSCPool> WorldParticleSystemPools;  
```
这个就是存储着所有释放过的特效数据源（UParticleSystem*），与对应的创建出来的实体（UParticleSystemComponent*）的数组的 Map。<br />至于为什么是数组，就是因为很可能有同时播放很多特效的需求，每一个都是一个**单独**的 Component，如前边说到的一刀三个冒血。
<a name="YnUif"></a>
### 3. FPSCPool
FWorldPSCPool::CreateWorldParticleSystem 中会从 WorldParticleSystemPools 中以特效为 Key 找出这个特效的小池子，并从中找出可用实体。
```cpp
FPSCPool& PSCPool = WorldParticleSystemPools.FindOrAdd(Template); 
PSC = PSCPool.Acquire(World, Template, PoolingMethod); 
```
**FPSCPool **就是针对每一个特效资源（UParticleSystem*）自己的小池子：
```cpp
USTRUCT()
struct FPSCPool
{
    GENERATED_BODY()

    //Collection of all currently allocated, free items ready to be grabbed for use.
    //TODO: Change this to a FIFO queue to get better usage. May need to make this whole class behave similar to TCircularQueue.
    UPROPERTY(transient)
    TArray<FPSCPoolElem> FreeElements;

    //Array of currently in flight components that will auto release.
    UPROPERTY(transient)
    TArray<UParticleSystemComponent*> InUseComponents_Auto;

    //Array of currently in flight components that need manual release.
    UPROPERTY(transient)
    TArray<UParticleSystemComponent*> InUseComponents_Manual;
	
    /** Keeping track of max in flight systems to help inform any future pre-population we do. */
    int32 MaxUsed;

public:
    FPSCPool();
    void Cleanup();

    /** Gets a PSC from the pool ready for use. */
    UParticleSystemComponent* Acquire(UWorld* World, UParticleSystem* Template, EPSCPoolMethod PoolingMethod);
    
    /** Returns a PSC to the pool. */
    void Reclaim(UParticleSystemComponent* PSC, const float CurrentTimeSeconds);

    /** Kills any components that have not been used since the passed KillTime. */
    void KillUnusedComponents(float KillTime, UParticleSystem* Template);

    int32 NumComponents() { return FreeElements.Num(); }
};
```
关键成员变量、函数：

- **FreeElements **- 就是存储着这个特效可用的实体 Component，正在使用的不在这里，还没有被回收到池子里。
- **InUseComponents_Auto**、**InUseComponents_Manual **- 可以不管，可以认为是用来 Debug 的（ENABLE_PSC_POOL_DEBUGGING）
- **MaxUsed **- 最多用了多少个
- **Acquire()** - 用于从自己的数组中取出可用 Component 的方法
- **Reclaim()** - 放回池子的方法
<a name="XIL8u"></a>
### 4. FPSCPoolElem
TArray<FPSCPoolElem> FreeElements; 数组里边存储的即这个数据源创建出来的每一个实体。<br />其中数组的大小是有限制的，就是特效资源上配置的 MaxPoolSize（代码详见 FPSCPool::Reclaim，如果 FreeElements.Num() < (int32)PSC->Template->MaxPoolSize，则不会回收这个 Component，而是直接 DestroyComponent）。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1654483157951-6cb905aa-339c-475b-a694-722b9b890391.png#averageHue=%23827a4f&clientId=u7ed93926-fb70-4&from=paste&id=uf0a64df2&originHeight=430&originWidth=587&originalType=url&ratio=1&rotation=0&showTitle=false&size=160608&status=done&style=none&taskId=u171ed9ce-1ff8-46c1-9001-162f6d25550&title=)<br />特效上的池子最大数量配置
```cpp
USTRUCT()
struct FPSCPoolElem
{
    GENERATED_BODY()

    UPROPERTY(transient)
    UParticleSystemComponent* PSC;

    float LastUsedTime;

    // 两个构造函数
};
```
**FPSCPoolElem **里边就只有实体（PSC）和这个实体上一次使用的时间（LastUsedTime），用于超时剔除等（详见FPSCPool::KillUnusedComponents）。
<a name="Mc6DB"></a>
## 2.2 关键流程
<a name="NK245"></a>
### 2.2.1 播放特效 / 从池子中取
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1654483157831-a93ed5d6-b1ba-4b63-936e-121d595e12c0.png#averageHue=%23f5f5f5&clientId=u7ed93926-fb70-4&from=paste&id=u60071314&originHeight=626&originWidth=647&originalType=url&ratio=1&rotation=0&showTitle=false&size=119238&status=done&style=none&taskId=ud0f9c496-c25a-4710-a9b2-a3c5d389081&title=)<br />特效 Component 获取大体流程图
<a name="CXYX3"></a>
### 2.2.2 结束特效 / 放回池子中
放回池子的流程稍微麻烦些，因为还有定时清理功能。
```cpp
void FWorldPSCPool::ReclaimWorldParticleSystem(UParticleSystemComponent* PSC)
{
    // Check blablabla
    if (GbEnableParticleSystemPooling)
    {
    float CurrentTime = PSC->GetWorld()->GetTimeSeconds();

    //Periodically clear up the pools.
    if (CurrentTime - LastParticleSytemPoolCleanTime > GParticleSystemPoolingCleanTime)
    {
        LastParticleSytemPoolCleanTime = CurrentTime;
        for (TPair<UParticleSystem*, FPSCPool>& Pair : WorldParticleSystemPools)
        {
            Pair.Value.KillUnusedComponents(CurrentTime - GParticleSystemPoolKillUnusedTime, PSC->Template);
        }
    }

    // Check blablabla
    PSCPool->Reclaim(PSC, CurrentTime);
    }
    else
    {
        PSC->DestroyComponent();
    }
}
```
每次回收一个 Component 的时候，都会判断现在距上次清理的时间隔了多久，如果超过了 GParticleSystemPoolingCleanTime（默认值是 30.f，即 30 秒），则会对 WorldParticleSystemPools 中 **_所有元素_** 进行清理（并不只清理当前这个特效的缓存），除此之外，和 2.2.1 中的流程类似：<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1654483157923-6b405577-cebc-425a-8ef4-13b29daee593.png#averageHue=%23f7f7f7&clientId=u7ed93926-fb70-4&from=paste&id=u48ee4897&originHeight=527&originWidth=720&originalType=url&ratio=1&rotation=0&showTitle=false&size=97542&status=done&style=none&taskId=u2f7d40af-581f-4d2f-b87f-7cea06e5fc7&title=)<br />特效 Component 回收大提流程图
<a name="hyhzH"></a>
## 2.3 特效池数据查看
在编辑器中的命令行窗口，输入 fx.DumpPSCPoolInfo，可以在 Output 窗口中看到当前池子的大小，以及每个 PS 有多少个 Free 的，有多少个正在 Used 的。<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1654483158085-5f939bc8-ad54-49d6-ba40-d239f5d11397.png#averageHue=%23323232&clientId=u7ed93926-fb70-4&from=paste&id=u113f9095&originHeight=180&originWidth=720&originalType=url&ratio=1&rotation=0&showTitle=false&size=132217&status=done&style=none&taskId=ub469787a-a8d0-46b2-870d-50e5ccfd6de&title=)<br />上边这张图可以看到，当前池子一共占内存 0.7 MB，每一个特效资源（ParticleSystem），会输出对应的数据：

- **Free **- 当前池子中可用的 Component 实体
- **Used **- 当前正在使用的 Component 实体（Auto、Manual 对应释放时设置的池化方法）
- **MaxUsed**- 最多同一时刻一起使用的 Component 实体数量
- **System **- 特效资源的路径

吐槽一下，第一行的输出，不应该换一行吗。。。。。。看起来好难受。。<br />以及并不能看到减少了多少次 NewObject
<a name="K0MvH"></a>
## 三、特效池需要注意的问题
<a name="bWMAn"></a>
### 3.1 生命周期管理
FPSCPool::Acquire 中会进行 RetElem.PSC->Rename(nullptr, World, REN_ForceNoResetLoaders);，即把这个 Component 的 OwnerPrivate（GwtOwner()）设为世界，官方的注释是：<br />Rename the PSC to move it into the current PersistentLevel - it may have been spawned in one level but is now needed in another level.<br />也就是为了防止在一个 Level 中创建了这个特效 Component，又想在另一个 Level 中使用，所以全部存在 World 上，毕竟 PSCPool 就存在 World 上。<br />然而因为特效的顿帧（即 Tick 的 DeltaTime 是跟 OwnerPrivate 相关的，详见 FActorComponentTickFunction::ExecuteTickHelper），所以如果希望特效的速率和角色 A 一致，那么需要 PSC->SpawnedParticle->Rename(nullptr, Actor); 将它的 OwnerPrivate 设为角色 A。<br />这样会导致当角色 A Destroy 的时候，会将身上的 Component 也销毁，会触发 FPSCPool::Acquire 中的 check：
```cpp
check(!RetElem.PSC->IsPendingKill()); 
```
可行的解决方案就是：

- 如果是 ManualRelease 方式，那么可以直接 Rename，只不过需要在 ReleaseToPool 之前在 Rename 回当前 World
- 如果是 AutoRelease 方式，那就需要用别的方式修改顿帧了

总结就是如果需要使用池子，那就不要对返回值 PSC 做任何操作了。<br />再吐槽一下，修改 Owner 竟然没有 SetOwner 这种函数，而是用 Rename。。感觉很奇怪，可能就是不想让你可以 SetOwner 吧
<a name="pgW2Z"></a>
### 3.2 Auto/Manual 一定要对应
如果释放的时候选的 PoolMethod 是 AutoRelease，但是又手动调用了 ReleaseToPool() ，那么会有 Warning：

<a name="FSvNI"></a>
### 3.3 特效上存了数据
又遇到拖尾特效，或者存了一些数据的特效在 ReleaseToPool 之后下次再次释放时候表现不正确的问题，还没有找到原因，先挖个坑，之后填。
<a name="z2nDy"></a>
### 3.4 可能导致没有成功回收的原因
如果在 ReleaseToPool 之前调用了 ResetParticles(true)（参数 bEmptyInstances 默认是 false），则无法回收，因为走不到 Complete 了。
