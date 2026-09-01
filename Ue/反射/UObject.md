### 类型
- 通过给类、枚举、属性、函数加上特定的宏来标记更多的元数据
- 在有必要的时候这些标记宏甚至也可以安插进生成的代码来合成编译

![[注册 2026-08-31 10.07.10.excalidraw]]

### 代码生成

将宏转化为对应的类型结构记录反射信息，然后定义两个静态类型自动收集信息到一个全局的变量。
![[Pasted image 20260831111619.jpg]]
![[Pasted image 20260831111910.jpg]]
![[Pasted image 20260831112047.jpg]]
![[Pasted image 20260831112135.jpg]]

![[Pasted image 20260831142802.jpg]]

```cpp
static void DequeuePendingAutoRegistrants(TArray<FPendingRegistrant>& OutPendingRegistrants)
{
    FPendingRegistrant* NextPendingRegistrant = GFirstPendingRegistrant;
    GFirstPendingRegistrant = NULL;
    GLastPendingRegistrant = NULL;
    while (NextPendingRegistrant)
    {
        FPendingRegistrant* PendingRegistrant = NextPendingRegistrant;
        OutPendingRegistrants.Add(*PendingRegistrant);
        NextPendingRegistrant = PendingRegistrant->NextAutoRegister;
        delete PendingRegistrant;
    };
}

static void UObjectProcessRegistrants()
{
    TArray<FPendingRegistrant> PendingRegistrants;
    DequeuePendingAutoRegistrants(PendingRegistrants);  //从链表中提取注册项列表

    for(int32 RegistrantIndex = 0;RegistrantIndex < PendingRegistrants.Num();++RegistrantIndex)
    {
        const FPendingRegistrant& PendingRegistrant = PendingRegistrants[RegistrantIndex];
        UObjectForceRegistration(PendingRegistrant.Object); //真正的注册
        DequeuePendingAutoRegistrants(PendingRegistrants);  //继续尝试提取
    }
}

void UObjectForceRegistration(UObjectBase* Object)
{
    TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();//得到对象的注册信息
    FPendingRegistrantInfo* Info = PendingRegistrants.Find(Object);
    if (Info)   //有可能为空，因为之前已经被注册过了
    {
        const TCHAR* PackageName = Info->PackageName;//对象所在的Package
        const TCHAR* Name = Info->Name; //对象名字
        PendingRegistrants.Remove(Object);//删除
        Object->DeferredRegister(UClass::StaticClass(),PackageName,Name);//延迟注册
    }
}

void UObjectBase::DeferredRegister(UClass *UClassStaticClass,const TCHAR* PackageName,const TCHAR* InName)
{
    // Set object properties.
    UPackage* Package = CreatePackage(nullptr, PackageName);    //创建属于的Package
    Package->SetPackageFlags(PKG_CompiledIn);
    OuterPrivate = Package; //设定Outer到该Package

    ClassPrivate = UClassStaticClass;   //设定属于的UClass*类型

    // Add to the global object table.
    AddObject(FName(InName), EInternalObjectFlags::None);   //注册该对象的名字
}
void UObjectBase::AddObject(FName InName, EInternalObjectFlags InSetInternalFlags)
{
    NamePrivate = InName;   //设定对象的名字
    //...
    //AllocateUObjectIndexForCurrentThread(this);
    //HashObject(this);
}
```

```cpp
void ProcessNewlyLoadedUObjects()
{
    UClassRegisterAllCompiledInClasses();   //为代码里定义的那些类生成UClass*
    //提取收集到的注册项信息
    const TArray<UClass* (*)()>& DeferredCompiledInRegistration=GetDeferredCompiledInRegistration();
    const TArray<FPendingStructRegistrant>& DeferredCompiledInStructRegistration=GetDeferredCompiledInStructRegistration();
    const TArray<FPendingEnumRegistrant>& DeferredCompiledInEnumRegistration=GetDeferredCompiledInEnumRegistration();
    //有待注册项就继续循环注册
    bool bNewUObjects = false;
    while (GFirstPendingRegistrant || 
    DeferredCompiledInRegistration.Num() || 
    DeferredCompiledInStructRegistration.Num() || 
    DeferredCompiledInEnumRegistration.Num())
    {
        bNewUObjects = true;
        UObjectProcessRegistrants();    //注册UClass*
        UObjectLoadAllCompiledInStructs();  //为代码里的枚举和结构构造类型对象
        UObjectLoadAllCompiledInDefaultProperties();    //为代码里的类继续构造UClass对象
    }

    if (bNewUObjects && !GIsInitialLoad)
    {
        UClass::AssembleReferenceTokenStreams();    //构造引用记号流，为后续GC用
    }
}
```

这里先构造enum在构造struct的原因是：struct中可以包含enum因此在构造struct的时候可以通过enum名字查找到相应的`UEnum*`对象。
```cpp
static void UObjectLoadAllCompiledInStructs()
{
    TArray<FPendingEnumRegistrant> PendingEnumRegistrants = MoveTemp(GetDeferredCompiledInEnumRegistration());
    for (const FPendingEnumRegistrant& EnumRegistrant : PendingEnumRegistrants)
    {
        CreatePackage(nullptr, EnumRegistrant.PackageName); //创建其所属于的Package
    }

    TArray<FPendingStructRegistrant> PendingStructRegistrants = MoveTemp(GetDeferredCompiledInStructRegistration());
    for (const FPendingStructRegistrant& StructRegistrant : PendingStructRegistrants)
    {
        CreatePackage(nullptr, StructRegistrant.PackageName);   //创建其所属于的Package
    }

    for (const FPendingEnumRegistrant& EnumRegistrant : PendingEnumRegistrants)
    {
        EnumRegistrant.RegisterFn();    //调用生成代码里Z_Construct_UEnum_Hello_EMyEnum
    }
    for (const FPendingStructRegistrant& StructRegistrant : PendingStructRegistrants)
    {
        StructRegistrant.RegisterFn(); //调用生成代码里Z_Construct_UScriptStruct_FMyStruct
    }
}
```

```cpp
static void UObjectLoadAllCompiledInDefaultProperties()
{
    static FName LongEnginePackageName(TEXT("/Script/Engine")); //引擎包的名字
    if(GetDeferredCompiledInRegistration().Num() <= 0) return;
    TArray<UClass*> NewClassesInCoreUObject;
    TArray<UClass*> NewClassesInEngine;
    TArray<UClass*> NewClasses;
    TArray<UClass* (*)()> PendingRegistrants = MoveTemp(GetDeferredCompiledInRegistration());
    for (UClass* (*Registrant)() : PendingRegistrants) 
    {
        UClass* Class = Registrant();//调用生成代码里的Z_Construct_UClass_UMyClass创建UClass*
        //按照所属于的Package分到3个数组里
        if (Class->GetOutermost()->GetFName() == GLongCoreUObjectPackageName)
        {
            NewClassesInCoreUObject.Add(Class);
        }
        else if (Class->GetOutermost()->GetFName() == LongEnginePackageName)
        {
            NewClassesInEngine.Add(Class);
        }
        else
        {
            NewClasses.Add(Class);
        }
    }
    //分别构造CDO对象
    for (UClass* Class : NewClassesInCoreUObject)   { Class->GetDefaultObject(); }
    for (UClass* Class : NewClassesInEngine)        { Class->GetDefaultObject(); }
    for (UClass* Class : NewClasses)                { Class->GetDefaultObject(); }
}
```

![[Pasted image 20260831143625.jpg]]

#### UEnum
```cpp
static UEnum* MyEnum_StaticEnum()   //RegisterFn指向该函数
{
    static UEnum* Singleton = nullptr;
    if (!Singleton)
    {
        Singleton = GetStaticEnum(Z_Construct_UEnum_Hello_MyEnum, Z_Construct_UPackage__Script_Hello(), TEXT("MyEnum"));
    }
    return Singleton;
}

static FCompiledInDeferEnum Z_CompiledInDeferEnum_UEnum_MyEnum(MyEnum_StaticEnum, TEXT("/Script/Hello"), TEXT("MyEnum"), false, nullptr, nullptr);    //收集点

UEnum* Z_Construct_UEnum_Hello_MyEnum() //实际调用点
{
    static UEnum* ReturnEnum = nullptr;
    if (!ReturnEnum)
    {
        static const UE4CodeGen_Private::FEnumeratorParam Enumerators[] = {
            { "MyEnum::Dance", (int64)MyEnum::Dance }, //注意枚举项的名字是以"枚举::"开头的
            { "MyEnum::Rain", (int64)MyEnum::Rain },
            { "MyEnum::Song", (int64)MyEnum::Song },
        };

        static const UE4CodeGen_Private::FEnumParams EnumParams = {
            (UObject*(*)())Z_Construct_UPackage__Script_Hello,
            UE4CodeGen_Private::EDynamicType::NotDynamic,
            "MyEnum",
            RF_Public|RF_Transient|RF_MarkAsNative,
            GetMyEnumDisplayName,//一般为nullptr，我们自定义了一个，所以这里才有
            (uint8)UEnum::ECppForm::EnumClass,
            "MyEnum",
            Enumerators,
            ARRAY_COUNT(Enumerators),
            METADATA_PARAMS(Enum_MetaDataParams, ARRAY_COUNT(Enum_MetaDataParams))
        };
        UE4CodeGen_Private::ConstructUEnum(ReturnEnum, EnumParams);//最终生成点
    }
    return ReturnEnum;
}

void ConstructUEnum(UEnum*& OutEnum, const FEnumParams& Params)
{
    UObject* (*OuterFunc)() = Params.OuterFunc;
    UObject* Outer = OuterFunc ? OuterFunc() : nullptr; //先确保创建Outer
    if (OutEnum) {return;}  //防止重复构造

    UEnum* NewEnum = new (EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags) UEnum(FObjectInitializer());    //创建一个UEnum
    OutEnum = NewEnum;

    //生成枚举名字值对数组
    TArray<TPair<FName, int64>> EnumNames;
    EnumNames.Reserve(Params.NumEnumerators);
    for (const FEnumeratorParam* Enumerator = Params.EnumeratorParams, *EnumeratorEnd = Enumerator + Params.NumEnumerators; Enumerator != EnumeratorEnd; ++Enumerator)
    {
        EnumNames.Emplace(UTF8_TO_TCHAR(Enumerator->NameUTF8), Enumerator->Value);
    }
    //设置枚举项数组
    NewEnum->SetEnums(EnumNames, (UEnum::ECppForm)Params.CppForm, Params.DynamicType == EDynamicType::NotDynamic);
    NewEnum->CppType = UTF8_TO_TCHAR(Params.CppTypeUTF8);  //cpp名字

    if (Params.DisplayNameFunc)
    {
        NewEnum->SetEnumDisplayNameFn(Params.DisplayNameFunc);  //设置自定义显示名字回调
    }
}

bool UEnum::SetEnums(TArray<TPair<FName, int64>>& InNames, UEnum::ECppForm InCppForm, bool bAddMaxKeyIfMissing)
{
    if (Names.Num() > 0)
    {
        RemoveNamesFromMasterList();   //去除之前的名字
    }
    Names   = InNames;
    CppForm = InCppForm;
    if (bAddMaxKeyIfMissing)
    {
        if (!ContainsExistingMax())
        {
            FName MaxEnumItem = *GenerateFullEnumName(*(GenerateEnumPrefix() + TEXT("_MAX")));
            if (LookupEnumName(MaxEnumItem) != INDEX_NONE)
            {
                // the MAX identifier is already being used by another enum
                return false;
            }
            Names.Emplace(MaxEnumItem, GetMaxEnumValue() + 1);
        }
    }
    AddNamesToMasterList();
    return true;
}
```

#### UScriptStruct
```cpp
class UScriptStruct* FMyStruct::StaticStruct()  //RegisterFn指向该函数
{
    static class UScriptStruct* Singleton = NULL;
    if (!Singleton)
    {
        Singleton = GetStaticStruct(Z_Construct_UScriptStruct_FMyStruct, Z_Construct_UPackage__Script_Hello(), TEXT("MyStruct"), sizeof(FMyStruct), Get_Z_Construct_UScriptStruct_FMyStruct_CRC());
    }
    return Singleton;
}

static FCompiledInDeferStruct Z_CompiledInDeferStruct_UScriptStruct_FMyStruct(FMyStruct::StaticStruct, TEXT("/Script/Hello"), TEXT("MyStruct"), false, nullptr, nullptr);     //收集点

static struct FScriptStruct_Hello_StaticRegisterNativesFMyStruct
{
    FScriptStruct_Hello_StaticRegisterNativesFMyStruct()
    {
        UScriptStruct::DeferCppStructOps(FName(TEXT("MyStruct")),new UScriptStruct::TCppStructOps<FMyStruct>);
    }
} ScriptStruct_Hello_StaticRegisterNativesFMyStruct; //收集点

struct Z_Construct_UScriptStruct_FMyStruct_Statics
{
    static void* NewStructOps() //创建结构操作辅助类
    {
        return (UScriptStruct::ICppStructOps*)new UScriptStruct::TCppStructOps<FMyStruct>();
    }
    //属性参数...
    //结构参数
    static const UE4CodeGen_Private::FStructParams ReturnStructParams= 
    {
        (UObject* (*)())Z_Construct_UPackage__Script_Hello,//Outer
        nullptr,    //构造基类的函数指针
        &NewStructOps,//构造结构操作类的函数指针
        "MyStruct",//结构名字
        RF_Public|RF_Transient|RF_MarkAsNative, //对象标记
        EStructFlags(0x00000201),   //结构标记
        sizeof(FMyStruct),//结构大小
        alignof(FMyStruct), //结构内存对齐
        PropPointers, ARRAY_COUNT(PropPointers) //属性列表
    };
};

UScriptStruct* Z_Construct_UScriptStruct_FMyStruct() //真正的构造实现
{
    static UScriptStruct* ReturnStruct = nullptr;
    if (!ReturnStruct)
    {
        UE4CodeGen_Private::ConstructUScriptStruct(ReturnStruct, Z_Construct_UScriptStruct_FMyStruct_Statics::ReturnStructParams);
    }
    return ReturnStruct;
}

void ConstructUScriptStruct(UScriptStruct*& OutStruct, const FStructParams& Params)
{
    UObject* Outer = Params.OuterFunc ? Params.OuterFunc() : nullptr;//构造Outer
    UScriptStruct* Super = Params.SuperFunc ? Params.SuperFunc() : nullptr;//构造SuperStruct
    UScriptStruct::ICppStructOps* StructOps = Params.StructOpsFunc ? Params.StructOpsFunc() : nullptr;//构造结构操作类

    if (OutStruct) {return;}
    //构造UScriptStruct
    UScriptStruct* NewStruct = new(EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Params.NameUTF8), Params.ObjectFlags) UScriptStruct(FObjectInitializer(), Super, StructOps, (EStructFlags)Params.StructFlags, Params.SizeOf, Params.AlignOf);
    OutStruct = NewStruct;
    //构造属性集合
    ConstructUProperties(NewStruct, Params.PropertyArray, Params.NumProperties);
    //链接
    NewStruct->StaticLink();
}
```

#### UClass
```cpp
//函数参数...
struct Z_Construct_UClass_UMyClass_Statics
{
    //依赖项列表
    static UObject* (*const DependentSingletons[])()=
    {
        (UObject* (*)())Z_Construct_UClass_UObject,//依赖基类UObject
        (UObject* (*)())Z_Construct_UPackage__Script_Hello,//依赖所属于的Hello模块
    };
    //属性参数...
    //函数参数...
    //接口
    static const UE4CodeGen_Private::FImplementedInterfaceParams InterfaceParams[]= 
    {
        {
            Z_Construct_UClass_UMyInterface_NoRegister,//构造UMyInterface所属的UClass*函数指针
            (int32)VTABLE_OFFSET(UMyClass, IMyInterface),//多重继承的指针偏移
            false   //是否是在蓝图实现
        }
    };

    static const FCppClassTypeInfoStatic StaticCppClassTypeInfo= {
        TCppClassTypeTraits<UMyClass>::IsAbstract,//c++类信息，是否是虚类
    };

    static const UE4CodeGen_Private::FClassParams ClassParams = 
    {
        &UMyClass::StaticClass,//取出UClass*的函数指针
        DependentSingletons, ARRAY_COUNT(DependentSingletons),//依赖项
        0x001000A0u,//类标志
        FuncInfo, ARRAY_COUNT(FuncInfo),//函数列表
        PropPointers, ARRAY_COUNT(PropPointers),//属性列表
        nullptr,//Config文件名
        &StaticCppClassTypeInfo,//c++类信息
        InterfaceParams, ARRAY_COUNT(InterfaceParams)//接口列表
    };
};

UClass* Z_Construct_UClass_UMyClass()
{
    static UClass* OuterClass = nullptr;
    if (!OuterClass)
    {
        UE4CodeGen_Private::ConstructUClass(OuterClass, Z_Construct_UClass_UMyClass_Statics::ClassParams);
    }
    return OuterClass;
}

IMPLEMENT_CLASS(UMyClass, 4008851639); //收集点
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyClass(Z_Construct_UClass_UMyClass, &UMyClass::StaticClass, TEXT("/Script/Hello"), TEXT("UMyClass"), false, nullptr, nullptr, nullptr); //收集点

void ConstructUClass(UClass*& OutClass, const FClassParams& Params)
{
    if (OutClass && (OutClass->ClassFlags & CLASS_Constructed)) {return;}  //防止重复构造
    for(int i=0;i<Params.NumDependencySingletons;++i)
    {
        Params.DependencySingletonFuncArray[i]();   //构造依赖的对象
    }

    UClass* NewClass = Params.ClassNoRegisterFunc();    //取得先前生成的UClass*，NoRegister是指没有经过DeferRegister
    OutClass = NewClass;

    if (NewClass->ClassFlags & CLASS_Constructed) {return;}//防止重复构造

    UObjectForceRegistration(NewClass); //确保此UClass*已经注册

    NewClass->ClassFlags |= (EClassFlags)(Params.ClassFlags | CLASS_Constructed);//标记已经构造

    if ((NewClass->ClassFlags & CLASS_Intrinsic) != CLASS_Intrinsic)
    {
        check((NewClass->ClassFlags & CLASS_TokenStreamAssembled) != CLASS_TokenStreamAssembled);
        NewClass->ReferenceTokenStream.Empty();//对于蓝图类需要重新生成一下引用记号流
    }
    //构造函数列表
    NewClass->CreateLinkAndAddChildFunctionsToMap(Params.FunctionLinkArray, Params.NumFunctions);
    //构造属性列表
    ConstructUProperties(NewClass, Params.PropertyArray, Params.NumProperties);

    if (Params.ClassConfigNameUTF8)
    {   //配置文件名
        NewClass->ClassConfigName = FName(UTF8_TO_TCHAR(Params.ClassConfigNameUTF8));
    }

    NewClass->SetCppTypeInfoStatic(Params.CppClassInfo);//C++类型信息

    if (Params.NumImplementedInterfaces)
    {
        NewClass->Interfaces.Reserve(Params.NumImplementedInterfaces);
        for(int i=0;i<Params.Params.NumImplementedInterfaces;++i)
        {
            const auto& ImplementedInterface = Params.ImplementedInterfaceArray[i];
            UClass* (*ClassFunc)() = ImplementedInterface.ClassFunc;
            UClass* InterfaceClass = ClassFunc ? ClassFunc() : nullptr;//取得UMyInterface所属于的UClass*对象

            NewClass->Interfaces.Emplace(InterfaceClass, ImplementedInterface.Offset, ImplementedInterface.bImplementedByK2);//添加实现的接口
        }
    }

    NewClass->StaticLink();//链接
}
```

#### UProperty
```cpp
//一个个属性的参数
static const UE4CodeGen_Private::FFloatPropertyParams NewProp_Score = 
{ 
    UE4CodeGen_Private::EPropertyClass::Float, 
    "Score", 
    RF_Public|RF_Transient|RF_MarkAsNative,//对象标记
    (EPropertyFlags)0x0010000000000004, //属性标记
    1,  //数组维度，固定的数组的大小
    nullptr, //RepNotify函数的名称
    STRUCT_OFFSET(FMyStruct, Score) //属性的结构偏移地址
};
 //结构参数数组，会发送给ConstructUProperties来构造。
static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[] = 
{
    &NewProp_Score
};

void ConstructUProperty(UObject* Outer, const FPropertyParamsBase* const*& PropertyArray, int32& NumProperties)
{
    const FPropertyParamsBase* PropBase = *PropertyArray++;
    uint32 ReadMore = 0;
    UProperty* NewProp = nullptr;
    switch (PropBase->Type)
    {
       case EPropertyClass::Array:
        {
            const FArrayPropertyParams* Prop = (const FArrayPropertyParams*)PropBase;
            NewProp = new (EC_InternalUseOnlyConstructor, Outer, UTF8_TO_TCHAR(Prop->NameUTF8), Prop->ObjectFlags) UArrayProperty(FObjectInitializer(), EC_CppProperty, Prop->Offset, Prop->PropertyFlags);//构造Property对象

            // Next property is the array inner
            ReadMore = 1;//需要一个子属性
        }
        break;
        //case其他的各种类型属性
    }

    NewProp->ArrayDim = PropBase->ArrayDim;//设定属性维度，单属性为1，int32 prop[10]这种的为10
    if (PropBase->RepNotifyFuncUTF8)
    {   //属性的复制通知函数名
        NewProp->RepNotifyFunc = FName(UTF8_TO_TCHAR(PropBase->RepNotifyFuncUTF8));
    }

    --NumProperties;
    for (; ReadMore; --ReadMore)
    {   //构造子属性，注意这里以现在的属性NewProp为Outer
        ConstructUProperty(NewProp, PropertyArray, NumProperties);
    }
}
```

#### UFunction
```cpp
//测试函数：int32 Func(float param1);
void UMyClass::ImplementableFunc()  //UHT为我们生成了函数实体
{
    ProcessEvent(FindFunctionChecked("ImplementableFunc"),NULL);
}
void UMyClass::NativeFunc() //UHT为我们生成了函数实体，但我们可以自定义_Implementation
{
    ProcessEvent(FindFunctionChecked("NativeFunc"),NULL);
}
void UMyClass::StaticRegisterNativesUMyClass()  //之前的Native函数收集点
{
    UClass* Class = UMyClass::StaticClass();
    static const FNameNativePtrPair Funcs[] = {
        { "Func", &UMyClass::execFunc },
        { "NativeFunc", &UMyClass::execNativeFunc },
    };
    FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, ARRAY_COUNT(Funcs));
}
struct Z_Construct_UFunction_UMyClass_Func_Statics
{
    struct MyClass_eventFunc_Parms  //把所有参数打包成一个结构来存储
    {
        float param1;
        int32 ReturnValue;
    };
    static const UE4CodeGen_Private::FIntPropertyParams NewProp_ReturnValue= 
    { 
        UE4CodeGen_Private::EPropertyClass::Int, 
        "ReturnValue", 
        RF_Public|RF_Transient|RF_MarkAsNative,
        (EPropertyFlags)0x0010000000000580,
        1, 
        nullptr, 
        STRUCT_OFFSET(MyClass_eventFunc_Parms, ReturnValue) 
    };

    static const UE4CodeGen_Private::FFloatPropertyParams NewProp_param1 =
    { 
        UE4CodeGen_Private::EPropertyClass::Float,
        "param1", 
        RF_Public|RF_Transient|RF_MarkAsNative, 
        (EPropertyFlags)0x0010000000000080, 
        1,
        nullptr, 
        STRUCT_OFFSET(MyClass_eventFunc_Parms, param1) 
    };
    //函数的子属性
    static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[]= 
    {
        &NewProp_ReturnValue,   //返回值也用属性表示
        &NewProp_param1,        //参数用属性表示
    };
    //函数的参数
    static const UE4CodeGen_Private::FFunctionParams FuncParams=
    { 
        (UObject*(*)())Z_Construct_UClass_UMyClass, //外部对象
        "Func", //名字
        RF_Public|RF_Transient|RF_MarkAsNative, //对象标记
        nullptr, //父函数，在蓝图中重载基类函数时候指向基类函数版本
        (EFunctionFlags)0x04020401, //函数标记
        sizeof(MyClass_eventFunc_Parms),//属性的结构大小
        PropPointers, ARRAY_COUNT(PropPointers),//属性列表
        0,  //RPCId
        0   //RPCResponseId
    };
};

UFunction* Z_Construct_UFunction_UMyClass_Func()
{
    static UFunction* ReturnFunction = nullptr;
    if (!ReturnFunction)
    {   //构造函数
        UE4CodeGen_Private::ConstructUFunction(ReturnFunction, Z_Construct_UFunction_UMyClass_Func_Statics::FuncParams);
    }
    return ReturnFunction;
}

//其他函数...
static const FClassFunctionLinkInfo FuncInfo[]= //发给ClassParams来构造UClass*
{
    { &Z_Construct_UFunction_UMyClass_Func, "Func" }, // 2606493682
    { &Z_Construct_UFunction_UMyClass_ImplementableFunc, "ImplementableFunc" }, // 3752866266
    { &Z_Construct_UFunction_UMyClass_NativeFunc, "NativeFunc" }, // 3036938731
}; 
//接口函数...
void IMyInterface::Execute_ImplementableInterfaceFunc(UObject* O)
{   //通过名字查找函数
    UFunction* const Func = O->FindFunction("ImplementableInterfaceFunc");
    if (Func)
    {
        O->ProcessEvent(Func, NULL);
    }//找不到，其实不会报错，所以是在尝试调用一个接口函数
}
void IMyInterface::Execute_NativeInterfaceFunc(UObject* O)
{   //通过名字查找函数
    UFunction* const Func = O->FindFunction("NativeInterfaceFunc");
    if (Func)
    {
        O->ProcessEvent(Func, NULL);
    }
    else if (auto I = (IMyInterface*)(O->GetNativeInterfaceAddress(UMyInterface::StaticClass())))
    {   //如果找不到蓝图中的版本，则会尝试调用C++里的_Implementation默认实现。
        I->NativeInterfaceFunc_Implementation();
    }
}
```
