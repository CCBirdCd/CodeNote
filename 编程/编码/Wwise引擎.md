<a name="TUJXn"></a>
# Wwise
<a name="xE45e"></a>
## [使用wwise音效引擎的好处](https://www.cnblogs.com/VariousCloudShadow/p/7842914.html)
用过一段时间的wwise，做以下几个具体功能的时候比较方便：<br />1.当策划需求一个声音需要随机播放多个随机音源的其中一个时，例如脚步声、普通攻击声，当这类声音一直播放的都是同一个音源的时候，人会产生听觉疲劳。如果使用wwise，客户端只需要触发事件就可以了，至于一个事件怎样去随机一个音效，就是策划管的事了。<br />2.根据材质播放不同的声音，例如挖掘土地和挖掘金属、挖掘石块要产生不同的音效，这个功能可以让策划配置不同材料的switch名，客户端根据导表信息，自动设置switch，就会播放对应的音效。<br />3.声音曲线RTPC（real time parameter control），x轴可以是距离、时间等，y轴可以是音量、音调等，曲线是可以拖动调整的，这就产生了一个可以可视化调节的对应关系，而这种对应关系就不需要程序去管了。<br />4.对全部声音的状态处理，例如在陆地上和水下，听到的声音效果应该是不同的，这个在wwise里可以进行比较简单的设置。<br />5.对音源文件的内存优化，例如一段较长的背景音乐，可以设置只加载正在播放的那一小片段。<br />6.交互式音乐，类似dota那种，当战斗的时候播放战斗音乐，战斗结束切换为一般BGM，这是通过触发器trigger实现的，客户端只需要判断切换战斗音乐的时机并触发，wwise会自动根据策划配置平滑过渡。<br />7.音效播放的各种效果，例如淡入淡出，播放之后停顿几秒继续播放，声音播放几秒之后慢慢变小等，都可以交由策划配置，客户端不用多写代码支持这些功能。<br />8.多平台支持，多语言支持。wwise支持多个平台，会根据平台不同输出不同的bank文件（一类音源的打包小集合），并做相关优化。当游戏的语音有多个国家语言版本的时候，wwise也能轻松设置。<br />9.性能监视。<br />10.动态混响，例如吃鸡这样的游戏，在房间里打狙击枪和在平地上打狙击枪，由于房间内声音会来回反射，所以混响效果应该是不同的。通过游戏过程中，对RTPC实时变量的控制，可以达到此效果
<a name="bYVXB"></a>
### 一、Wwise初始化（考虑热更）需要以下几个步骤：
一、加载AkTerminator、AkInitializer两个脚本<br />这里有两种方法，可以根据需求选择一种。<br />1、 动态加载这两个脚本，之后通过修改AkInitializer里的属性来改变设置。优点：简单方便。缺点：因为在加载AkInitializer时就会完成Wwise初始化。这里只有少部分参数设置（例如Log开关）会生效，另外的大多数参数设置(比如SoundEngine内存分配，默认语言)不会生效。<br />2、 将这两个脚本制作成Prefab，音频初始化时实例化这个prefab。优点：所有初始化参数都可以配置并热更。缺点：需要在初始化Wwise时进行prefab的实例化操作。<br />大多数情况下，初始化参数不需要通过热更的方式修改。但是在Wwise2017.1.3版本Wwise曾出现过一个重大bug，初始化参数里内存如果设置过小，游戏就会大概率出现闪退。当时有个项目我们恰好选择了第一种方式，最后不得不整包更新才解决了这个问题。所以，推荐使用第二种方式加载这两个脚本。
<a name="CHiBr"></a>
### 二、注册全局声音对象，用于所有2D声音的播放。
我们通常会自己管理一个对象播放所有2D声音（音乐、环境、UI）。这个对象不需要同步坐标，也省去了频繁注册注销SoundObject的损耗。这个步骤能够节省大量的CPU资源。
<a name="OoLZ1"></a>
### 三、加载需要在游戏开始就要用到的SoundBank（比如音量设置、UI、音乐等）
Wwise使用string作为LoadBank的参数，有三种方式可以实现这个需求：<br />1、 在lua中保存一个字段记录要加载的SoundBank Name，调用C#接口加载。<br />2、 做一个.asset配置文件，C#从配置文件中读取要加载的SoundBank Name。<br />3、 将加载SoundBank操作做成prefab，在初始化时实例化prefab。<br />初始化时要加载的SoundBank通常需要设计师修改，第三种方式对设计师最友好，但是也需要prefab的实例化操作。
<a name="sW8du"></a>
### 四、音量的初始化设置
关于音乐、音效、语音的开关及音量的设置应该保存在客户端本地，并在Wwise播出第一个声音前设置正确的参数。设计师一般会将Mute、Unmute（开关）操作封装成Event，BusVolume（音量）设置封装成RTPC，然后打包为SoundBank交给程序调用。所以这一步骤依赖LoadBank的完成。一般情况下项目使用的LoadBank方法是异步的。所以这里需要把跟这个功能有关的SoundBank单独拆出来，使用同步的方式加载，或者通过回调确定LoadBank已完成。<br />以上所有操作应该在C#内被封装成一个方法，让lua在合适的时机去调用。
<a name="KkUil"></a>
### 五、关于热更
Wwise提供了两个API用于设置SoundBank路径。分别是SetBasePath和AddBasePath。SetBasePath设置基础目录，AddBasePath设置后续DLC目录。Wwise允许设置多个DLC目录，LoadBank时会从最后一次AddBasePath的路径依据SoundBank的文件名开始搜索，依次向前最后到SetBasePath的路径，搜索到第一个目标SoundBank后加载。<br />（注意，AddBasePath不是全平台都有效。在PC下是无效的，所以不能在PC平台测试。）<br />在iOS和Android平台：<br />SetBasePath的默认路径为：<br />Application.streamingAssetsPath/Audio/GeneratedSoundBanks/(Platform)<br />AddBasePath的默认路径为：<br />Application.persistentDataPath<br />一般我们会把这里改为：<br />Application.persistentDataPath/Audio/GeneratedSoundBanks/(Platform)<br />在发布热更包时，将音频资源放到对应位置即可。
<a name="k2jHl"></a>
### 六、关于初始化时机：
Wwise有两种SoundBank，Init bank和资源bank。默认情况下Init bank不需要手动加载，在AkInitializer.Initialze();会加载这个bank。Init bank中保存了关于总线、游戏同步器（游戏参数，切换开关和状态）和环境效果的设置。<br />考虑到Init Bank也需要热更。初始化有以下几种选择：<br />1、 在完成资源热更后再进行初始化。这样最安全，但是在热更完成前，一直会处于无声的状态。（市面上大多数产品最终都选择了这个方案。）<br />2、 在游戏运行时进行初始化，同时载入一个登陆音乐的SoundBank，播放音乐。热更完成后，卸载Init Bank和登陆音乐的SoundBank并重新加载。这里要避免使用异步加载卸载的接口。Wwise官方不推荐这样做。<br />3、 在游戏运行时进行初始化，同时载入一个登陆音乐的SoundBank，播放音乐。热更完成后不重新加载。游戏下次运行时会使用全新的SoundBank，本次会使用旧的Init bank及登陆音乐SoundBank。Wwise官方同样不推荐这样做。（我们的产品大多数使用了这种方案。）<br />这里需要注意，非Stream的SoundBank文件在加载后被播放时，源文件不会被占用，可以通过热更替换。而Stream文件播放时，源文件会被占用，不能热更。所以第2、3种方法里的登陆音乐如果有热更需求，就不能使用Stream。
<a name="SHVE9"></a>
### 七、IOS Wwise库需要剥离符号
程序员在第一次接触Wwise的时候，都会发现Wwise的iOS版本库文件特别大。即使是Release版本也有300多M。<br />官方文档在这个地方专门做过解释：<br />[Build your Unity Game for a Target Platform](https://link.zhihu.com/?target=https%3A//www.audiokinetic.com/library/edge/%3Fsource%3DUnity%26id%3Dpg__howtobuilddeployios.html)<br />也可以参考这个问题的回答：<br />[Why is the Wwise iOS Unity Integration so big?](https://link.zhihu.com/?target=https%3A//www.audiokinetic.com/qa/622/why-is-the-wwise-ios-unity-integration-so-big)<br />简单的说，是因为Unity不支持 Thumb指令，所以即使Release版本也包含了所有的调试符号。 这个只能在XCode里面去剥离，最终的大小是 1-2M。
<a name="QDvWO"></a>
## 功能点
<a name="a9nyN"></a>
### 单个和发声位和多个发声位
![image.png](media/image-14.png)<br />![image.png](media/image-7.png)<br />![image.png](media/image-12.png)
<a name="AARCK"></a>
### 声音切换组（Switch Group）
![image.png](media/image-4.png)
<a name="AKZB4"></a>
### 可以直接在动画序列添加 AnimNotify_AKEvent
通过配置 Notify 的 Event，可以通过发送 Event 来触发某一固定声音，缺点是没办法在播放声音之间切换 SwitchGroup(即 无法满足多态下的声音切换，需要自己定义 AnimNotify 实现切换和 postEvent 事件)
<a name="uA3f8"></a>
### 使用 Event-Based Packaging 的工作流程
<a name="ZrSgJ"></a>
#### 1、简介
Unreal Integration 2019.2 中针对素材管理实施了一些重大改进，包括新增了 Event-Based Packaging 工作流程。现在无需手动将 Wwise Event 导入到 Content Browser 中，也不必通过手动创建的 SoundBank Unreal 素材来管理引用。<br />这一新的工作流程会自动创建工程所需的全部素材：Event、Auxiliary Bus、Media、Init Bank、RTPC (Game Parameter)、Switch Value、State Value、Trigger 和 Acoustic Texture。在构建 Sound Data 时，所有 Event 和 Auxiliary Bus 素材都会包含不带媒体的 SoundBank。媒体素材现在被封装到了专门的 Unreal 素材中，而每个 Event 都对应有一系列所需媒体。不再需要手动加载 SoundBank；若地图中引用了 Event 或 Aux Bus，则会将必要的 SoundBank 数据和相关媒体自动加载到内存中。<br />EBP的设计把资源的加载/卸载控制权完全交给了UE。对于音效师来说，工作流中可以忽略对于资源加载问题的考虑，把重心转移到Event的结构与逻辑设计上。对于程序来说，因为音效资源也从bnk被包装成了uasset，资源的加载/卸载管理也统一到了UE的资源管理流程中，维护音效资源和维护其他资源没有了任何差别，可以使用相同的维护逻辑

<a name="iIxMK"></a>
#### 2、在Unreal工程中集成Wwise
创建unreal工程，在Wwise Launcher中江Wwise工程集成到Unreal工程中，集成成功后，可以在Unreal工程的ProjectSettings界面看到新增的Wwise模块<br />![image.png](media/image-3.png)


启用 Use Event—Based Packaging，这里的Wwise Sound Data Folder是Unreal用于存放Wwise相关资源的文件夹

![image.png](media/image-9.png)

启用Enable Automatic Asset Synchronization，可以启用Auto Connect to WAAPI。这里的Auto Connect to WAAPI若启用了，则每次在 Wwise 中创建对象时都会创建对应的 Unreal 素材。同样地，在重命名、删除、移动 Wwise 对象或重新设置父对象时，会立刻反映到 Unreal Content Browser 中。在禁用 WAAPI 的情况下，将只在保存 Wwise 工程时执行此同步操作。

![image.png](media/image-11.png)

当上述选项设置完毕后，在Wwise中创建的Event会自动同步到Unreal的Wwise Sound Data Folder中

![image.png](media/image.png)	![image.png](media/image-13.png)

<a name="wKm8H"></a>
#### 3、在关卡中引用AkAudioEvent对象
在Unreal中创建与Wwise中同名的SoundBank，在Unreal中绑定Event和SoundBank

![image.png](media/image-2.png)![image.png](media/image-1.png)

![image.png](media/image-10.png)<br />![image.png](media/image-5.png)<br />在Wwise中Generate SoundBank

![image.png](media/image-8.png)

在unreal中build-Generate Sound Data，如果在wwise中为Event绑过Audio，现在在unreal中play event可以听到对应的Audio声音。

<a name="GqEHA"></a>
#### 4、测试
在关卡蓝图中进行测试

![image.png](media/image-6.png)

能正常听到声音<br />[<br />](https://blog.csdn.net/qq_41936799/article/details/119777000)
