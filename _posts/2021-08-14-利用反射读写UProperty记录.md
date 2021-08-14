---
layout:     post
title:      利用反射读写UProperty记录
subtitle:   
date:       2021-08-14
author:     huahai
catalog: true
tags:
    - UnrealEngine
---





## 一、前言

之前有个需求是要做Proto与UObject的一个对等的相互转化。其中UObject侧的核心就是利用反射机制对UObject和UProperty的获取和修改，虽然不是什么难点，不过部分较为复杂的UProperty（例如TArray和TMap的获取与修改）还是让我踩了一些坑的，在此记录一下。

## 二、所有的UProperty

![preview](https://pic4.zhimg.com/v2-a8bd69dd6125cc6f55f05e86470bad77_r.jpg)

## 三、部分UProperty值的读写

### 3.1 FIntProperty

读取：

```c++
FIntProperty* Int32Property = Cast<FIntProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
int32 Int32Value = Int32Property->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
const FIntProperty* Int32Property = Cast<FIntProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
Int32Property->SetPropertyValue_InContainer(ResultObject, ResultInt32, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultInt32为待写入的值。
```

### 3.2 FInt64Property

读取：

```c++
FInt64Property* Int64Property = Cast<FInt64Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
int64 Int64Value = Int64Property->GetPropertyValue_InContainer(TargetObject, 0);// TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FInt64Property* Int64Property = Cast<FInt64Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
Int64Property->SetPropertyValue_InContainer(ResultObject, ResultInt64, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultInt64为待写入的值。
```

### 3.3 FUInt32Property

读取：

```c++
FUInt32Property* Uint32Property = Cast<FUInt32Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
uint32 Uint32Value = Uint32Property->GetPropertyValue_InContainer(TargetObject, 0);// TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FUInt32Property* Uint32Property = Cast<FUInt32Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
Uint32Property->SetPropertyValue_InContainer(ResultObject, ResultUint32, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultUint32为待写入的值。
```

### 3.4 FUInt64Property

读取：

```c++
FUInt64Property* Uint64Property = Cast<FUInt64Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
uint64 Uint64Value = Uint64Property->GetPropertyValue_InContainer(TargetObject, 0);// TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FUInt64Property* Uint64Property = Cast<FUInt64Property>(TargetProperty); // TargetProperty为UProperty*格式的对象
Uint64Property->SetPropertyValue_InContainer(ResultObject, ResultUint64, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultUint64为待写入的值。
```

### 3.5 FDoubleProperty

读取：

```c++
FDoubleProperty* DoubleProperty = Cast<FDoubleProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
double DoubleValue = DoubleProperty->GetPropertyValue_InContainer(TargetObject, 0);// TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FDoubleProperty* DoubleProperty = Cast<FDoubleProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
DoubleProperty->SetPropertyValue_InContainer(ResultObject, ResultDouble, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultDouble为待写入的值。
```

### 3.6 FFloatProperty

读取：

```c++
FFloatProperty* FloatProperty = Cast<FFloatProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
float FloatValue = FloatProperty->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FFloatProperty* FloatProperty = Cast<FFloatProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
FloatProperty->SetPropertyValue_InContainer(ResultObject, ResultFloat, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultFloat为待写入的值。
```

### 3.7 FBoolProperty

读取：

```c++
FBoolProperty* BoolProperty = Cast<FBoolProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
bool BoolValue = BoolProperty->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FBoolProperty* BoolProperty = Cast<FBoolProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
BoolProperty->SetPropertyValue_InContainer(ResultObject, ResultBool, 0); // ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultBool为待写入的值。
```

### 3.8 FByteProperty（TEnumAsByte）

读取：

```c++
FByteProperty* ByteProperty = Cast<FByteProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
int32 ResultValue = ByteProperty->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FByteProperty* ByteProperty = Cast<FByteProperty>(TargetProperty);// TargetProperty为UProperty*格式的对象
ByteProperty->SetPropertyValue_InContainer(ResultObject, ResultInt32, 0);// ResultObject为该UProperty所在的UObject实例，数据的接收者。ResultInt32为待写入的值。
```

### 3.9 FStrProperty

读取：

```c++
FStrProperty* StrProperty = Cast<FStrProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
FString StrValue = StrProperty->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FStrProperty* StrProperty = Cast<FStrProperty>(TargetProperty);// TargetProperty为UProperty*格式的对象
StrProperty->SetPropertyValue_InContainer(ResultObject, FString(ResultString.c_str()), 0);// ResultObject为该UProperty所在的UObject实例，数据的接收者。FString(ResultString.c_str())为待写入的值。
```

### 3.10 FObjectProperty

读取：

```c++
FObjectProperty* TargetObjectProperty = CastField<FObjectProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
UObject* ObjectValue = TargetObjectProperty->GetPropertyValue_InContainer(TargetObject, 0); // TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

```c++
FObjectProperty* TargetObjectProperty = Cast<FObjectProperty>(TargetProperty);// TargetProperty为UProperty*格式的对象
TargetObjectProperty->SetPropertyValue_InContainer(ResultObject, SubObject, 0);// ResultObject为该UProperty所在的UObject实例，数据的接收者。SubObject为待写入的值。
```

### 3.11 FStructProperty

读取：

```c++
FStructProperty* TargetStructProperty = CastField<FStructProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
UScriptStruct* SubStruct = TargetStructProperty->Struct; // 获取UScriptStruct
void* SubStructValue = TargetStructProperty->ContainerPtrToValuePtr<void>(TargetObject, 0);// TargetObject为该UProperty所在的UObject实例，数据的来源

for (TFieldIterator<FProperty> It(SubStruct); It; ++It) // 遍历struct下的UProperty
{
    FProperty* SubStructProperty = *It;

    if (FIntProperty* SubStructInt32 = CastField<FIntProperty>(SubStructProperty))
    {
        int32 IntValue = SubStructInt32->GetPropertyValue_InContainer(SubStructValue, 0); // 这里举例子进行读取
        UE_LOG(LogTemp, Warning, TEXT("SubStructInt32: %d"), IntValue);
    }
}

```

写入：

TODO

### 3.12 FArrayProperty

读取：

```c++
// 根据TArray的元素类型不同而有细微不同，这里以int32为例，其他类型替换int32即可
TArray<int32>* TargetInt32s = TargetProperty->ContainerPtrToValuePtr<TArray<int32>>(TargetObject); // TargetProperty为UProperty*格式的对象，TargetObject为该UProperty所在的UObject实例，数据的来源
```

写入：

TArray的写入操作需要借助FScriptArrayHelper。

```c++
FArrayProperty* TargetArrayProperty = Cast<FArrayProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
void* ArrayValuePtr = TargetArrayProperty->ContainerPtrToValuePtr<void>(ResultObject);
FScriptArrayHelper ArrayHelper(TargetArrayProperty, ArrayValuePtr);
switch (FieldCppType) // FieldCppType是TArray元素的类型，根据元素类型不一样，读取方式不一样
{
case CPPTYPE_INT32:
   {
      TArray<int32> TargetArray; // 根据数据源对TargetArray进行数据填充
       
      // 数据填充...
                
      ArrayHelper.MoveAssign(&TargetArray); // 将填充之后的TargetArray赋值给ArrayHelper，即完成了写入
   }
   break;
case CPPTYPE_INT64:
   break;
case CPPTYPE_UINT32:
   break;
case CPPTYPE_UINT64:
   break;
case CPPTYPE_DOUBLE:
   break;
case CPPTYPE_FLOAT:
   break;
case CPPTYPE_BOOL:
   break;
default: ;
}
```

### 3.13 FMapProperty

读取：

```c++
FMapProperty* TargetMapProperty = Cast<FMapProperty>(TargetProperty); // TargetProperty为UProperty*格式的对象
		// 根据key和value类型的不同，进行读取
         if (CastField<FUInt32Property>(TargetMapProperty->KeyProp)) // key为uint32
         {
            if (CastField<FObjectProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为UObject
            {
               TMap<uint32, UObject*>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, UObject*>>(TargetObject); // 读取TMap里面的数据，TargetObject为该UProperty所在的UObject实例，数据的来源
            }
            else if (CastField<FEnumProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为enum
            {
               TMap<uint32, uint8>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, uint8>>(TargetObject);
            }
#pragma endregion
         }
         else if (CastField<FUInt64Property>(TargetMapProperty->KeyProp)) // key为uint64
         {             
         }
         else if (CastField<FIntProperty>(TargetMapProperty->KeyProp)) // key为int32
         {
         }
         else if (CastField<FStrProperty>(TargetMapProperty->KeyProp)) // key为string
         {
         }
```

写入：

TMap的写入操作需要借助FScriptMapHelper。

```c++
    FMapProperty* TargetMapProperty = Cast<FMapProperty>(TargetProperty);
      void* MapValuePtr = TargetMapProperty->ContainerPtrToValuePtr<void*>(ResultObject, 0);
      // 伪动态映射。用于以一种合理的方式处理映射属性
      FScriptMapHelper MapHelper(TargetMapProperty, MapValuePtr);
      
      switch (cpp_type()) // 判断TMap的key是哪种类型
      {
      case CPPTYPE_INT32: // key int32
         {
#pragma region key int32
            switch (cpp_type()) // 判断TMap的value是哪种类型
            {
            case CPPTYPE_INT32: // key int32 value int32
               {
                  TMap<int32, int32> TargetMap; // 对TargetMap进行数据填充
                   
                   // 数据填充...
                   
                  // 将TargetMap写入MapHelper
                  MapHelper.MoveAssign(&TargetMap);
               }
               break;
            case CPPTYPE_INT64: // key int32 value int64
               {
               }
               break;
            case CPPTYPE_UINT32: // key int32 value UINT32
               {
               }
               break;
            case CPPTYPE_UINT64: // key int32 value UINT64
               {
               }
               break;
            case CPPTYPE_DOUBLE: // key int32 value DOUBLE
               {
               }
               break;
            case CPPTYPE_FLOAT: // key int32 value FLOAT
               {
               }
               break;
            case CPPTYPE_BOOL: // key int32 value BOOL
               {
               }
               break;
            case CPPTYPE_ENUM: // key int32 value ENUM
               {
               }
               break;
            case CPPTYPE_STRING: // key int32 value STRING
               {
               }
               break;
            default: ;
            }
#pragma endregion
         }
         break;
      case CPPTYPE_INT64: // key int64
         {
         }
         break;
      case CPPTYPE_UINT32: // key uint32
         {
         }
         break;
      case CPPTYPE_UINT64: // key uint64
         {
         }
         break;
      case CPPTYPE_STRING: // key string
         {
         }
         break;
      default: ;
      }
```

## 四、参考资料

1. [《InsideUE4》UObject（十一）类型系统构造-构造绑定链接](https://zhuanlan.zhihu.com/p/59553490)
2. [【UE4随笔】反射写代码之TMap](https://zhuanlan.zhihu.com/p/351867411)
3. [UE4:反射系统获取UPROPERTY](https://www.muchenhen.com/2020/08/23/UE4-%E5%8F%8D%E5%B0%84%E7%B3%BB%E7%BB%9F%E8%8E%B7%E5%8F%96UPROPERTY/#%E8%AF%BB%E5%8F%96%E6%96%B9%E6%B3%95)
4. [void*是怎样的存在？](https://zhuanlan.zhihu.com/p/98061960)

