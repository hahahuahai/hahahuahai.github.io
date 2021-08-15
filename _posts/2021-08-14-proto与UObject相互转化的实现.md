---
layout:     post
title:      proto与UObject相互转化的实现
subtitle:   
date:       2021-08-14
author:     huahai
catalog: true
tags:
    - UnrealEngine
---





## 一、痛点

游戏运行开始，有大量的UObject需要转化为message对象，如果程序手动一个个去适配接口填充message，工作量比较大，变动起来维护成本高。所以需要一个工具能自动在运行开始转为message。必要的时候再将message对象自动转为UObject对象。

后来在此基础上又加了一个需求：不要依赖于proto生成的C++文件。

## 二、需要解决的技术点

要完成这个需求，需要用到以下4个技术：

1. 动态编译proto文件，从而不依赖于生成的C++文件。
2. 利用metadata标签将UObject、UStruct与message一一对应，UProperty与message中的字段一一对应。完成UObject、UStruct与message的字段适配。
3. 利用proto的反射技术，运行时能获取message及其字段信息，并且进行读写。
4. 利用UObject的反射技术，运行时能获取属性信息，并且进行读写。包括复杂类型属性（TMap和TArray）的读写。
5. 对复杂类型的嵌套关系处理，例如UObject中嵌套UObject或者UStruct等。对于这种情况要进行人为约束，减少复杂度。

## 三、特殊类型的映射关系的处理

常规类型的映射都比较直观，例如int32、bool等。但是像Enum、UObject、TMap、TArray等类型就没这么直观。

## 四、示例代码

```C++
#include "ProtoReflection.h"

void ProtoReflection::InitProtoReflection(TMap<FString, FString> ProtoPaths)
{
	if (ProtoPaths.Num() <= 0)
	{
		UE_LOG(LogTemp, Warning, TEXT("InitProtoReflection失败！！参数传入错误..."));
		return;
	}

#pragma region dynamic compile
	// 1.准备配置好文件系统
	google::protobuf::compiler::DiskSourceTree sourceTree;
	// 2.将当前路径映射为项目根目录 ， project_root 仅仅是个名字，你可以你想要的合法名字.
	FString PluginPath = FPaths::ProjectPluginsDir();
	PluginPath = FPaths::ConvertRelativePathToFull(PluginPath); // E:/TDM/TDAll/TheDivision_Client/Plugins/
	// UE_LOG(LogTemp, Warning, TEXT("PluginPath : %s"), *PluginPath);

	for (TTuple<FString, FString> ProtoPath : ProtoPaths)
	{
		sourceTree.MapPath(std::string(TCHAR_TO_UTF8(*ProtoPath.Key)), std::string(TCHAR_TO_UTF8(*PluginPath)) + std::string(TCHAR_TO_UTF8(*ProtoPath.Value)));
		sourceTree.MapPath("", std::string(TCHAR_TO_UTF8(*ProtoPath.Value))); // 这个路径是用于当proto中有import其他proto文件的时候告诉编译器去哪个目录去找，这个一定要添加，不然导入失败。
	}
	// 配置动态编译器.
	ImporterInstance = new compiler::Importer(&sourceTree, nullptr);
	// 3.获取所有proto文件名
	for (TTuple<FString, FString> ProtoPath : ProtoPaths)
	{
		FString ProtoFullPath(PluginPath + ProtoPath.Value); // Proto/Source/TDProto/cscommon/protocol/cs
		const TCHAR* ProtoFolder = ProtoFullPath.GetCharArray().GetData(); // 获取proto文件存放文件夹路径
		IFileManager& MyFileManager = IFileManager::Get();
		UE_LOG(LogTemp, Warning, TEXT("ProtoPath.Key ProtoFullName : %s"), *ProtoPath.Key);
		if (MyFileManager.DirectoryExists(ProtoFolder))
		{
			TArray<FString> ProtoFullNames;
			MyFileManager.FindFiles(ProtoFullNames, ProtoFolder);
			UE_LOG(LogTemp, Warning, TEXT("ProtoFullNames num : %d"), ProtoFullNames.Num());
			if (ProtoFullNames.Num() > 0)
			{
				for (FString ProtoFullName : ProtoFullNames)
				{
					// 判断是否为proto文件
					if (!ProtoFullName.Contains(".proto"))
					{
						continue;
					}
					// 动态编译proto源文件。
					ImporterInstance->Import(std::string(TCHAR_TO_UTF8(*ProtoPath.Key)) + "/" + std::string(TCHAR_TO_UTF8(*ProtoFullName)));
				}
			}
		}
	}
#pragma endregion

#pragma region
	for (TObjectIterator<UClass> It; It; ++It)
	{
		UClass* TargetClass = *It;
		const FString& ProtoMessageName = TargetClass->GetMetaData(TEXT("ProtoMessageName"));
		if (ProtoMessageName.Len() > 0)
		{
			Message2UClassMap.Add(*ProtoMessageName, TargetClass);
		}
	}
	for (TObjectIterator<UEnum> It; It; ++It)
	{
		UEnum* TargetEnum = *It;
		const FString& ProtoMessageName = TargetEnum->GetMetaData(TEXT("ProtoMessageName"));
		if (ProtoMessageName.Len() > 0)
		{
			Message2UEnumMap.Add(*ProtoMessageName, TargetEnum);
		}
	}
#pragma endregion
}

Message* ProtoReflection::GetMessageFromObject(UObject* TargetObject)
{
	// 初始化，动态编译协议等
	if (ImporterInstance == nullptr)
	{
		TMap<FString, FString> ProtoPaths;
		ProtoPaths.Add("cs_path", "Proto/Source/TDProto/cscommon/protocol/cs");
		ProtoPaths.Add("object_path", "Proto/Source/TDProto/cscommon/protocol/object");
		ProtoPaths.Add("comm_path", "Proto/Source/TDProto/cscommon/protocol/comm");
		ProtoPaths.Add("resource_path", "Proto/Source/TDProto/cscommon/resource");
		InitProtoReflection(ProtoPaths);
	}

	// 提取协议类型名字
	const FString& MessageTypeName = TargetObject->GetClass()->GetMetaData(TEXT("ProtoMessageName"));
	if (MessageTypeName.Len() <= 0) // 没有添加协议标签，无法进行
	{
		return nullptr;
	}

	// 现在可以从编译器中提取类型的描述信息.
	UE_LOG(LogTemp, Warning, TEXT("MessageTypeName : %s"), *MessageTypeName);
	const Descriptor* BaseDescriptor = ImporterInstance->pool()->FindMessageTypeByName(std::string(TCHAR_TO_UTF8(*MessageTypeName)));
	if (BaseDescriptor == nullptr)
	{
		UE_LOG(LogTemp, Warning, TEXT("BaseDescriptor == nullptr!!!!!!!!!!!"));
		return nullptr;
	}

	UE_LOG(LogTemp, Warning, TEXT("BaseDescriptor->full_name: %s"), *FString(BaseDescriptor->full_name().c_str()));
	UE_LOG(LogTemp, Warning, TEXT("descriptor->name : %s"), *FString(BaseDescriptor->name().c_str()));

	const Message* MessagePrototype = DynamicMessageFactoryInstance.GetPrototype(BaseDescriptor);

	Message* MessageInstance = MessagePrototype->New();
	const Reflection* ReflectionInstance = MessageInstance->GetReflection();

	const FString& ProtoMessageName = TargetObject->GetClass()->GetMetaData(TEXT("ProtoMessageName"));

	// 收集属性和标签
	TArray<PropertyData*> PropertyDataMap;
	for (TFieldIterator<UProperty> i(TargetObject->GetClass()); i; ++i) // 遍历属性获取标签
	{
		UProperty* TargetProperty = *i;
		const FString& MessageFieldName = TargetProperty->GetMetaData(TEXT("MessageFieldName"));

		if (ProtoMessageName.Len() > 0)
		{
			PropertyDataMap.Add(new ProtoReflection::PropertyData(ProtoMessageName, MessageFieldName, TargetProperty, TargetObject));
		}
	}
	UE_LOG(LogTemp, Warning, TEXT("PropertyDataMap.Num: %d"), PropertyDataMap.Num());

	// 协议字段赋值
	UE_LOG(LogTemp, Warning, TEXT("BaseDescriptor->field_count: %d"), BaseDescriptor->field_count());
	for (int i = 0; i < BaseDescriptor->field_count(); ++i) // 遍历协议字段
	{
		const FieldDescriptor* TargetFieldDescriptor = BaseDescriptor->field(i);
		UE_LOG(LogTemp, Warning, TEXT("TargetFieldDescriptor->name : %s"), *FString(TargetFieldDescriptor->name().c_str()));
		switch (TargetFieldDescriptor->cpp_type())
		{
		case FieldDescriptor::CPPTYPE_INT32: // CPPTYPE_INT32
			{
				UE_LOG(LogTemp, Warning, TEXT("协议中存在int32字段name: %s"), *FString(TargetFieldDescriptor->name().c_str()));
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_INT32);
			}
			break;
		case FieldDescriptor::CPPTYPE_INT64: // CPPTYPE_INT64
			{
				UE_LOG(LogTemp, Warning, TEXT("协议中存在int64字段name: %s"), *FString(TargetFieldDescriptor->name().c_str()));
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_INT64);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT32: // CPPTYPE_UINT32
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_UINT32);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT64: // CPPTYPE_UINT64
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_UINT64);
			}
			break;
		case FieldDescriptor::CPPTYPE_DOUBLE: // CPPTYPE_DOUBLE
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_DOUBLE);
			}
			break;
		case FieldDescriptor::CPPTYPE_FLOAT: // CPPTYPE_FLOAT
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_FLOAT);
			}
			break;
		case FieldDescriptor::CPPTYPE_BOOL: // CPPTYPE_BOOL
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_BOOL);
			}
			break;
		case FieldDescriptor::CPPTYPE_ENUM: // CPPTYPE_ENUM
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_ENUM);
			}
			break;
		case FieldDescriptor::CPPTYPE_STRING: // CPPTYPE_STRING
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_STRING);
			}
			break;
		case FieldDescriptor::CPPTYPE_MESSAGE: // CPPTYPE_MESSAGE
#pragma region message ALL
			{
				SetDataInMessage(MessageInstance, PropertyDataMap, TargetFieldDescriptor, FieldDescriptor::CPPTYPE_MESSAGE);
			}
#pragma endregion
			break;
		default: ;
		}
	}
	return MessageInstance;
}


UObject* ProtoReflection::GetUObjectFromMessage(const Message* TargetMessage)
{
	// 必要的初始化
	if (ImporterInstance == nullptr)
	{
		TMap<FString, FString> ProtoPaths;
		ProtoPaths.Add("cs_path", "Proto/Source/TDProto/cscommon/protocol/cs");
		ProtoPaths.Add("object_path", "Proto/Source/TDProto/cscommon/protocol/object");
		ProtoPaths.Add("comm_path", "Proto/Source/TDProto/cscommon/protocol/comm");
		ProtoPaths.Add("resource_path", "Proto/Source/TDProto/cscommon/resource");
		InitProtoReflection(ProtoPaths);
	}

	UObject* ResultObject = nullptr;
	const Descriptor* BaseDescriptor = TargetMessage->GetDescriptor();
	const Reflection* BaseReflection = TargetMessage->GetReflection();
	UE_LOG(LogTemp, Warning, TEXT("BaseDescriptor->full_name : %s"), *FString(BaseDescriptor->full_name().c_str()));

	UClass* CurrentUClass = Message2UClassMap[FString(BaseDescriptor->full_name().c_str())];
	if (CurrentUClass == nullptr) { return nullptr; }

	ResultObject = CurrentUClass->ClassDefaultObject; // 用CDO来初始化
	const FString& ProtoMessageName = CurrentUClass->GetMetaData(TEXT("ProtoMessageName"));
	for (TFieldIterator<UProperty> i(CurrentUClass); i; ++i) // 遍历属性获取标签
	{
		UProperty* TargetProperty = *i;

		const FString& MessageFieldName = TargetProperty->GetMetaData(TEXT("MessageFieldName"));

		if (ProtoMessageName.Len() > 0)
		{
			UE_LOG(LogTemp, Warning, TEXT("GetUObjectFromMessage ProtoMessageName : %s"), *ProtoMessageName);
			if (FString(BaseDescriptor->full_name().c_str()) == ProtoMessageName) // 找到协议名称相等的uproperty
			{
				UE_LOG(LogTemp, Warning, TEXT("FString(BaseDescriptor->full_name().c_str()) == ProtoMessageName"));
				if (MessageFieldName.Len() > 0)
				{
					UE_LOG(LogTemp, Warning, TEXT("GetUObjectFromMessage MessageFieldName : %s"), *MessageFieldName);

					const FieldDescriptor* TargetFieldDescriptor = BaseDescriptor->FindFieldByName(std::string(TCHAR_TO_UTF8(*MessageFieldName))); // 找到message中对应的field
					if (TargetFieldDescriptor == nullptr)
					{
						continue;
					}
					switch (TargetFieldDescriptor->cpp_type())
					{
					case FieldDescriptor::CPPTYPE_INT32: // int32
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_INT32);
						}
						break;
					case FieldDescriptor::CPPTYPE_INT64:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_INT64);
						}
						break;
					case FieldDescriptor::CPPTYPE_UINT32:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_UINT32);
						}
						break;
					case FieldDescriptor::CPPTYPE_UINT64:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_UINT64);
						}
						break;
					case FieldDescriptor::CPPTYPE_DOUBLE:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_DOUBLE);
						}
						break;
					case FieldDescriptor::CPPTYPE_FLOAT:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_FLOAT);
						}
						break;
					case FieldDescriptor::CPPTYPE_BOOL:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_BOOL);
						}
						break;
					case FieldDescriptor::CPPTYPE_ENUM:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_ENUM);
						}
						break;
					case FieldDescriptor::CPPTYPE_STRING:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_STRING);
						}
						break;
					case FieldDescriptor::CPPTYPE_MESSAGE:
						{
							SetDataInObject(ResultObject, TargetMessage, TargetFieldDescriptor, TargetProperty, FieldDescriptor::CPPTYPE_MESSAGE);
						}
						break;
					default: ;
					}
				}
			}
		}
	}
	return ResultObject;
}

void ProtoReflection::SetDataInMessage(Message* MessageInstance, const TArray<PropertyData*> PropertyDataMap, const FieldDescriptor* TargetFieldDescriptor, const FieldDescriptor::CppType FieldCppType)
{
	const FString MessageTypeName(MessageInstance->GetTypeName().c_str());
	const Reflection* ReflectionInstance = MessageInstance->GetReflection();


	// 获取UProperty上的数据
	if (TargetFieldDescriptor->is_map()) // message map
	{
#pragma region message map
		FString SubMessageTypeName(TargetFieldDescriptor->message_type()->full_name().c_str());
		for (PropertyData* SinglePropertyDataMap : PropertyDataMap)
		{
			if (SinglePropertyDataMap->ProtoMessageName != MessageTypeName) // 判断是否为同一个协议
			{
				continue;
			}
			if (FString(TargetFieldDescriptor->name().c_str()) != SinglePropertyDataMap->MessageFieldName) // 判断是否为同一个字段
			{
				continue;
			}

			// 获取UProperty上的数据
			FMapProperty* TargetMapProperty = Cast<FMapProperty>(SinglePropertyDataMap->TaggedProperty);
			if (CastField<FUInt32Property>(TargetMapProperty->KeyProp)) // key为uint32
			{
				UE_LOG(LogTemp, Warning, TEXT("key为FUInt32Property"));
#pragma region key uint32
				if (CastField<FObjectProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为message
				{
					TMap<uint32, UObject*>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, UObject*>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, UObject*> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						Message* ResultSubMessage = GetMessageFromObject(SingleObjectMap.Value);
						MapReflection->SetAllocatedMessage(MapMessageInstance, ResultSubMessage, ValueFieldDescriptor); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FEnumProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为enum
				{
					TMap<uint32, uint8>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, uint8>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, uint8> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetEnumValue(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FIntProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为int32
				{
					TMap<uint32, int32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, int32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, int32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FInt64Property>(TargetMapProperty->ValueProp)) // key为uint32,value为int64
				{
					TMap<uint32, int64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, int64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, int64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt32Property>(TargetMapProperty->ValueProp)) // key为uint32,value为uint32
				{
					TMap<uint32, uint32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, uint32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, uint32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt64Property>(TargetMapProperty->ValueProp)) // key为uint32,value为uint64
				{
					TMap<uint32, uint64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, uint64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, uint64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FDoubleProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为double
				{
					TMap<uint32, double>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, double>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, double> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetDouble(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FFloatProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为float
				{
					TMap<uint32, float>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, float>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, float> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetFloat(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为bool
				{
					TMap<uint32, bool>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, bool>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, bool> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetBool(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为uint32,value为string
				{
					TMap<uint32, FString>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint32, FString>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint32, FString> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetString(MapMessageInstance, ValueFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Value))); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
#pragma endregion
			}
			else if (CastField<FUInt64Property>(TargetMapProperty->KeyProp)) // key为uint64
			{
				UE_LOG(LogTemp, Warning, TEXT("key为FUInt64Property"));
#pragma region key uint64
				if (CastField<FObjectProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为message
				{
					TMap<uint64, UObject*>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, UObject*>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, UObject*> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						Message* ResultSubMessage = GetMessageFromObject(SingleObjectMap.Value);
						MapReflection->SetAllocatedMessage(MapMessageInstance, ResultSubMessage, ValueFieldDescriptor); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FEnumProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为enum
				{
					TMap<uint64, uint8>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, uint8>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, uint8> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetEnumValue(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FIntProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为int32
				{
					TMap<uint64, int32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, int32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, int32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FInt64Property>(TargetMapProperty->ValueProp)) // key为uint64,value为int64
				{
					TMap<uint64, int64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, int64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, int64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt32Property>(TargetMapProperty->ValueProp)) // key为uint64,value为uint32
				{
					TMap<uint64, uint32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, uint32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, uint32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt64Property>(TargetMapProperty->ValueProp)) // key为uint64,value为uint64
				{
					TMap<uint64, uint64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, uint64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, uint64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FDoubleProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为double
				{
					TMap<uint64, double>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, double>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, double> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetDouble(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FFloatProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为float
				{
					TMap<uint64, float>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, float>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, float> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetFloat(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为bool
				{
					TMap<uint64, bool>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, bool>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, bool> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetBool(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为uint64,value为string
				{
					TMap<uint64, FString>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<uint64, FString>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<uint64, FString> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetUInt64(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetString(MapMessageInstance, ValueFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Value))); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
#pragma endregion
			}
			else if (CastField<FIntProperty>(TargetMapProperty->KeyProp)) // key为int32
			{
				UE_LOG(LogTemp, Warning, TEXT("key为FIntProperty"));
#pragma region key int32
				if (CastField<FObjectProperty>(TargetMapProperty->ValueProp)) // key为int32,value为message
				{
					TMap<int32, UObject*>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, UObject*>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, UObject*> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						Message* ResultSubMessage = GetMessageFromObject(SingleObjectMap.Value);
						MapReflection->SetAllocatedMessage(MapMessageInstance, ResultSubMessage, ValueFieldDescriptor); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FEnumProperty>(TargetMapProperty->ValueProp)) // key为int32,value为enum
				{
					TMap<int32, uint8>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, uint8>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, uint8> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetEnumValue(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FIntProperty>(TargetMapProperty->ValueProp)) // key为int32,value为int32
				{
					TMap<int32, int32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, int32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, int32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FInt64Property>(TargetMapProperty->ValueProp)) // key为int32,value为int64
				{
					TMap<int32, int64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, int64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, int64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt32Property>(TargetMapProperty->ValueProp)) // key为int32,value为uint32
				{
					TMap<int32, uint32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, uint32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, uint32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt64Property>(TargetMapProperty->ValueProp)) // key为int32,value为uint64
				{
					TMap<int32, uint64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, uint64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, uint64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetUInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FDoubleProperty>(TargetMapProperty->ValueProp)) // key为int32,value为double
				{
					TMap<int32, double>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, double>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, double> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetDouble(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FFloatProperty>(TargetMapProperty->ValueProp)) // key为int32,value为float
				{
					TMap<int32, float>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, float>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, float> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetFloat(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为int32,value为bool
				{
					TMap<int32, bool>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, bool>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, bool> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetBool(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FStrProperty>(TargetMapProperty->ValueProp)) // key为int32,value为string
				{
					TMap<int32, FString>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<int32, FString>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<int32, FString> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetInt32(MapMessageInstance, KeyFieldDescriptor, SingleObjectMap.Key); // map单个元素中的key赋值
						MapReflection->SetString(MapMessageInstance, ValueFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Value))); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
#pragma endregion
			}
			else if (CastField<FStrProperty>(TargetMapProperty->KeyProp)) // key为string
			{
				UE_LOG(LogTemp, Warning, TEXT("key为FStrProperty"));
#pragma region key string
				if (CastField<FObjectProperty>(TargetMapProperty->ValueProp)) // key为string,value为message
				{
					TMap<FString, UObject*>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, UObject*>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, UObject*> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						Message* ResultSubMessage = GetMessageFromObject(SingleObjectMap.Value);
						MapReflection->SetAllocatedMessage(MapMessageInstance, ResultSubMessage, ValueFieldDescriptor); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FEnumProperty>(TargetMapProperty->ValueProp)) // key为string,value为enum
				{
					TMap<FString, uint8>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, uint8>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, uint8> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetEnumValue(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FIntProperty>(TargetMapProperty->ValueProp)) // key为string,value为int32
				{
					TMap<FString, int32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, int32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, int32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FInt64Property>(TargetMapProperty->ValueProp)) // key为string,value为int64
				{
					TMap<FString, int64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, int64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, int64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt32Property>(TargetMapProperty->ValueProp)) // key为string,value为uint32
				{
					TMap<FString, uint32>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, uint32>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, uint32> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetUInt32(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FUInt64Property>(TargetMapProperty->ValueProp)) // key为string,value为uint64
				{
					TMap<FString, uint64>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, uint64>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, uint64> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetUInt64(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FDoubleProperty>(TargetMapProperty->ValueProp)) // key为string,value为double
				{
					TMap<FString, double>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, double>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, double> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetDouble(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FFloatProperty>(TargetMapProperty->ValueProp)) // key为string,value为float
				{
					TMap<FString, float>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, float>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, float> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetFloat(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为string,value为bool
				{
					TMap<FString, bool>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, bool>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, bool> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetBool(MapMessageInstance, ValueFieldDescriptor, SingleObjectMap.Value); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
				else if (CastField<FBoolProperty>(TargetMapProperty->ValueProp)) // key为string,value为string
				{
					TMap<FString, FString>* TargetObjectMap = TargetMapProperty->ContainerPtrToValuePtr<TMap<FString, FString>>(SinglePropertyDataMap->ObjectBelongTo);
					const Message* MapMessagePrototype = DynamicMessageFactoryInstance.GetPrototype(TargetFieldDescriptor->message_type());
					for (TTuple<FString, FString> SingleObjectMap : *TargetObjectMap)
					{
						Message* MapMessageInstance = MapMessagePrototype->New();
						const Reflection* MapReflection = MapMessageInstance->GetReflection();
						const FieldDescriptor* KeyFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("key");
						const FieldDescriptor* ValueFieldDescriptor = MapMessageInstance->GetDescriptor()->FindFieldByName("value");

						// 设置数据
						MapReflection->SetString(MapMessageInstance, KeyFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Key))); // map单个元素中的key赋值
						MapReflection->SetString(MapMessageInstance, ValueFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleObjectMap.Value))); // map单个元素中的value赋值	 
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, MapMessageInstance); // 将单个元素赋值到map中
					}
				}
#pragma endregion
			}
			break;
		}
#pragma endregion
	}
	else if (TargetFieldDescriptor->is_repeated())
	{
#pragma region repeated
		for (PropertyData* SinglePropertyDataMap : PropertyDataMap)
		{
			if (SinglePropertyDataMap->ProtoMessageName != MessageTypeName) // 判断是否为同一个协议
			{
				continue;
			}
			UE_LOG(LogTemp, Warning, TEXT("SinglePropertyDataMap.Value->MessageFieldName: %s"), *SinglePropertyDataMap->MessageFieldName);

			if (FString(TargetFieldDescriptor->name().c_str()) != SinglePropertyDataMap->MessageFieldName) // 判断是否为同一个字段
			{
				continue;
			}
			// 获取UProperty上的数据
			UE_LOG(LogTemp, Warning, TEXT("TaggedProperty->GetName: %s"), *SinglePropertyDataMap->TaggedProperty->GetName());

			switch (FieldCppType) // 处理除了message类型以外的类型
			{
			case FieldDescriptor::CPPTYPE_INT32:
				{
					TArray<int32>* TargetInt32s = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<int32>>(SinglePropertyDataMap->ObjectBelongTo);
					for (int32 SingleObject : *TargetInt32s)
					{
						// 设置数据
						ReflectionInstance->AddInt32(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_INT64:
				{
					TArray<int64>* TargetInt64s = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<int64>>(SinglePropertyDataMap->ObjectBelongTo);
					for (int64 SingleObject : *TargetInt64s)
					{
						// 设置数据
						ReflectionInstance->AddInt64(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_UINT32:
				{
					TArray<uint32>* TargetUint32s = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<uint32>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetUint32s->Num: %d"), TargetUint32s->Num());
					for (uint32 SingleObject : *TargetUint32s)
					{
						// 设置数据
						ReflectionInstance->AddUInt32(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_UINT64:
				{
					TArray<uint64>* TargetUint64s = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<uint64>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetUint64s->Num: %d"), TargetUint64s->Num());
					for (uint64 SingleObject : *TargetUint64s)
					{
						// 设置数据
						ReflectionInstance->AddUInt64(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_DOUBLE:
				{
					TArray<double>* TargetDoubles = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<double>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetDoubles->Num: %d"), TargetDoubles->Num());
					for (double SingleObject : *TargetDoubles)
					{
						// 设置数据
						ReflectionInstance->AddDouble(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_FLOAT:
				{
					TArray<float>* TargetFloats = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<float>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetFloats->Num: %d"), TargetFloats->Num());
					for (float SingleObject : *TargetFloats)
					{
						// 设置数据
						ReflectionInstance->AddFloat(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_BOOL:
				{
					TArray<bool>* TargetBools = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<bool>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetBools->Num: %d"), TargetBools->Num());
					for (bool SingleObject : *TargetBools)
					{
						// 设置数据
						ReflectionInstance->AddBool(MessageInstance, TargetFieldDescriptor, SingleObject);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_ENUM:
				{
					TArray<uint8>* TargetEnums = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<uint8>>(SinglePropertyDataMap->ObjectBelongTo);
					// 通过uint8的格式来读取enum，将enum转为整数
					UE_LOG(LogTemp, Warning, TEXT("222TargetObjects->Num: %d"), TargetEnums->Num());

					for (uint8 SingleEnum : *TargetEnums)
					{
						// 设置数据
						UE_LOG(LogTemp, Warning, TEXT("(uint8)SingleEnum: %d"), SingleEnum);
						ReflectionInstance->AddEnumValue(MessageInstance, TargetFieldDescriptor, SingleEnum);
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_STRING:
				{
					// 获取UProperty上的数据
					TArray<FString>* TargetStrings = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<FString>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetStrings->Num: %d"), TargetStrings->Num());
					for (FString SingleString : *TargetStrings)
					{
						// 设置数据
						ReflectionInstance->AddString(MessageInstance, TargetFieldDescriptor, std::string(TCHAR_TO_UTF8(*SingleString)));
					}
				}
				break;
			case FieldDescriptor::CPPTYPE_MESSAGE:
				{
					TArray<UObject*>* TargetObjects = SinglePropertyDataMap->TaggedProperty->ContainerPtrToValuePtr<TArray<UObject*>>(SinglePropertyDataMap->ObjectBelongTo);
					UE_LOG(LogTemp, Warning, TEXT("222TargetObjects->Num: %d"), TargetObjects->Num());

					for (UObject* SingleObject : *TargetObjects)
					{
						// 设置数据
						Message* ResultSubMessage = GetMessageFromObject(SingleObject);
						ReflectionInstance->AddAllocatedMessage(MessageInstance, TargetFieldDescriptor, ResultSubMessage); // 子协议赋值给父协议对应repeated字段
					}
				}
				break;
			default: ;
			}


			break;
		}
#pragma endregion
	}
	else // singular
	{
#pragma region singular
		for (PropertyData* SinglePropertyDataMap : PropertyDataMap)
		{
			UE_LOG(LogTemp, Warning, TEXT("SinglePropertyDataMap->ProtoMessageName: %s"), *SinglePropertyDataMap->ProtoMessageName);
			UE_LOG(LogTemp, Warning, TEXT("SinglePropertyDataMap.Key MessageTypeName: %s"), *MessageTypeName);
			if (SinglePropertyDataMap->ProtoMessageName != MessageTypeName) // 判断是否为同一个协议
			{
				continue;
			}
			UE_LOG(LogTemp, Warning, TEXT("SinglePropertyDataMap.Value->MessageFieldName: %s"), *SinglePropertyDataMap->MessageFieldName);

			if (FString(TargetFieldDescriptor->name().c_str()) != SinglePropertyDataMap->MessageFieldName) // 判断是否为同一个字段
			{
				continue;
			}
			UE_LOG(LogTemp, Warning, TEXT("存在uproperty有此字段的数据！！！"));

			UE_LOG(LogTemp, Warning, TEXT("TaggedProperty->GetName: %s"), *SinglePropertyDataMap->TaggedProperty->GetName());
			// 获取UProperty上的数据
			switch (FieldCppType)
			{
			case FieldDescriptor::CPPTYPE_INT32:
				{
					FIntProperty* Int32Property = Cast<FIntProperty>(SinglePropertyDataMap->TaggedProperty);
					int32 Int32Value = Int32Property->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetInt32(MessageInstance, TargetFieldDescriptor, Int32Value);
					// UE_LOG(LogTemp, Warning, TEXT("Int32Value: %d"), Int32Value);
				}
				break;
			case FieldDescriptor::CPPTYPE_INT64:
				{
					FInt64Property* Int64Property = Cast<FInt64Property>(SinglePropertyDataMap->TaggedProperty);
					int64 Int64Value = Int64Property->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetInt64(MessageInstance, TargetFieldDescriptor, Int64Value);
					// UE_LOG(LogTemp, Warning, TEXT("1111Int64Value: %d"), Int64Value);
				}
				break;
			case FieldDescriptor::CPPTYPE_UINT32:
				{
					FUInt32Property* Uint32Property = Cast<FUInt32Property>(SinglePropertyDataMap->TaggedProperty);
					uint32 Uint32Value = Uint32Property->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetUInt32(MessageInstance, TargetFieldDescriptor, Uint32Value);
					// UE_LOG(LogTemp, Warning, TEXT("11111Uint32Value: %d"), Uint32Value);
				}
				break;
			case FieldDescriptor::CPPTYPE_UINT64:
				{
					FUInt64Property* Uint64Property = Cast<FUInt64Property>(SinglePropertyDataMap->TaggedProperty);
					uint64 Uint64Value = Uint64Property->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetUInt64(MessageInstance, TargetFieldDescriptor, Uint64Value);
					// UE_LOG(LogTemp, Warning, TEXT("11111Uint64Value: %d"), Uint64Value);
				}
				break;
			case FieldDescriptor::CPPTYPE_DOUBLE:
				{
					FDoubleProperty* DoubleProperty = Cast<FDoubleProperty>(SinglePropertyDataMap->TaggedProperty);
					double DoubleValue = DoubleProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetDouble(MessageInstance, TargetFieldDescriptor, DoubleValue);
					// UE_LOG(LogTemp, Warning, TEXT("11111DoubleValue: %d"), DoubleValue);
				}
				break;
			case FieldDescriptor::CPPTYPE_FLOAT:
				{
					FFloatProperty* FloatProperty = Cast<FFloatProperty>(SinglePropertyDataMap->TaggedProperty);
					float FloatValue = FloatProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetFloat(MessageInstance, TargetFieldDescriptor, FloatValue);
					// UE_LOG(LogTemp, Warning, TEXT("11111FloatValue: %d"), FloatValue);
				}
				break;
			case FieldDescriptor::CPPTYPE_BOOL:
				{
					FBoolProperty* BoolProperty = Cast<FBoolProperty>(SinglePropertyDataMap->TaggedProperty);
					bool BoolValue = BoolProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetBool(MessageInstance, TargetFieldDescriptor, BoolValue);
					// UE_LOG(LogTemp, Warning, TEXT("11111BoolValue: %d"), BoolValue);
				}
				break;
			case FieldDescriptor::CPPTYPE_ENUM:
				{
					FByteProperty* ByteProperty = Cast<FByteProperty>(SinglePropertyDataMap->TaggedProperty);
					int32 ResultValue = ByteProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// int* ResultValue = ByteProperty->ContainerPtrToValuePtr<int32>(TargetObject, 0);
					UE_LOG(LogTemp, Warning, TEXT("enum ResultValue : %d"), ResultValue);
					// 设置数据
					const EnumDescriptor* TargetEnumDescriptor = TargetFieldDescriptor->enum_type();
					const EnumValueDescriptor* TargetEnumValueDescriptor = TargetEnumDescriptor->FindValueByNumber(ResultValue);
					ReflectionInstance->SetEnum(MessageInstance, TargetFieldDescriptor, TargetEnumValueDescriptor);
				}
				break;
			case FieldDescriptor::CPPTYPE_STRING:
				{
					FStrProperty* StrProperty = Cast<FStrProperty>(SinglePropertyDataMap->TaggedProperty);
					FString StrValue = StrProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
					// 设置数据
					ReflectionInstance->SetString(MessageInstance, TargetFieldDescriptor, std::string(TCHAR_TO_UTF8(*StrValue)));
					// UE_LOG(LogTemp, Warning, TEXT("11111StrValue: %s"), *StrValue);
				}
				break;
			case FieldDescriptor::CPPTYPE_MESSAGE:
				{
					if (FObjectProperty* TargetObjectProperty = CastField<FObjectProperty>(SinglePropertyDataMap->TaggedProperty)) // message对应uobject的情况
					{
						UObject* ObjectValue = TargetObjectProperty->GetPropertyValue_InContainer(SinglePropertyDataMap->ObjectBelongTo, 0);
						// 设置数据
						Message* ResultSubMessage = GetMessageFromObject(ObjectValue);

						ReflectionInstance->SetAllocatedMessage(MessageInstance, ResultSubMessage, TargetFieldDescriptor); // 子协议赋值给父协议对应字段							
					}
					else if (FStructProperty* TargetStructProperty = CastField<FStructProperty>(SinglePropertyDataMap->TaggedProperty)) // message对应ustruct的情况
					{
						UScriptStruct* SubStruct = TargetStructProperty->Struct;
						void* SubStructValue = TargetStructProperty->ContainerPtrToValuePtr<void>(SinglePropertyDataMap->ObjectBelongTo, 0);
						Message* SubStructMessage = SetUStructData(SubStruct, SubStructValue);
					}
				}
				break;
			default: ;
			}

			break;
		}
#pragma endregion
	}
}

void ProtoReflection::SetDataInObject(UObject* ResultObject, const Message* TargetMessage, const FieldDescriptor* TargetFieldDescriptor, UProperty* TargetProperty,
                                      const FieldDescriptor::CppType FieldCppType)
{
	const Reflection* BaseReflection = TargetMessage->GetReflection();
	if (TargetFieldDescriptor->is_map()) // message map
	{
#pragma region message map
		UE_LOG(LogTemp, Warning, TEXT("TargetFieldDescriptor BaseReflection->FieldSize : %d"), BaseReflection->FieldSize(*TargetMessage, TargetFieldDescriptor));
		int32 MapCount = BaseReflection->FieldSize(*TargetMessage, TargetFieldDescriptor); // 获得这个map的个数
		
		FMapProperty* TargetMapProperty = Cast<FMapProperty>(TargetProperty);
		void* MapValuePtr = TargetMapProperty->ContainerPtrToValuePtr<void*>(ResultObject, 0);
		// 伪动态映射。用于以一种合理的方式处理映射属性
		FScriptMapHelper MapHelper(TargetMapProperty, MapValuePtr);
		
		switch (TargetFieldDescriptor->message_type()->FindFieldByName("key")->cpp_type())
		{
		case FieldDescriptor::CPPTYPE_INT32: // key int32
			{
#pragma region key int32
				switch (TargetFieldDescriptor->message_type()->FindFieldByName("value")->cpp_type())
				{
				case FieldDescriptor::CPPTYPE_INT32: // key int32 value int32
					{
						TMap<int32, int32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_INT64: // key int32 value int64
					{
						TMap<int32, int64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT32: // key int32 value UINT32
					{
						TMap<int32, uint32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT64: // key int32 value UINT64
					{
						TMap<int32, uint64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_DOUBLE: // key int32 value DOUBLE
					{
						TMap<int32, double> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							double ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetDouble(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_FLOAT: // key int32 value FLOAT
					{
						TMap<int32, float> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							float ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetFloat(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_BOOL: // key int32 value BOOL
					{
						TMap<int32, bool> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							bool ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetBool(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_ENUM: // key int32 value ENUM
					{
						TMap<int32, uint8> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint8 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetEnumValue(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_STRING: // key int32 value STRING
					{
						TMap<int32, FString> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							string ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, ValueFieldDescriptor);
							// 获得value字段的值

							TargetMap.Add(KeyFieldValue, FString(ValueFieldMessageInstance.c_str()));
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_MESSAGE: // key int32 value MESSAGE
					{
						TMap<int32, UObject*> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							const Message& ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetMessage(*SingleMapMessageInstance, ValueFieldDescriptor);

							UObject* ValueFieldObject = GetUObjectFromMessage(&ValueFieldMessageInstance); // 获得value字段的值										
							UE_LOG(LogTemp, Warning, TEXT("KeyFieldDescriptor KeyFieldValue : %d"), KeyFieldValue);
							ULittleMessage* SpecObject = Cast<ULittleMessage>(ValueFieldObject);
							UE_LOG(LogTemp, Warning, TEXT("ValueFieldDescriptor SpecObject->LitInt : %d"), SpecObject->LitInt);
							UE_LOG(LogTemp, Warning, TEXT("MapHelper.GetMaxIndex : %d"), MapHelper.GetMaxIndex());
							TargetMap.Add(KeyFieldValue, DuplicateObject(ValueFieldObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				default: ;
				}
#pragma endregion
			}
			break;
		case FieldDescriptor::CPPTYPE_INT64: // key int64
			{
#pragma region key int64
				switch (TargetFieldDescriptor->message_type()->FindFieldByName("value")->cpp_type())
				{
				case FieldDescriptor::CPPTYPE_INT32: // key int64 value int32
					{
						TMap<int64, int32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_INT64: // key int64 value int64
					{
						TMap<int64, int64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT32: // key int64 value UINT32
					{
						TMap<int64, uint32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT64: // key int64 value UINT64
					{
						TMap<int64, uint64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_DOUBLE: // key int64 value DOUBLE
					{
						TMap<int64, double> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							double ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetDouble(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_FLOAT: // key int64 value FLOAT
					{
						TMap<int64, float> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							float ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetFloat(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_BOOL: // key int64 value BOOL
					{
						TMap<int64, bool> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							bool ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetBool(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_ENUM: // key int64 value ENUM
					{
						TMap<int64, uint8> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint8 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetEnumValue(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_STRING: // key int64 value STRING
					{
						TMap<int64, FString> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							string ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, ValueFieldDescriptor);
							// 获得value字段的值

							TargetMap.Add(KeyFieldValue, FString(ValueFieldMessageInstance.c_str()));
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_MESSAGE: // key int64 value MESSAGE
					{
						TMap<int64, UObject*> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							int64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							const Message& ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetMessage(*SingleMapMessageInstance, ValueFieldDescriptor);

							UObject* ValueFieldObject = GetUObjectFromMessage(&ValueFieldMessageInstance); // 获得value字段的值										
							UE_LOG(LogTemp, Warning, TEXT("KeyFieldDescriptor KeyFieldValue : %d"), KeyFieldValue);
							ULittleMessage* SpecObject = Cast<ULittleMessage>(ValueFieldObject);
							UE_LOG(LogTemp, Warning, TEXT("ValueFieldDescriptor SpecObject->LitInt : %d"), SpecObject->LitInt);
							UE_LOG(LogTemp, Warning, TEXT("MapHelper.GetMaxIndex : %d"), MapHelper.GetMaxIndex());
							TargetMap.Add(KeyFieldValue, DuplicateObject(ValueFieldObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				default: ;
				}
#pragma endregion
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT32: // key uint32
			{
#pragma region key uint32
				switch (TargetFieldDescriptor->message_type()->FindFieldByName("value")->cpp_type())
				{
				case FieldDescriptor::CPPTYPE_INT32: // key uint32 value int32
					{
						TMap<uint32, int32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_INT64: // key uint32 value int64
					{
						TMap<uint32, int64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT32: // key uint32 value UINT32
					{
						TMap<uint32, uint32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT64: // key uint32 value UINT64
					{
						TMap<uint32, uint64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_DOUBLE: // key uint32 value DOUBLE
					{
						TMap<uint32, double> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							double ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetDouble(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_FLOAT: // key uint32 value FLOAT
					{
						TMap<uint32, float> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							float ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetFloat(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_BOOL: // key uint32 value BOOL
					{
						TMap<uint32, bool> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							bool ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetBool(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_ENUM: // key uint32 value ENUM
					{
						TMap<uint32, uint8> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint8 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetEnumValue(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_STRING: // key uint32 value STRING
					{
						TMap<uint32, FString> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							string ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, ValueFieldDescriptor);
							// 获得value字段的值

							TargetMap.Add(KeyFieldValue, FString(ValueFieldMessageInstance.c_str()));
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_MESSAGE: // key uint32 value MESSAGE
					{
						TMap<uint32, UObject*> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint32 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							const Message& ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetMessage(*SingleMapMessageInstance, ValueFieldDescriptor);

							UObject* ValueFieldObject = GetUObjectFromMessage(&ValueFieldMessageInstance); // 获得value字段的值										
							UE_LOG(LogTemp, Warning, TEXT("KeyFieldDescriptor KeyFieldValue : %d"), KeyFieldValue);
							ULittleMessage* SpecObject = Cast<ULittleMessage>(ValueFieldObject);
							UE_LOG(LogTemp, Warning, TEXT("ValueFieldDescriptor SpecObject->LitInt : %d"), SpecObject->LitInt);
							UE_LOG(LogTemp, Warning, TEXT("MapHelper.GetMaxIndex : %d"), MapHelper.GetMaxIndex());
							TargetMap.Add(KeyFieldValue, DuplicateObject(ValueFieldObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				default: ;
				}
#pragma endregion
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT64: // key uint64
			{
#pragma region key uint64
				switch (TargetFieldDescriptor->message_type()->FindFieldByName("value")->cpp_type())
				{
				case FieldDescriptor::CPPTYPE_INT32: // key uint64 value int32
					{
						TMap<uint64, int32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_INT64: // key uint64 value int64
					{
						TMap<uint64, int64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT32: // key uint64 value UINT32
					{
						TMap<uint64, uint32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT64: // key uint64 value UINT64
					{
						TMap<uint64, uint64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_DOUBLE: // key uint64 value DOUBLE
					{
						TMap<uint64, double> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							double ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetDouble(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_FLOAT: // key uint64 value FLOAT
					{
						TMap<uint64, float> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							float ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetFloat(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_BOOL: // key uint64 value BOOL
					{
						TMap<uint64, bool> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							bool ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetBool(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_ENUM: // key uint64 value ENUM
					{
						TMap<uint64, uint8> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint8 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetEnumValue(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(KeyFieldValue, ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_STRING: // key uint64 value STRING
					{
						TMap<uint64, FString> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							string ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, ValueFieldDescriptor);
							// 获得value字段的值

							TargetMap.Add(KeyFieldValue, FString(ValueFieldMessageInstance.c_str()));
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_MESSAGE: // key uint64 value MESSAGE
					{
						TMap<uint64, UObject*> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							uint64 KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							const Message& ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetMessage(*SingleMapMessageInstance, ValueFieldDescriptor);

							UObject* ValueFieldObject = GetUObjectFromMessage(&ValueFieldMessageInstance); // 获得value字段的值										
							UE_LOG(LogTemp, Warning, TEXT("KeyFieldDescriptor KeyFieldValue : %d"), KeyFieldValue);
							ULittleMessage* SpecObject = Cast<ULittleMessage>(ValueFieldObject);
							UE_LOG(LogTemp, Warning, TEXT("ValueFieldDescriptor SpecObject->LitInt : %d"), SpecObject->LitInt);
							UE_LOG(LogTemp, Warning, TEXT("MapHelper.GetMaxIndex : %d"), MapHelper.GetMaxIndex());
							TargetMap.Add(KeyFieldValue, DuplicateObject(ValueFieldObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				default: ;
				}
#pragma endregion
			}
			break;
		case FieldDescriptor::CPPTYPE_STRING: // key string
			{
#pragma region key string
				switch (TargetFieldDescriptor->message_type()->FindFieldByName("value")->cpp_type())
				{
				case FieldDescriptor::CPPTYPE_INT32: // key string value int32
					{
						TMap<FString, int32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_INT64: // key string value int64
					{
						TMap<FString, int64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							int64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT32: // key string value UINT32
					{
						TMap<FString, uint32> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint32 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt32(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_UINT64: // key string value UINT64
					{
						TMap<FString, uint64> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint64 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetUInt64(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_DOUBLE: // key string value DOUBLE
					{
						TMap<FString, double> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							double ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetDouble(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_FLOAT: // key string value FLOAT
					{
						TMap<FString, float> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							float ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetFloat(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_BOOL: // key string value BOOL
					{
						TMap<FString, bool> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							bool ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetBool(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_ENUM: // key string value ENUM
					{
						TMap<FString, uint8> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							uint8 ValueFieldValue = SingleMapMessageInstance->GetReflection()->GetEnumValue(*SingleMapMessageInstance, ValueFieldDescriptor); // 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), ValueFieldValue);
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_STRING: // key string value STRING
					{
						TMap<FString, FString> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							string ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, ValueFieldDescriptor);
							// 获得value字段的值

							TargetMap.Add(FString(KeyFieldValue.c_str()), FString(ValueFieldMessageInstance.c_str()));
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				case FieldDescriptor::CPPTYPE_MESSAGE: // key string value MESSAGE
					{
						TMap<FString, UObject*> TargetMap;
						for (int32 j = 0; j < MapCount; ++j)
						{
							// 1.从message中获取这个字段对应的message map数据
							const Message* SingleMapMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得map中单个数据，即一个message	
							const FieldDescriptor* KeyFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("key"); // 获得单个数据中的key字段

							const FieldDescriptor* ValueFieldDescriptor = SingleMapMessageInstance->GetDescriptor()->FindFieldByName("value"); // 获得单个数据中的value字段
							string KeyFieldValue = SingleMapMessageInstance->GetReflection()->GetString(*SingleMapMessageInstance, KeyFieldDescriptor); // 获得key字段的值
							const Message& ValueFieldMessageInstance = SingleMapMessageInstance->GetReflection()->GetMessage(*SingleMapMessageInstance, ValueFieldDescriptor);

							UObject* ValueFieldObject = GetUObjectFromMessage(&ValueFieldMessageInstance); // 获得value字段的值										
							ULittleMessage* SpecObject = Cast<ULittleMessage>(ValueFieldObject);
							UE_LOG(LogTemp, Warning, TEXT("ValueFieldDescriptor SpecObject->LitInt : %d"), SpecObject->LitInt);
							UE_LOG(LogTemp, Warning, TEXT("MapHelper.GetMaxIndex : %d"), MapHelper.GetMaxIndex());
							TargetMap.Add(FString(KeyFieldValue.c_str()), DuplicateObject(ValueFieldObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
						}
						// 2.将KeyFieldValue和ValueFieldObject的值赋值给uobject
						MapHelper.MoveAssign(&TargetMap);
					}
					break;
				default: ;
				}
#pragma endregion
			}
			break;
		default: ;
		}
#pragma endregion
	}
	else if (TargetFieldDescriptor->is_repeated()) // repeated
	{
		int32 RepeatedCount = BaseReflection->FieldSize(*TargetMessage, TargetFieldDescriptor); // 获得这个repeated的个数
		FArrayProperty* TargetArrayProperty = Cast<FArrayProperty>(TargetProperty);
		void* ArrayValuePtr = TargetArrayProperty->ContainerPtrToValuePtr<void>(ResultObject);
		FScriptArrayHelper ArrayHelper(TargetArrayProperty, ArrayValuePtr);
		switch (FieldCppType)
		{
		case FieldDescriptor::CPPTYPE_INT32:
			{
				TArray<int32> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					int32 SingleRepeatedInt32 = BaseReflection->GetRepeatedInt32(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个int32
					TargetArray.Add(SingleRepeatedInt32);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_INT64:
			{
				TArray<int64> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					int64 SingleRepeatedInt64 = BaseReflection->GetRepeatedInt64(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个int64
					TargetArray.Add(SingleRepeatedInt64);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT32:
			{
				TArray<uint32> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					uint32 SingleRepeatedUint32 = BaseReflection->GetRepeatedUInt32(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个uint32
					TargetArray.Add(SingleRepeatedUint32);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT64:
			{
				TArray<uint64> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					uint64 SingleRepeatedUint64 = BaseReflection->GetRepeatedUInt64(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个uint64
					TargetArray.Add(SingleRepeatedUint64);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_DOUBLE:
			{
				TArray<double> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					double SingleRepeatedDouble = BaseReflection->GetRepeatedDouble(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个double
					TargetArray.Add(SingleRepeatedDouble);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_FLOAT:
			{
				TArray<float> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					float SingleRepeatedFloat = BaseReflection->GetRepeatedFloat(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个float
					TargetArray.Add(SingleRepeatedFloat);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_BOOL:
			{
				TArray<bool> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					bool SingleRepeatedBool = BaseReflection->GetRepeatedBool(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个bool
					TargetArray.Add(SingleRepeatedBool);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_ENUM:
			{
				TArray<uint8> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					uint8 SingleEnumValue = BaseReflection->GetRepeatedEnumValue(*TargetMessage, TargetFieldDescriptor, j);
					TargetArray.Add(SingleEnumValue);
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_STRING:
			{
				TArray<FString> TargetArray;
				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					string SingleRepeatedString = BaseReflection->GetRepeatedString(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个int32
					TargetArray.Add(FString(SingleRepeatedString.c_str()));
				}
				// 2.赋值
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		case FieldDescriptor::CPPTYPE_MESSAGE:
			{
				TArray<UObject*> TargetArray;

				for (int32 j = 0; j < RepeatedCount; ++j)
				{
					// 1.从message中获取这个字段对应的message repeated数据
					const Message* SingleRepeatedMessageInstance = &BaseReflection->GetRepeatedMessage(*TargetMessage, TargetFieldDescriptor, j); // 获得repeated中单个数据，即一个message
					UObject* SingleRepeatedObject = GetUObjectFromMessage(SingleRepeatedMessageInstance); // 获得message对应的uobject数据
					TargetArray.Add(DuplicateObject(SingleRepeatedObject, nullptr)); // 需要用DuplicateObject进行深拷贝，不然TArray里面的元素全是指向最后一个元素。
				}
				// 2.添加数据到uobject中
				ArrayHelper.MoveAssign(&TargetArray);
			}
			break;
		default: ;
		}
	}
	else // singular
	{
		switch (FieldCppType)
		{
		case FieldDescriptor::CPPTYPE_INT32:
			{
				// 获取值
				int32 ResultInt32 = BaseReflection->GetInt32(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				const FIntProperty* Int32Property = Cast<FIntProperty>(TargetProperty);
				Int32Property->SetPropertyValue_InContainer(ResultObject, ResultInt32, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_INT64:
			{
				// 获取值
				int64 ResultInt64 = BaseReflection->GetInt64(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FInt64Property* Int64Property = Cast<FInt64Property>(TargetProperty);
				Int64Property->SetPropertyValue_InContainer(ResultObject, ResultInt64, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT32:
			{
				// 获取值
				uint32 ResultUint32 = BaseReflection->GetUInt32(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FUInt32Property* Uint32Property = Cast<FUInt32Property>(TargetProperty);
				Uint32Property->SetPropertyValue_InContainer(ResultObject, ResultUint32, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_UINT64:
			{
				// 获取值
				uint64 ResultUint64 = BaseReflection->GetUInt64(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FUInt64Property* Uint64Property = Cast<FUInt64Property>(TargetProperty);
				Uint64Property->SetPropertyValue_InContainer(ResultObject, ResultUint64, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_DOUBLE:
			{
				// 获取值
				double ResultDouble = BaseReflection->GetDouble(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FDoubleProperty* DoubleProperty = Cast<FDoubleProperty>(TargetProperty);
				DoubleProperty->SetPropertyValue_InContainer(ResultObject, ResultDouble, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_FLOAT:
			{
				// 获取值
				float ResultFloat = BaseReflection->GetFloat(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FFloatProperty* FloatProperty = Cast<FFloatProperty>(TargetProperty);
				FloatProperty->SetPropertyValue_InContainer(ResultObject, ResultFloat, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_BOOL:
			{
				// 获取值
				bool ResultBool = BaseReflection->GetBool(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FBoolProperty* BoolProperty = Cast<FBoolProperty>(TargetProperty);
				BoolProperty->SetPropertyValue_InContainer(ResultObject, ResultBool, 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_ENUM:
			{
				// 获取值
				const EnumValueDescriptor* TargetEnumValueDescriptor = BaseReflection->GetEnum(*TargetMessage, TargetFieldDescriptor);
				UE_LOG(LogTemp, Warning, TEXT("TargetEnumValueDescriptor->number : %d"), TargetEnumValueDescriptor->number());
				// 设置值
				FByteProperty* ByteProperty = Cast<FByteProperty>(TargetProperty);
				ByteProperty->SetPropertyValue_InContainer(ResultObject, TargetEnumValueDescriptor->number(), 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_STRING:
			{
				// 获取值
				string ResultString = BaseReflection->GetString(*TargetMessage, TargetFieldDescriptor);
				// 设置值
				FStrProperty* StrProperty = Cast<FStrProperty>(TargetProperty);
				StrProperty->SetPropertyValue_InContainer(ResultObject, FString(ResultString.c_str()), 0);
			}
			break;
		case FieldDescriptor::CPPTYPE_MESSAGE:
			{
				// 获取这个字段对应的message数据
				const Message& SubMessage = BaseReflection->GetMessage(*TargetMessage, TargetFieldDescriptor);
				// 把message作为参数，递归调用本函数
				UObject* SubObject = GetUObjectFromMessage(&SubMessage);
				// 将得到的uobject赋值给对应的属性字段
				FObjectProperty* TargetObjectProperty = Cast<FObjectProperty>(TargetProperty);
				TargetObjectProperty->SetPropertyValue_InContainer(ResultObject, SubObject, 0);
			}
			break;
		default: ;
		}
	}
}

Message* ProtoReflection::SetUStructData(UScriptStruct* OutTargetStruct, const void* InStructValue)
{
	// 提取协议类型名字
	const FString& MessageTypeName = OutTargetStruct->GetMetaData(TEXT("ProtoMessageName"));
	if (MessageTypeName.Len() <= 0) // 没有添加协议标签，无法进行
	{
		return nullptr;
	}
	
	// 现在可以从编译器中提取类型的描述信息.
	UE_LOG(LogTemp, Warning, TEXT("SetUStructData MessageTypeName : %s"), *MessageTypeName);
	const Descriptor* BaseDescriptor = ImporterInstance->pool()->FindMessageTypeByName(std::string(TCHAR_TO_UTF8(*MessageTypeName)));
	if (BaseDescriptor == nullptr)
	{
		UE_LOG(LogTemp, Warning, TEXT("SetUStructData BaseDescriptor == nullptr!!!!!!!!!!!"));
		return nullptr;
	}
	
	UE_LOG(LogTemp, Warning, TEXT("SetUStructData BaseDescriptor->full_name: %s"), *FString(BaseDescriptor->full_name().c_str()));
	UE_LOG(LogTemp, Warning, TEXT("SetUStructData descriptor->name : %s"), *FString(BaseDescriptor->name().c_str()));
	
	const Message* MessagePrototype = DynamicMessageFactoryInstance.GetPrototype(BaseDescriptor);
	Message* MessageInstance = MessagePrototype->New();
	const Reflection* ReflectionInstance = MessageInstance->GetReflection();
	
	for (TFieldIterator<FProperty> It(OutTargetStruct); It; ++It)
	{
		FProperty* SubStructProperty = *It;

		if (FIntProperty* SubStructInt32 = CastField<FIntProperty>(SubStructProperty))
		{
			int32 IntValue = SubStructInt32->GetPropertyValue_InContainer(InStructValue, 0);
			UE_LOG(LogTemp, Warning, TEXT("SubStructInt32: %d"), IntValue);
		}
	}

	return MessageInstance;
}

```

