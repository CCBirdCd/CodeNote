<a name="yiYQQ"></a>
# 问题：
纹理流送带来的贴图延迟加载
<a name="pRE2S"></a>
# 可参考的解决方案
<a name="rPohm"></a>
## 禁用全局纹理的流送
不推荐
<a name="i0qZY"></a>
## 一些可能有用的设置
r.Streaming.FramesForFullUpdate 0只是减少了io（从磁盘到cpu）分摊的时间，所以在移动设备之类IO不慢的平台上，0和5的差别可能并不是特别大，更大的时间在cpu->gpu上传贴图的时候。<br />r.Streaming.AllowFastForceResident 1本身其实是让设置成Force Resident的streamable asset（这里是贴图）的所有mip都streamin，而无视Streaming manager的wanted mip计算，如果大量的资产都这么设置，很可能导致实际streaming的资产需求比不开更高，并且也影响了实际的streaming manager对于需求mip的计算，所以没有明显受益。
<a name="sVrP7"></a>
## 禁用指定纹理的流送
优点：能立刻加载出最高质量的贴图<br />缺点：内存性能消耗消耗较大，因为引擎无法根据距离等条件管理贴图的实际加载大小<br />![image.png](https://cdn.nlark.com/yuque/0/2024/png/26747865/1705635821621-e815d4d6-2159-4af4-9b96-6a514458209a.png#averageHue=%23514c46&clientId=ua473c496-2220-4&from=paste&height=962&id=u8ec9061a&originHeight=962&originWidth=1838&originalType=binary&ratio=1&rotation=0&showTitle=false&size=1031278&status=done&style=none&taskId=u69f473d4-d42a-416a-81df-15bf8d9f888&title=&width=1838)
<a name="sDDZV"></a>
## 提前加载纹理
![image.png](https://cdn.nlark.com/yuque/0/2024/png/26747865/1705635687082-1de770e1-abdc-44d8-b900-9ac30c8f416a.png#averageHue=%23fbfaf8&clientId=ud537e036-4e0b-4&from=paste&height=843&id=u35d8e9ba&originHeight=843&originWidth=1066&originalType=binary&ratio=1&rotation=0&showTitle=false&size=126254&status=done&style=none&taskId=uf84d9b2c-8175-4bfc-bbfd-0a4ad4cb5f2&title=&width=1066)
