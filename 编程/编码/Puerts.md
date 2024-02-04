<a name="y7seW"></a>
# 函数回调
<a name="WgFHW"></a>
## 声明
![image.png](media/image-18.png)<br />![image.png](media/image-19.png)

<a name="ezhHz"></a>
## 绑定
场景：在类A将函数接口传递给类B（会存在this指向变更的问题，所以在赋值时需要绑定this指针）<br />![image.png](media/image-17.png)

<a name="ERIvc"></a>
# 声明合并
[声明合并 · TypeScript中文网 · TypeScript——JavaScript的超集](https://www.tslang.cn/docs/handbook/declaration-merging.html)
```javascript
描述：LLVehicle 是一个c++生成的TS类
declare module "ue" {
  interface LLVehicle {
    CalcNearstWayRoad(Driver: $Nullable<NCharacterBase>): UE.TArray<UE.Vector>;
                      OnGetInWalkFinished(Driver: $Nullable<NCharacterBase>): void;
  }
}

UE.LLVehicle.prototype.CalcNearstWayRoad = function CalcNearstWayRoad(Driver: $Nullable<NCharacterBase>): UE.TArray<UE.Vector> {
  return UE.NewArray(UE.Vector);
};
UE.LLVehicle.prototype.OnGetInWalkFinished = function OnGetInWalkFinished(Driver: $Nullable<NCharacterBase>): void {};
```

