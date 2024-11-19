---
layout: post
title: "需要捕捉Attribute的Source和Target是怎样放到FGameplayEffectSpec中的？"
date: 2024-11-19
categories: [programming, UE]
tags: [FGameplayEffectSpec, AttributeSet]

---



## 需要捕捉Attribute的Source和Target是怎样放到FGameplayEffectSpec中的？

疑问代码：SourceAttackPower是如何获取值的？值存储在哪里？其中Target和Source又是怎么区分？

```c++
struct FYushaDamageCapture
{
	DECLARE_ATTRIBUTE_CAPTUREDEF(AttackPower)

	FYushaDamageCapture()
	{
		DEFINE_ATTRIBUTE_CAPTUREDEF(UYushaAttributeSet, AttackPower, Source, false);
	}
};

static const FYushaDamageCapture& GetDamageCapture()
{
	static FYushaDamageCapture YushaDamageCapture;
	return YushaDamageCapture;
}

UGEExeCalc_DamageTaken::UGEExeCalc_DamageTaken()
{
	RelevantAttributesToCapture.Add(GetDamageCapture().AttackPowerDef);
}

//以上部分相当于：
//UGEExeCalc_DamageTaken::UGEExeCalc_DamageTaken()
//{
    //FProperty* AttackPowerProperty = FindFieldChecked<FProperty>(
    //	UYushaAttributeSet::StaticClass(),
    //	GET_MEMBER_NAME_CHECKED(UYushaAttributeSet,AttackPower)
    //);
    //FGameplayEffectAttributeCaptureDefinition AttackPowerCaptureDefinition(
    //	AttackPowerProperty,
    //	EGameplayEffectAttributeCaptureSource::Source,
    //	false
    //);
    //RelevantAttributesToCapture.Add(AttackPowerCaptureDefinition);
//}


void UGEExeCalc_DamageTaken::Execute_Implementation(const FGameplayEffectCustomExecutionParameters& ExecutionParams, FGameplayEffectCustomExecutionOutput& OutExecutionOutput) const
{
	const FGameplayEffectSpec& EffectSpec = ExecutionParams.GetOwningSpec();
	FAggregatorEvaluateParameters EvaluationParameters;
	EvaluationParameters.SourceTags = EffectSpec.CapturedSourceTags.GetAggregatedTags();
	EvaluationParameters.TargetTags = EffectSpec.CapturedTargetTags.GetAggregatedTags();

	float SourceAttackPower = 0.f;
	ExecutionParams.AttemptCalculateCapturedAttributeMagnitude(GetDamageCapture().AttackPowerDef, EvaluationParameters, SourceAttackPower);
    .......
}
//此处的SourceAttackPower是如何获取值的？值存储在哪里？其中Target和Source又是怎么区分？
```

简答：

创建 FGameplayEffectSpec 的 ASC 的 Owner 作为 Source，而应用 FGameplayEffectSpec 函数 ApplyGameplayEffectSpecToTarget 函数中的Target (也就是 TargetASC ) 的 Owner 作为 Target ，将获取到值都放在 FGameplayEffectSpec 中的 CapturedRelevantAttributes 中。AttemptCalculateCapturedAttributeMagnitude 则会从其中读取。



详细分析：

首先Spec怎么知道需要捕捉哪个属性的？

```c++
UGEExeCalc_DamageTaken::UGEExeCalc_DamageTaken()
{
    //RelevantAttributesToCapture.Add(GetDamageCapture().AttackPowerDef);
    //相当于
    Property* AttackPowerProperty = ......
    FGameplayEffectAttributeCaptureDefinition AttackPowerCaptureDefinition(......);
    RelevantAttributesToCapture.Add(AttackPowerCaptureDefinition);
}

FGameplayEffectAttributeCaptureSpec::FGameplayEffectAttributeCaptureSpec(const FGameplayEffectAttributeCaptureDefinition& InDefinition)
	: BackingDefinition(InDefinition)
{
}
```

UGEExeCalc_DamageTaken::UGEExeCalc_DamageTaken构造函数在构造CDO时已经被调用，也就是模拟Play还没开始，若早已配置，则会在引擎初始化的时候完成。

```c++
static void UObjectLoadAllCompiledInDefaultProperties(TArray<UClass*>& OutAllNewClasses)
```

需要被捕获的值，在游戏引擎加载阶段就已经被加入了。当FGameplayEffectSpec创建时，会设置其 AttributeCaptureDefinitions。

```c++
void FGameplayEffectSpec::Initialize(const UGameplayEffect* InDef, const FGameplayEffectContextHandle& InEffectContext, float InLevel)
{
	......
	// Prep the spec with all of the attribute captures it will need to perform
	SetupAttributeCaptureDefinitions();
    ......
    // Prepare source tags before accessing them in the Components
	CaptureDataFromSource();
    ......
}

void FGameplayEffectSpec::SetupAttributeCaptureDefinitions()
{
	// Add duration if required
	if (Def->DurationPolicy == EGameplayEffectDurationType::HasDuration)
	{
		CapturedRelevantAttributes.AddCaptureDefinition(UAbilitySystemComponent::GetOutgoingDurationCapture());
		CapturedRelevantAttributes.AddCaptureDefinition(UAbilitySystemComponent::GetIncomingDurationCapture());
	}

	TArray<FGameplayEffectAttributeCaptureDefinition> CaptureDefs;

	// Gather capture definitions from duration	
	{
		CaptureDefs.Reset();
		Def->DurationMagnitude.GetAttributeCaptureDefinitions(CaptureDefs);
		for (const FGameplayEffectAttributeCaptureDefinition& CurDurationCaptureDef : CaptureDefs)
		{
			CapturedRelevantAttributes.AddCaptureDefinition(CurDurationCaptureDef);
		}
	}

	// Gather all capture definitions from modifiers
	for (int32 ModIdx = 0; ModIdx < Modifiers.Num(); ++ModIdx)
	{
		const FGameplayModifierInfo& ModDef = Def->Modifiers[ModIdx];
		const FModifierSpec& ModSpec = Modifiers[ModIdx];

		CaptureDefs.Reset();
		ModDef.ModifierMagnitude.GetAttributeCaptureDefinitions(CaptureDefs);
		
		for (const FGameplayEffectAttributeCaptureDefinition& CurCaptureDef : CaptureDefs)
		{
			CapturedRelevantAttributes.AddCaptureDefinition(CurCaptureDef);
		}
	}

	// Gather all capture definitions from executions
	for (const FGameplayEffectExecutionDefinition& Exec : Def->Executions)
	{
		CaptureDefs.Reset();
		Exec.GetAttributeCaptureDefinitions(CaptureDefs);
		for (const FGameplayEffectAttributeCaptureDefinition& CurExecCaptureDef : CaptureDefs)
		{
			CapturedRelevantAttributes.AddCaptureDefinition(CurExecCaptureDef);
		}
	}
}
```

所以创建 FGameplayEffectSpec 时，Source和Target的BackingDefinition就已经设置好了，只剩下AttributeAggregator没有对象 (刚创建好的Spec在Initialize时会调用CaptureDataFromSource();获取 Source Attributes )

![image-20241118183147291](\images\image-20241118183147291.png)

**捕获到的数据最终是存储在 FGameplayEffectSpecHandle 的 CapturedRelevantAttributes 属性中** AttemptCalculateCapturedAttributeMagnitude函数实际就是从 CapturedRelevantAttributes 中获取的对应值



接下来值是如何被捕获的？

ASC中保存有 ActiveGameplayEffects

```c++
UPROPERTY(Replicated)
FActiveGameplayEffectsContainer ActiveGameplayEffects;
```

ActiveGameplayEffects 中有 AttributeAggregatorMap 此中保存有需要捕获的数据

```c++
TMap<FGameplayAttribute, FAggregatorRef>		AttributeAggregatorMap;
```

![image-20241117210158719](\images\image-20241117210158719.png)

不论Target还是Source，捕获Attribute时都是用的以下函数，执行完以下流程后，会将AttributeAggregatorMap中对应的值放到CapturedRelevantAttributes 中：

```c++
void FGameplayEffectAttributeCaptureSpecContainer::CaptureAttributes(UAbilitySystemComponent* InAbilitySystemComponent, EGameplayEffectAttributeCaptureSource InCaptureSource)
{
	if (InAbilitySystemComponent)
	{
		const bool bSourceComponent = (InCaptureSource == EGameplayEffectAttributeCaptureSource::Source);
		TArray<FGameplayEffectAttributeCaptureSpec>& AttributeArray = (bSourceComponent ? SourceAttributes : TargetAttributes);

		// Capture every spec's requirements from the specified component
		for (FGameplayEffectAttributeCaptureSpec& CurCaptureSpec : AttributeArray)
		{
			InAbilitySystemComponent->CaptureAttributeForGameplayEffect(CurCaptureSpec);
		}
	}
}
↓↓↓
void UAbilitySystemComponent::CaptureAttributeForGameplayEffect(OUT FGameplayEffectAttributeCaptureSpec& OutCaptureSpec)
{
	// Verify the capture is happening on an attribute the component actually has a set for; if not, can't capture the value
	const FGameplayAttribute& AttributeToCapture = OutCaptureSpec.BackingDefinition.AttributeToCapture;
	if (AttributeToCapture.IsValid() && (AttributeToCapture.IsSystemAttribute() || GetAttributeSubobject(AttributeToCapture.GetAttributeSetClass())))
	{
		ActiveGameplayEffects.CaptureAttributeForGameplayEffect(OutCaptureSpec);
	}
}
↓↓↓
void FActiveGameplayEffectsContainer::CaptureAttributeForGameplayEffect(OUT FGameplayEffectAttributeCaptureSpec& OutCaptureSpec)
{
	FAggregatorRef& AttributeAggregator = FindOrCreateAttributeAggregator(OutCaptureSpec.BackingDefinition.AttributeToCapture);
	
	if (OutCaptureSpec.BackingDefinition.bSnapshot)
	{
		OutCaptureSpec.AttributeAggregator.TakeSnapshotOf(AttributeAggregator);
	}
	else
	{
		OutCaptureSpec.AttributeAggregator = AttributeAggregator;
	}
}
↓↓↓
/*
*如果此时AttributeAggregatorMap中有对应的Attribute值，则会直接返回，若没有则会向Map中进行添加
*/
FAggregatorRef& FActiveGameplayEffectsContainer::FindOrCreateAttributeAggregator(const FGameplayAttribute& Attribute)
{
	FAggregatorRef* RefPtr = AttributeAggregatorMap.Find(Attribute);
	if (RefPtr)
	{
		return *RefPtr;
	}

	// Create a new aggregator for this attribute.
	float CurrentBaseValueOfProperty = Owner->GetNumericAttributeBase(Attribute);
	UE_LOG(LogGameplayEffects, Log, TEXT("Creating new entry in AttributeAggregatorMap for %s. CurrentValue: %.2f"), *Attribute.GetName(), CurrentBaseValueOfProperty);

	FAggregator* NewAttributeAggregator = new FAggregator(CurrentBaseValueOfProperty);
	
	if (Attribute.IsSystemAttribute() == false)
	{
		NewAttributeAggregator->OnDirty.AddUObject(Owner, &UAbilitySystemComponent::OnAttributeAggregatorDirty, Attribute, false);
		NewAttributeAggregator->OnDirtyRecursive.AddUObject(Owner, &UAbilitySystemComponent::OnAttributeAggregatorDirty, Attribute, true);

		// Callback in case the set wants to do something
		const UAttributeSet* Set = Owner->GetAttributeSubobject(Attribute.GetAttributeSetClass());
		Set->OnAttributeAggregatorCreated(Attribute, NewAttributeAggregator);
	}

	return AttributeAggregatorMap.Add(Attribute, FAggregatorRef(NewAttributeAggregator));
}
```

## Source获取路径(获取时机):

```c++
void FGameplayEffectSpec::Initialize(const UGameplayEffect* InDef, const FGameplayEffectContextHandle& InEffectContext, float InLevel)
{
    ......
    // Prepare source tags before accessing them in the Components
	CaptureDataFromSource();
    ......
}

void FGameplayEffectSpec::CaptureDataFromSource(bool bSkipRecaptureSourceActorTags /*= false*/)
{
	// Capture source actor tags
	if (!bSkipRecaptureSourceActorTags)
	{
		RecaptureSourceActorTags();
	}

	// Capture source Attributes
	// Is this the right place to do it? Do we ever need to create spec and capture attributes at a later time? If so, this will need to move.
	CapturedRelevantAttributes.CaptureAttributes(EffectContext.GetInstigatorAbilitySystemComponent(), EGameplayEffectAttributeCaptureSource::Source);

	// Now that we have source attributes captured, re-evaluate the duration since it could be based on the captured attributes.
	float DefCalcDuration = 0.f;
	if (AttemptCalculateDurationFromDef(DefCalcDuration))
	{
		SetDuration(DefCalcDuration, false);
	}

	bCompletedSourceAttributeCapture = true;
}
```

## Target获取路径(获取时机):

```c++
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(const FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
    ......
        if (!OurCopyOfSpec)
		{
			StackSpec = MakeUnique<FGameplayEffectSpec>(Spec);
			OurCopyOfSpec = StackSpec.Get();

			UAbilitySystemGlobals::Get().GlobalPreGameplayEffectSpecApply(*OurCopyOfSpec, this);
			OurCopyOfSpec->CaptureAttributeDataFromTarget(this);	//从AttributeAggregatorMap将需要捕获的Target放到OurCopyOfSpec
		}
    ......
    ......
    	else if (Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant)
	{
		// This is a non-predicted instant effect (it never gets added to ActiveGameplayEffects)
		ExecuteGameplayEffect(*OurCopyOfSpec, PredictionKey);
	}
}
```

## 进入Execute_Implementation

ExecuteGameplayEffect 真正执行 GameplayEffect 的函数：

```c++
/** This is the main function that executes a GameplayEffect on Attributes and ActiveGameplayEffects */
void FActiveGameplayEffectsContainer::ExecuteActiveEffectsFrom(FGameplayEffectSpec &Spec, FPredictionKey PredictionKey)
{
    ......
        for (const FGameplayEffectExecutionDefinition& CurExecDef : SpecToUse.Def->Executions)
	{
	bool bRunConditionalEffects = true; // Default to true if there is no CalculationClass specified.

	if (CurExecDef.CalculationClass)
	{
		const UGameplayEffectExecutionCalculation* ExecCDO = CurExecDef.CalculationClass->GetDefaultObject<UGameplayEffectExecutionCalculation>();
		check(ExecCDO);

		// Run the custom execution
		FGameplayEffectCustomExecutionParameters ExecutionParams(SpecToUse, CurExecDef.CalculationModifiers, Owner, CurExecDef.PassedInTags, PredictionKey);
		FGameplayEffectCustomExecutionOutput ExecutionOutput;
		ExecCDO->Execute(ExecutionParams, ExecutionOutput); //到ExecutionCalculation类的 Execute_Implementation，至此ExecutionParams中的GESpecHandle已经包含Source和Target的值
    }
    ......
}
```

