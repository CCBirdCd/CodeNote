# VisibleAnywhere
- 指示此属性在所有属性窗口中都可见，但不能编辑。此说明符与“编辑”说明符不兼容。
``` cpp
UPROPERTY(VisibleAnywhere)
int32 VisibleAnywhereNumber;
```
[[编程/UE/Image/UPROPERTY 介绍/608d5c67ca1aaf85e367fbc3f4767599_MD5.jpeg|Open: image-20240207142724364.png]]
![[编程/UE/Image/UPROPERTY 介绍/608d5c67ca1aaf85e367fbc3f4767599_MD5.jpeg]]

---
# VisibleDefaultsOnly

``` cpp
UPROPERTY(VisibleAnywhere)
int32 VisibleAnywhereNumber;
```

---
# VisibleInstanceOnly
- 此属性仅在蓝图实例中可见，如地图中存在的实例。这是一个很少使用的可见性说明符，但是您可以使用它向Blueprint用户显示在C++中定义的或基于其他属性计算的属性值。
- 蓝图中无法看见只能在实例化的对象的细节面板内看见
 [[编程/UE/Image/UPROPERTY 介绍/9fd6fa8c3d1acacba04c9dd158d9f1b6_MD5.jpeg|Open: image-20240207143410599.png]]
![[编程/UE/Image/UPROPERTY 介绍/9fd6fa8c3d1acacba04c9dd158d9f1b6_MD5.jpeg]]

---
# EditAnywhere
- 可以在蓝图资产的详细信息面板和蓝图实例的详细信息面板中更改属性。请注意，这是指在**细节**面板，在蓝图图中不可见。为此你需要使用`BlueprintReadWrite`.
- 只有使用 类似 BlueprintReadWrite 的标识修饰过的属性才能通过蓝图节点访问

---
# EditInstanceOnly
- 只能在实例化的对象上编辑，如下图蓝图中无法找到这个变量，但是将蓝图拖到场景中之后却能找到
- 应用场景，不需要默认值

[[编程/UE/Image/UPROPERTY 介绍/b1754bdfdc153c329f528b9d8baadbd4_MD5.jpeg|Open: image-20240207145445241.png]]
![[编程/UE/Image/UPROPERTY 介绍/b1754bdfdc153c329f528b9d8baadbd4_MD5.jpeg]]

[[编程/UE/Image/UPROPERTY 介绍/d550cf08d5b8c46742936abb6d08c106_MD5.jpeg|Open: image-20240207145545757.png]]
![[编程/UE/Image/UPROPERTY 介绍/d550cf08d5b8c46742936abb6d08c106_MD5.jpeg]]

---
# EditDefaultsOnly
- 和 EditInstanceOnly 完全相反，被 EditDefaultsOnly 修饰的变量 能再蓝图细节面板修改，却无法在实例中修改

[[编程/UE/Image/UPROPERTY 介绍/888585d62f1afa08a0d8df34c3adaba8_MD5.jpeg|Open: image-20240207150056175.png]]
![[编程/UE/Image/UPROPERTY 介绍/888585d62f1afa08a0d8df34c3adaba8_MD5.jpeg]]

[[编程/UE/Image/UPROPERTY 介绍/10d7cc2a515d3e45f59ca9756152e9cc_MD5.jpeg|Open: image-20240207150108485.png]]
![[编程/UE/Image/UPROPERTY 介绍/10d7cc2a515d3e45f59ca9756152e9cc_MD5.jpeg]]

---
# Category
- 用于将变量进行分类

``` cpp
UPROPERTY(EditAnywhere, Category="Animals")
bool bIsCute;

UPROPERTY(EditAnywhere, Category="Animals|Dogs")
FString BarkWord;

UPROPERTY(EditAnywhere, Category="Animals|Birds")
int32 FlyingSpeed = 99;

```

[[编程/UE/Image/UPROPERTY 介绍/240f2a210f48486aa73dfa1a880c52f5_MD5.jpeg|Open: image-20240207152604721.png]]
![[编程/UE/Image/UPROPERTY 介绍/240f2a210f48486aa73dfa1a880c52f5_MD5.jpeg]]

[[编程/UE/Image/UPROPERTY 介绍/c70ca875436312ec67565b948b443556_MD5.jpeg|Open: image-20240207152610132.png]]
![[编程/UE/Image/UPROPERTY 介绍/c70ca875436312ec67565b948b443556_MD5.jpeg]]

# AdvancedDisplay
- 高级属性显示分类

``` cpp
UPROPERTY(EditAnywhere, Category="Toy")  
FString Name;  
UPROPERTY(EditAnywhere, Category="Toy")  
int32 HappyPhraseCount;  
UPROPERTY(EditAnywhere, Category="Toy", AdvancedDisplay)  
bool bEnableEvilMode;
```

[[编程/UE/Image/UPROPERTY 介绍/aaa1f5e736f1ce570a8f932e4a5064f6_MD5.jpeg|Open: image-20240207153948777.png]]
![[编程/UE/Image/UPROPERTY 介绍/aaa1f5e736f1ce570a8f932e4a5064f6_MD5.jpeg]]


---
# DisplayName
- 显示变量名称
``` cpp
UPROPERTY(EditAnywhere, meta=(DisplayName="Display Font"))
FSoftObjectPath DisplayFontPath;
```
---
# ToolTip
- 悬浮显示文本提示
``` cpp
UPROPERTY(EditAnywhere, meta=(ToolTip="Something that's shown when hovering."))
int32 Legs
```

# meta
## HideInDetailPanel
- 隐藏细节面板显示参数，不过好像没啥用

---
## ShowOnlyInnerProperties
- 避免结构展开
- 当您希望避免让用户单击以展开结构时非常有用，例如当它是外部类中的唯一内容时。由结构属性使用。指示内部属性不会显示在可扩展结构内部，而是提升一个级别。

``` cpp
STRUCT()
struct FCat
{
    GENERATED_BODY()
    UPROPERTY(EditDefaultsOnly)
    FString Name;
    UPROPERTY(EditDefaultsOnly)
    int32 Age;
    UPROPERTY(EditDefaultsOnly)
    FLinearColor Color;
};
// 下面的代码按照顺序详细对用，简单来说就是少了一层数据分类
UPROPERTY(EditAnywhere, Category="Cat Without ShowOnlyInnerProperties")
FCat Cat;
UPROPERTY(EditAnywhere, Category="Cat With ShowOnlyInnerProperties", meta=(ShowOnlyInnerProperties))
FCat Cat;
```
[[编程/UE/Image/UPROPERTY 介绍/e3a6c26e98fd3ad7a30a495f9bea08c8_MD5.jpeg|Open: image-20240207151629211.png]]
![[编程/UE/Image/UPROPERTY 介绍/e3a6c26e98fd3ad7a30a495f9bea08c8_MD5.jpeg]]

---
## EditCondition
- 控制参数是否可以编辑
``` cpp
UPROPERTY(EditAnywhere, Category="Toy")  
bool bCanFly;  
  
UPROPERTY(EditAnywhere, Category="Toy", meta=(EditCondition="bCanFly", Units="s"))  
float FlapPeriodSeconds;
```

[[编程/UE/Image/UPROPERTY 介绍/d652aa8ebc37de847be0fd8d39a660c5_MD5.gif|Open: meta 编辑条件.gif]]
![[编程/UE/Image/UPROPERTY 介绍/d652aa8ebc37de847be0fd8d39a660c5_MD5.gif]]

---
## EditConditionHides
- 隐藏参数使用配合 EditCondition
``` cpp
UPROPERTY(EditDefaultsOnly, meta=(EditCondition="PlantType==EPlantType::Flower", EditConditionHides))
FLinearColor FlowerColor = FLinearColor::White;
```
---
## InlineEditConditionToggle
- 内嵌编辑条件变量
- 无法配合 EditConditionHides 使用，原因是 EditConditionHides 会隐藏需要编辑的变量本身，而 InlineEditConditionToggle 会控制隐藏变量标识，最终会导致整个参数不可见（无论 编辑条件是否符合显示的情况，这似乎是UE的保护机制）
- 内联显示在它所控制的属性的左侧。注意，这个元标志应该放在`bool`属性，而不是其他类型的条件
``` cpp
UPROPERTY(EditAnywhere, Category="Toy", meta=(InlineEditConditionToggle))
bool bCanFly;

UPROPERTY(EditAnywhere, Category="Toy", meta=(EditCondition="bCanFly", Units="s"))
float FlapPeriodSeconds;
```

[[编程/UE/Image/UPROPERTY 介绍/eeaaf09c341bcb02164ccc57f090516e_MD5.gif|Open: meta 编辑条件内嵌.gif]]
![[编程/UE/Image/UPROPERTY 介绍/eeaaf09c341bcb02164ccc57f090516e_MD5.gif]]

---
## DisplayAfter
- 控制显示顺序
``` cpp
UPROPERTY(EditAnywhere, meta=(DisplayAfter="ShowFirst"))
int32 ShowAfterFirst1;
UPROPERTY(EditAnywhere, meta=(DisplayAfter="ShowFirst"))
int32 ShowAfterFirst2;
UPROPERTY(EditAnywhere)
int32 ShowFirst;
UPROPERTY(EditAnywhere)
int32 ShowNext;
```

[[编程/UE/Image/UPROPERTY 介绍/1ea30e2194fe96fd0b09af9639f5fed5_MD5.jpeg|Open: image-20240207160154519.png]]
![[编程/UE/Image/UPROPERTY 介绍/1ea30e2194fe96fd0b09af9639f5fed5_MD5.jpeg]]



