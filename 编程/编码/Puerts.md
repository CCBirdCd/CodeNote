<a name="y7seW"></a>
# 函数回调
<a name="WgFHW"></a>
## 声明
![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1666075186160-5acbc784-c090-48e0-a9aa-8168e03eb843.png#averageHue=%231e1e1e&clientId=ua2143b2a-3f23-4&from=paste&height=95&id=ua9e20107&originHeight=95&originWidth=652&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7315&status=done&style=none&taskId=ud74dd959-91a2-4437-8787-64a8e03ad16&title=&width=652)<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1666075161461-10c28ee5-808b-4bc5-ae93-f54d20d305b9.png#averageHue=%231f1e1e&clientId=ua2143b2a-3f23-4&from=paste&height=106&id=u5083f79e&originHeight=106&originWidth=658&originalType=binary&ratio=1&rotation=0&showTitle=false&size=10160&status=done&style=none&taskId=u0d28445d-0889-4cac-b768-9281f87486c&title=&width=658)

<a name="ezhHz"></a>
## 绑定
场景：在类A将函数接口传递给类B（会存在this指向变更的问题，所以在赋值时需要绑定this指针）<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/26747865/1666075068506-2fb9a19f-b06c-45af-9061-d2c1cc770d13.png#averageHue=%23211f1e&clientId=ua2143b2a-3f23-4&from=paste&height=135&id=u19581f31&originHeight=135&originWidth=687&originalType=binary&ratio=1&rotation=0&showTitle=false&size=14492&status=done&style=none&taskId=u13427d05-0b10-4c8f-986f-95985f5c18b&title=&width=687)

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

