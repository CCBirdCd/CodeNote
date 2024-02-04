# 问题：
纹理流送带来的贴图延迟加载
# 可参考的解决方案
## 禁用全局纹理的流送
不推荐
## 一些可能有用的设置
r.Streaming.FramesForFullUpdate 0只是减少了io（从磁盘到cpu）分摊的时间，所以在移动设备之类IO不慢的平台上，0和5的差别可能并不是特别大，更大的时间在cpu->gpu上传贴图的时候。
r.Streaming.AllowFastForceResident 1本身其实是让设置成Force Resident的streamable asset（这里是贴图）的所有mip都streamin，而无视Streaming manager的wanted mip计算，如果大量的资产都这么设置，很可能导致实际streaming的资产需求比不开更高，并且也影响了实际的streaming manager对于需求mip的计算，所以没有明显受益。
## 禁用指定纹理的流送
优点：能立刻加载出最高质量的贴图
缺点：内存性能消耗消耗较大，因为引擎无法根据距离等条件管理贴图的实际加载大小
![[编程/UE/功能模块/Image/TextureStreaming/ad2f28a108d4aa916ab8ebf4274b7406_MD5.png]]
## 提前加载纹理
![[编程/UE/功能模块/Image/TextureStreaming/d7a3a7b8ed6c791cd2c7dd2640db7a3b_MD5.png]]
