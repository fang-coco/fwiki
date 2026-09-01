![[GC 2026-09-01 11.12.37.excalidraw]]

### PreCollectGarbageImpl

```cpp
template<bool bPerformFullPurge>
AUTORTFM_DISABLE void PreCollectGarbageImpl(EObjectFlags KeepFlags)
{
	// 只有全新开始的一轮 GC，才需要做完整的前置准备。
	if (!GIsIncrementalReachabilityPending)
	{
		// 先释放 GC 锁，避免 FlushAsyncLoading / 广播委托时和别的线程互相等待。
		ReleaseGCLock();

		// 先把异步加载刷干净，避免对象图在 mark 前继续变化。
		if (GFlushStreamingOnGC && IsAsyncLoading())
		{
			FlushAsyncLoading();
		}

		{
			// 广播 GC 前置委托，让外部系统先整理引用状态。
			UE::RemoteObject::Private::FUnsafeToMigrateScope UnsafeToMigrate;
			FCoreUObjectDelegates::GetPreGarbageCollectDelegate().Broadcast();
		}

		// 上面的委托可能又触发了加载，所以再补刷一次。
		if (GFlushStreamingOnGC && IsAsyncLoading())
		{
			FlushAsyncLoading();
		}

		// 前置扰动结束后重新进入 GC 临界区。
		AcquireGCLock();
	}

	// 从这里开始全局进入 GC 状态。
	GIsGarbageCollecting = true;

	if (!GIsIncrementalReachabilityPending)
	{
		// 只有新一轮 GC 才广播 started；恢复挂起轮次不重复广播。
		FCoreUObjectDelegates::GetGarbageCollectStartedDelegate().Broadcast();
	}

	if (IsIncrementalPurgePending())
	{
		// 先清掉上一轮还没 purge 完的垃圾，避免两轮 GC 生命周期重叠。
		IncrementalPurgeGarbage(false);
	}

	// reachability 期间锁住 UObject 哈希表，防止遍历引用图时被并发改写。
	GIsGarbageCollectingAndLockingUObjectHashTables = true;
	LockUObjectHashTables();

	if (!GIsIncrementalReachabilityPending)
	{
		// 如果当前禁用了 cluster，但运行时还有旧 cluster 残留，这里先全部拆掉。
		if (!GCreateGCClusters && GUObjectClusters.GetNumAllocatedClusters())
		{
			GUObjectClusters.DissolveClusters(true);
		}
	}
}
```

这一阶段还没有开始扫描对象引用，它先把后续 mark 依赖的运行环境固定下来：异步加载被刷到稳定状态，上一轮未完成的 purge 会先被推进，`UObject` 哈希表也会在 reachability 期间保持锁定。进入 `PerformReachabilityAnalysis(...)` 之后，对象图和对象表都已经处在适合做可达性分析的状态。

### PerformReachabilityAnalysis

```cpp
// 进入真正的可达性分析前，先处理“计时状态”
// 这里并不是决定要不要执行 PerformReachabilityAnalysis，
// 因为两个分支最终都会调用它。
if (bForceNonIncrementalReachability)
{
	// 非增量模式：这一轮从 0 开始重新统计 mark 阶段总耗时。
	IncrementalMarkPhaseTotalTime = 0.0;
	// 非增量模式：这一轮从 0 开始重新统计引用遍历耗时。
	ReferenceProcessingTotalTime = 0.0;
	// 真正进入可达性分析。
	PerformReachabilityAnalysis();
}
else
{
	// 增量模式下只有第一次进入时才清零；
	// 如果是恢复前面挂起的那一轮，就继续累计之前的时间。
	if (!GIsIncrementalReachabilityPending)
	{
		ReferenceProcessingTotalTime = 0.0;
		IncrementalMarkPhaseTotalTime = 0.0;
	}

	// 无论是否为恢复轮次，最终都会进入可达性分析。
	PerformReachabilityAnalysis();
}
```

这一段只决定本轮 mark 的执行模式。non-incremental 会把整轮可达性分析一次执行完；incremental 则允许在时间片耗尽时挂起，后续再从保存下来的上下文继续。这里向后传递的不是对象集合，而是 reachability 阶段的推进方式。

```cpp
void FReachabilityAnalysisState::PerformReachabilityAnalysis()
{
	if (!bIsSuspended)
	{
		// 不是从挂起状态恢复，说明这是本轮 reachability 的第一次进入。
		Init();
		// 允许通过 CVar 延迟若干轮，再真正开始增量可达性分析。
		NumRechabilityIterationsToSkip = FMath::Max(0, GDelayReachabilityIterations);
	}

	if (bPerformFullPurge)
	{
		// Full purge：这轮按完整 GC 语义执行，不走增量时间片切分。
		UE::GC::CollectGarbageFull(ObjectKeepFlags);
	}
	else if (NumRechabilityIterationsToSkip == 0 ||
		!bIsSuspended ||
		IterationTimeLimit <= 0.0f)
	{
		// 非 full purge 时，只要满足任一条件就进入增量主线：
		// 1. 已经不需要继续延迟
		// 2. 这是第一次进入，必须至少执行一轮初始化 mark
		// 3. 当前没有时间片限制，那就不必继续拖延
		UE::GC::CollectGarbageIncremental(ObjectKeepFlags);
	}
	else
	{
		// 还在“延迟若干轮再开始”阶段，这轮先不真正扫描对象图。
		--NumRechabilityIterationsToSkip;
	}

	// 收尾本轮 reachability iteration，更新迭代计数。
	FinishIteration();
}
```

`FReachabilityAnalysisState::PerformReachabilityAnalysis()` 把整轮 mark 包装成了一个可恢复的迭代过程。第一次进入时会初始化状态，并按 `GDelayReachabilityIterations` 决定是否延迟若干轮再真正开始扫描；之后根据 `bPerformFullPurge` 和时间片条件，分流到 `CollectGarbageFull(...)` 或 `CollectGarbageIncremental(...)`。

```cpp
AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageIncremental(EObjectFlags KeepFlags)
{
	// 增量版本只是把模板参数设为 false。
	CollectGarbageImpl<false>(KeepFlags);
}

AUTORTFM_DISABLE FORCENOINLINE static void CollectGarbageFull(EObjectFlags KeepFlags)
{
	// Full 版本把模板参数设为 true。
	CollectGarbageImpl<true>(KeepFlags);
}
```

`CollectGarbageFull(...)` 和 `CollectGarbageIncremental(...)` 最终都会汇入同一个 `CollectGarbageImpl` 骨架，差异由模板参数 `bPerformFullPurge` 往下传递。full 和 incremental 的区别从这里开始只体现在是否允许挂起、是否要求本轮完整收尾等行为上。

```cpp
void PerformReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	if (!GReachabilityState.IsSuspended())
	{
		// 第一次进入这一轮 mark，先建立初始输入。
		StartReachabilityAnalysis(KeepFlags, Options);
		// Verse 侧也会参与标记，这里先启动 Verse GC。
		StartVerseGC();
	}

	while (true)
	{
		// 每一轮 pass 都会消费一批输入对象，并继续扩散可达对象。
		PerformReachabilityAnalysisPass(Options);

		if (GReachabilityState.IsSuspended())
		{
			// 增量模式下如果时间片耗尽，就把当前进度挂起，下一轮继续。
			if (EnumHasAnyFlags(Options, EGCOptions::IncrementalReachability)
				&& GReachabilityState.IsTimeLimitExceeded())
			{
				break;
			}
		}
		else if (Private::GReachableObjects.IsEmpty()
			&& Private::GReachableClusters.IsEmpty())
		{
			// 没有挂起，而且全局补充队列也空了，说明没有新的可达对象需要继续传播。
			StopVerseGC();
			break;
		}
	}
}
```

第一次进入这一轮 mark 时，`StartReachabilityAnalysis(...)` 会准备初始输入，并启动 Verse 侧的配套标记；之后每一轮 `PerformReachabilityAnalysisPass(...)` 都会消费当前输入并继续扩散可达对象。增量模式在时间片耗尽时会把 `FWorkerContext` 挂起，下一帧从该上下文继续；如果没有挂起，而且 `GReachableObjects` 与 `GReachableClusters` 也都为空，就说明这一轮引用传播已经收敛。

```cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	// 初始化初始引用收集上下文。
	BeginInitialReferenceCollection(Options);

	// 重置这轮 mark 的对象计数和第一批待扫描对象列表。
	GObjectCountDuringLastMarkPhase.Reset();
	InitialObjects.Reset();

	if (FPlatformProperties::RequiresCookedData()
		&& GUObjectArray.IsDisregardForGC(FGCObject::GGCObjectReferencer))
	{
		// permanent object pool 里的 GGCObjectReferencer 也要手动塞进初始输入。
		InitialObjects.Add(FGCObject::GGCObjectReferencer);
	}

	// 把所有对象切到新的 reachability 初始状态，
	// 再从 cluster、root set、KeepFlags 对象里恢复第一批可达对象。
	MarkObjectsAsUnreachable(KeepFlags);
}
```

`StartReachabilityAnalysis(...)` 在这一层准备两类起始输入：`BeginInitialReferenceCollection(Options)` 负责构造原生侧的 `InitialReferences`，`MarkObjectsAsUnreachable(KeepFlags)` 负责把 UObject 侧的 `InitialObjects` 填出来。后续的 pass 至少会从这两类输入开始推进。

```cpp
FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	EGatherOptions GatherOptions = GetObjectGatherOptions();

	if (const bool bInitialMark = !Stats.bFoundGarbageRef)
	{
		// 正常的初始 mark：直接交换 Reachable / MaybeUnreachable 的语义，
		// 以极低成本把所有对象切换成“默认不可达，等待重新救活”的状态。
		FGCFlags::SwapReachableAndMaybeUnreachable();
	}
	else
	{
		// 重入场景下不能直接交换标志位，需要显式重置状态。
		ResetReachabilityFlags(GatherOptions);
	}

	// 记录这一轮参与判断的对象数量。
	GObjectCountDuringLastMarkPhase.Set(
		GUObjectArray.GetObjectArrayNumMinusAvailable() - GUObjectArray.GetFirstGCIndex());

	// 先恢复 cluster 体系里的可达对象。
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	// 再恢复 root set 和 KeepFlags 指定的额外保活对象。
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
}
```

`MarkObjectsAsUnreachable(...)` 会先把全局对象状态切换到“默认可能不可达”，再从 cluster、root set 和 `KeepFlags` 命中的对象里恢复第一批确定可达的对象。它向后传递的关键结果是 `InitialObjects`，第一轮 `PerformReachabilityAnalysisPass(...)` 直接消费的就是这份列表。

```cpp
FORCENOINLINE void MarkClusteredObjectsAsReachable(const EGatherOptions Options, TArray<UObject*>& OutRootObjects)
{
	using FMarkClustersState = TThreadedGather<FMarkClustersArrays, FMarkClustersArrays>;

	std::atomic<int32> TotalClusteredObjects = 0;
	FMarkClustersState GatherClustersState;
	TArray<FUObjectCluster>& ClusterArray = GUObjectClusters.GetClustersUnsafe();

	// cluster 的处理是并行的，每个线程拿到自己的一段 cluster 区间。
	const int32 NumThreads = !!(Options & EGatherOptions::Parallel)
		? FMath::Min(GetNumCollectReferenceWorkers(), (ClusterArray.Num() + 1) / 2)
		: 1;
	GatherClustersState.Start(Options, ClusterArray.Num(), 0, NumThreads);
	FMarkClustersState::FThreadIterators& ThreadIterators = GatherClustersState.GetThreadIterators();

	ParallelFor(TEXT("GC.MarkClusteredObjectsAsReachable"), GatherClustersState.NumWorkerThreads(), 1,
		[&ThreadIterators, &ClusterArray, &TotalClusteredObjects](int32 ThreadIndex)
	{
		FMarkClustersState::FIterator& ThreadState = ThreadIterators[ThreadIndex];
		int32 ThisThreadClusteredObjects = 0;

		while (ThreadState.Index <= ThreadState.LastIndex)
		{
			int32 ClusterIndex = ThreadState.Index++;
			FUObjectCluster& Cluster = ClusterArray[ClusterIndex];
			if (Cluster.RootIndex >= 0)
			{
				ThisThreadClusteredObjects += Cluster.Objects.Num();

				FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[Cluster.RootIndex];
				if (!RootItem->IsGarbage())
				{
					bool bKeepCluster = RootItem->HasAnyFlags(EInternalObjectFlags_RootFlags) || RootItem->GetRefCount() > 0;
					if (bKeepCluster)
					{
						// root 自己先命中保活条件，线程先记下这个 cluster 需要继续保留。
						ThreadState.Payload.KeepClusters.Add(RootItem);
					}

					for (int32 ObjectIndex : Cluster.Objects)
					{
						FUObjectItem* ClusteredItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ObjectIndex];
						// cluster 内对象先批量恢复为 reachable。
						FGCFlags::FastMarkAsReachableAndClearReachableInClusterInterlocked_ForGC(ClusteredItem);

						if (!bKeepCluster && (ClusteredItem->HasAnyFlags(EInternalObjectFlags_RootFlags) || ClusteredItem->GetRefCount() > 0))
						{
							// root 没命中，但 cluster 里有成员命中保活条件，整个 cluster 仍然要保留。
							ThreadState.Payload.KeepClusters.Add(RootItem);
							bKeepCluster = true;
						}
					}
				}
				else
				{
					// root 自己已经是垃圾，后面统一解散这个 cluster。
					ThreadState.Payload.ClustersToDissolve.Add(RootItem);
				}
			}
		}
		TotalClusteredObjects += ThisThreadClusteredObjects;
	});

	FMarkClustersArrays MarkClustersResults;
	GatherClustersState.Finish(MarkClustersResults);

	for (FUObjectItem* ObjectItem : MarkClustersResults.ClustersToDissolve)
	{
		// 先把无效 cluster 解散，里面对象重新回到普通对象流程。
		GUObjectClusters.DissolveClusterAndMarkObjectsAsUnreachable(ObjectItem);
		GUObjectClusters.SetClustersNeedDissolving();
	}

	for (FUObjectItem* ObjectItem : MarkClustersResults.KeepClusters)
	{
		// 真正向后传递的不是 KeepClusters，而是展开后的 UObject 输入列表。
		MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), OutRootObjects);
	}
}
```

`MarkClusteredObjectsAsReachable(...)` 在线程本地只暂存 `KeepClusters` 和 `ClustersToDissolve`，最终真正向后交接的是展开后的 `OutRootObjects`。cluster 分支在这里被转换成了第一轮 pass 可以直接扫描的 `UObject*` 输入。

```cpp
FORCENOINLINE void MarkRootObjectsAsReachable(const EGatherOptions Options, const EObjectFlags KeepFlags, TArray<UObject*>& OutRootObjects)
{
	using FMarkRootsState = TThreadedGather<TArray<UObject*>>;
	FMarkRootsState MarkRootsState;

	GRootsMutex.Lock();
	ProcessDirtyRootsNoLock();
	TArray<int32> RootsArray(GRoots.Array());
	GRootsMutex.Unlock();

	MarkRootsState.Start(Options, RootsArray.Num());
	FMarkRootsState::FThreadIterators& ThreadIterators = MarkRootsState.GetThreadIterators();

	ParallelFor(TEXT("GC.MarkRootObjectsAsReachable"), MarkRootsState.NumWorkerThreads(), 1,
		[&ThreadIterators, &RootsArray](int32 ThreadIndex)
	{
		FMarkRootsState::FIterator& ThreadState = ThreadIterators[ThreadIndex];
		while (ThreadState.Index <= ThreadState.LastIndex)
		{
			FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[RootsArray[ThreadState.Index++]];
			UObject* Object = static_cast<UObject*>(RootItem->GetObject());
			// 显式 root set 里的对象直接恢复为 reachable。
			FGCFlags::FastMarkAsReachableInterlocked_ForGC(RootItem);
			// 线程本地先收集，最后统一并进初始输入。
			ThreadState.Payload.Add(Object);
		}
	});

	if (KeepFlags != RF_NoFlags)
	{
		ParallelFor(TEXT("GC.SlowMarkObjectAsReachable"), MarkObjectsState.NumWorkerThreads(), 1,
			[&ThreadIterators, &KeepFlags](int32 ThreadIndex)
		{
			while (ThreadState.Index <= ThreadState.LastIndex)
			{
				FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ThreadState.Index++];
				UObject* Object = static_cast<UObject*>(ObjectItem->GetObject());
				if (Object && !ObjectItem->HasAnyFlags(EInternalObjectFlags_RootFlags) && Object->HasAnyFlags(KeepFlags))
				{
					// 命中 KeepFlags 的对象也补进同一批初始输入。
					FGCFlags::FastMarkAsReachableInterlocked_ForGC(ObjectItem);
					ThreadState.Payload.Add(Object);
				}
			}
		});
	}

	// 两路输入最后都并到 OutRootObjects，形成完整的 InitialObjects。
	MarkRootsState.Finish(OutRootObjects);
	MarkObjectsState.Finish(OutRootObjects);
}
```

`MarkRootObjectsAsReachable(...)` 把显式 root set 和命中 `KeepFlags` 的对象也并入 `OutRootObjects`。至此，`InitialObjects` 同时包含了 cluster 展开的对象、root set 对象和额外保活对象。

```cpp
void BeginInitialReferenceCollection(EGCOptions Options)
{
	InitialReferences.Reset();

	if (IsParallel(Options))
	{
		// 并行模式下，初始引用收集会先作为独立任务异步启动，
		// 和 MarkObjectsAsUnreachable 这一段重叠执行。
		InitialCollection = UE::Tasks::Launch(TEXT("CollectInitialReferences"),
			[&]() { FGCObject::GGCObjectReferencer->AddInitialReferences(InitialReferences); });
	}
}

TConstArrayView<UObject**> GetInitialReferences(EGCOptions Options)
{
	if (IsParallel(Options))
	{
		// 真正消费前再等待，减少主线程空等时间。
		InitialCollection.Wait();
	}
	return InitialReferences;
}
```

除了 `InitialObjects` 之外，系统还会构造 `InitialReferences`。并行模式下，初始引用收集会异步启动，到真正需要写入 `FWorkerContext` 时才等待完成；因此这一部分和前面的 mark 准备阶段是重叠执行的。

```cpp
void PerformReachabilityAnalysisPass(const EGCOptions Options)
{
	FWorkerContext* Context = nullptr;

	if (!GReachabilityState.IsSuspended())
	{
		// 新的一轮 pass：分配新的上下文，并接入这一轮的原生初始引用。
		Context = Pool.AllocateFromPool();
		Context->InitialNativeReferences = GetInitialReferences(Options);
	}
	else
	{
		// 恢复挂起轮次时，直接接着上一个 Context 做，不重新建输入。
		Context = GReachabilityState.GetContextArray()[0];
		Context->bDidWork = false;
		InitialObjects.Reset();
	}

	if (!Private::GReachableObjects.IsEmpty())
	{
		// barrier / 增量阶段新发现的普通对象，补回当前轮输入。
		Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
	}
	if (!Private::GReachableClusters.IsEmpty())
	{
		// 新发现的 cluster 先展开成对象，再并回当前轮输入。
		for (FUObjectItem* ObjectItem : KeepClusterRefs)
		{
			MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), InitialObjects);
		}
	}

	// 到这里这一轮真正要消费的 UObject 输入才正式挂到 Context 上。
	Context->SetInitialObjectsUnpadded(InitialObjects);
	PerformReachabilityAnalysisOnObjects(Context, Options);
}
```

`PerformReachabilityAnalysisPass(...)` 是准备阶段和扫描阶段的交接点。前面准备出来的 `InitialObjects`、`InitialReferences`，以及增量/barrier 阶段补回来的对象和 cluster，都会在这里汇入 `Context`，后续 collector 只围绕当前 `FWorkerContext` 里的输入继续向下推进。

```cpp
struct FWorkerContext
{
	// 这一轮 pass 直接要扫描的 UObject 输入。
	TConstArrayView<UObject*> InitialObjects;
	// 原生侧补进来的初始引用输入。
	TConstArrayView<UObject**> InitialNativeReferences;
	// 扫描过程中发现的新对象不会立刻递归深入，
	// 而是先按 block 写进这里，等后面批量继续处理。
	FWorkBlockifier ObjectsToSerialize;
	// 增量模式时间片耗尽时，当前 worker 的执行点会保存在这里。
	bool bIsSuspended = false;
	bool bDidWork = false;
};
```

`FWorkerContext` 同时保存了这一轮 pass 的入口数据、处理中间态以及挂起恢复状态。`InitialObjects` 和 `InitialNativeReferences` 是输入，`ObjectsToSerialize` 是扫描过程中持续增长的后续输入，`bIsSuspended` 和 `bDidWork` 则用来描述这一轮 worker 的执行状态。

```cpp
template <typename ProcessorType, typename CollectorType>
void ProcessObjectArray(FWorkerContext& Context)
{
	Context.bDidWork = true;
	Context.bIsSuspended = false;
	CollectorType Collector(Processor, Context);
	auto Dispatcher = GetDispatcher(Collector, Processor, Context);

StoleContext:
	// 先消费原生侧初始引用，把 C++ 层额外补进来的 UObject* 先送入处理器。
	Context.ReferencingObject = FGCObject::GGCObjectReferencer;
	for (UObject** InitialReference : Context.InitialNativeReferences)
	{
		Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
	}

	// 再处理 UObject 侧的初始输入或上一个 block 里新发现的对象。
	TConstArrayView<UObject*> CurrentObjects = Context.InitialObjects;
	while (true)
	{
		Context.Stats.AddObjects(CurrentObjects.Num());
		ProcessObjects(Dispatcher, CurrentObjects);

		if (CurrentObjects.GetData() != Context.InitialObjects.GetData())
		{
			// 这个 block 已经扫完，可以把它占用的存储释放掉。
			Context.ObjectsToSerialize.FreeOwningBlock(CurrentObjects.GetData());
		}

		if (Processor.IsTimeLimitExceeded())
		{
			// 时间片用完前，先把暂存结果刷回 Context，再挂起当前 worker。
			FlushWork(Dispatcher);
			Dispatcher.Suspend();
			SuspendWork(Context);
			return;
		}

		int32 BlockSize = FWorkBlock::ObjectCapacity;
		FWorkBlockifier& RemainingObjects = Context.ObjectsToSerialize;
		FWorkBlock* Block = RemainingObjects.PopFullBlock<Options>();
		if (!Block)
		{
			// 自己没有完整 block 时，先把 dispatcher 手里的新对象刷出来。
			FlushWork(Dispatcher);

			// 先尝试本地 partial block，再尝试从别的 worker 偷工作。
			if (Block = RemainingObjects.PopFullBlock<Options>(); Block);
			else if (Block = RemainingObjects.PopPartialBlock(BlockSize); Block);
			else if (bIsParallel)
			{
				switch (StealWork(Context, Collector, Block, Options))
				{
					case ELoot::Block:
						break;
					case ELoot::Context:
						goto StoleContext;
					default:
						break;
				}
			}

			if (!Block)
			{
				// 当前 worker 彻底没有剩余输入，这一轮 pass 可以结束。
				break;
			}
		}

		// 取出下一批待扫描对象，继续沿引用往外扩散。
		CurrentObjects = MakeArrayView(Block->Objects, BlockSize);
	}
}
```

`ProcessObjectArray(...)` 的职责是批量消费“当前已经确认需要扫描的对象”，并把新发现的对象组织成后续输入。它本身不解释对象成员里的引用如何被定位；那一层逻辑要到 `ProcessObjects(...) -> VisitMembers(...)` 才展开。这个函数结束后，最重要的结果不是返回值，而是 `ObjectsToSerialize` 中累计的新对象，或者增量模式下挂起后的 `Context`。

```cpp
FORCEINLINE_DEBUGGABLE void ProcessObjects(DispatcherType& Dispatcher, TConstArrayView<UObject*> CurrentObjects)
{
	for (...)
	{
		UObject* CurrentObject = It.GetCurrentObject();
		UClass* Class = CurrentObject->GetClass();

		if (!!(Options & EGCOptions::AutogenerateSchemas) && !Class->HasAnyClassFlags(CLASS_TokenStreamAssembled))
		{
			// 第一次见到这个类时，先补齐 schema，后面才能按偏移量扫描成员。
			Class->AssembleReferenceTokenStream();
		}

		FSchemaView Schema = Class->ReferenceSchema.Get();
		Dispatcher.Context.ReferencingObject = CurrentObject;

		// 这几个固定引用不经过 schema，直接单独送入后续处理。
		Dispatcher.HandleImmutableReference(Class, EMemberlessId::Class, EOrigin::Other);
		Dispatcher.HandleImmutableReference(CurrentObject->GetOuter(), EMemberlessId::Outer, EOrigin::Other);
		...

		if (!Schema.IsEmpty())
		{
			// 普通成员引用从这里开始按 schema 展开。
			Private::VisitMembers(Dispatcher, Schema, CurrentObject, Class->GetPropertiesStartOffset());
		}
	}
}
```

`ProcessObjects(...)` 先取得当前对象的 `Class->ReferenceSchema`，再把固定引用和普通成员引用分别送入后续处理。固定引用包括 `Class`、`Outer`、`ExternalPackage` 等；普通成员引用则交给 `VisitMembers(...)` 按 schema 展开。两类引用最终都会汇入 `HandleTokenStreamObjectReference(...)`。

```cpp
template<class DispatcherType, typename ObjectType>
void VisitMembers(DispatcherType& Dispatcher, FSchemaView Schema, ObjectType* Instance, int32 OffsetToPropertiesStart)
{
	for (const FMemberWord* WordIt = Schema.GetWords(); true; ++WordIt)
	{
		for (FMemberUnpacked Member : FMemberWordUnpacked(WordIt->Members).Members)
		{
			// 按 schema 记录的偏移量，直接定位到对象内存中的成员。
			uint8* MemberPtr = (uint8*)(InstanceCursor + Member.WordOffset);

			switch (Member.Type)
			{
			case EMemberType::Reference:
				// 普通 UObject* 成员直接取出来交给 dispatcher。
				Dispatcher.HandleKillableReference(*(UObject**)MemberPtr, FMemberId(DebugIdx), Origin);
				break;
			case EMemberType::ReferenceArray:
				// 数组成员会把数组里的对象逐个继续交给 dispatcher。
				Dispatcher.HandleKillableArray(*(TArray<UObject*>*)MemberPtr, FMemberId(DebugIdx), Origin);
				break;
			case EMemberType::StructArray:
				// 结构体数组不会直接结束，而是继续深入 inner schema。
				VisitStructArray(Dispatcher, FSchemaView((++WordIt)->InnerSchema, Origin), *(FScriptArray*)MemberPtr);
				break;
			case EMemberType::ARO:
				// ARO 走自定义补充引用逻辑，但最终还是汇入同一条 processor 入口。
				CallARO(Dispatcher, Instance, *++WordIt);
				return;
			case EMemberType::Stop:
				return;
			...
			}
		}
	}
}
```

`VisitMembers(...)` 按 `ReferenceSchema` 记录的成员类型和偏移量直接读取对象内存，把所有需要继续参与 GC 的引用字段提取出来。GC 在这一层不是遍历属性名做判断，而是直接消费类提前构造好的 schema。

```cpp
FORCEINLINE void HandleReferenceDirectly(UObject* ReferencingObject, UObject*& Object, FMemberId MemberId, EOrigin Origin, bool bAllowReferenceElimination) const
{
	if (IsObjectHandleResolved_ForGC(*reinterpret_cast<FObjectHandle*>(&Object)))
	{
		Processor.HandleTokenStreamObjectReference(Context, ReferencingObject, Object, MemberId, Origin, bAllowReferenceElimination);
	}
	Context.Stats.AddReferences(1);
}
```

`HandleReferenceDirectly(...)` 把从对象内存中取出的引用正式交给 processor。processor 会在这里判断该对象是否第一次被认定为可达，如果是，就把它写进 `Context.ObjectsToSerialize`，等待后续 block 继续扫描。

```cpp
virtual void HandleObjectReference(UObject*& Object, const UObject* ReferencingObject, const FProperty* ReferencingProperty) override
{
	if (!ReferencingObject)
	{
		ReferencingObject = Context.GetReferencingObject();
	}
	Processor.HandleTokenStreamObjectReference(Context, const_cast<UObject*>(ReferencingObject), Object, EMemberlessId::Collector, EOrigin::Other, false);
}
```

`AddReferencedObjects(...)` 这条自定义路径最终也会回到 `Processor.HandleTokenStreamObjectReference(...)`。因此，无论引用来自普通属性、容器、嵌套结构体还是 ARO，自后续“标记为可达并入队”这一层开始，处理入口是统一的。

![[GC 2026-09-01 15.56.03.excalidraw]]

引用传播主线可以整理成：`InitialObjects / InitialReferences` 作为第一批输入进入 `FWorkerContext`，`ProcessObjects(...)` 和 `VisitMembers(...)` 展开对象内部的引用，`HandleTokenStreamObjectReference(...)` 把新发现的对象转成新的扫描输入，`Context.ObjectsToSerialize` 则把这些输入组织成后续批处理继续消费的 block。

#### ReferenceSchema 是怎么建立的

```cpp
void UClass::AssembleReferenceTokenStream(bool bForce)
{
	if (ClassFlags & CLASS_Native)
	{
		AssembleReferenceTokenStreamInternal(bForce);
	}
	else
	{
		FScopeLock NonNativeLock(&GAssembleSchemaLock);
		AssembleReferenceTokenStreamInternal(bForce);
	}
}

void UClass::AssembleReferenceTokenStreamInternal(bool bForce)
{
	if (!HasAnyClassFlags(CLASS_TokenStreamAssembled) || bForce)
	{
		int32 StartOffset = -GetPropertiesStartOffset();
		FSchemaBuilder Schema(0);
		FSchemaView SuperSchema;

		if (UClass* SuperClass = GetSuperClass())
		{
			SuperClass->AssembleReferenceTokenStreamInternal();
			SuperSchema = SuperClass->ReferenceSchema.Get();
			Schema.Append(SuperSchema, StartOffset - (-SuperClass->GetPropertiesStartOffset()));
		}

		for (TFieldIterator<FProperty> It(this, EFieldIteratorFlags::ExcludeSuper); It; ++It)
		{
			FProperty* Property = *It;
			Property->EmitReferenceInfo(Schema, StartOffset, EncounteredStructProps, DebugPath);
		}

		if (ClassFlags & CLASS_Intrinsic)
		{
			Schema.Append(UE::GC::GetIntrinsicSchema(this), 0);
		}

		bool bReuseSuper = Schema.NumMembers() == NumSuperMembers && GetARO(this) == GetARO(GetSuperClass());
		FSchemaView View(bReuseSuper ? SuperSchema : Schema.Build(GetARO(this)), Origin);
		ReferenceSchema.Set(View);
		ClassFlags |= CLASS_TokenStreamAssembled;
	}
}
```

`UClass::AssembleReferenceTokenStreamInternal(...)` 会先继承父类的 schema，再把当前类声明的 `FProperty` 逐个交给 `Property->EmitReferenceInfo(...)` 写入 `FSchemaBuilder`，最后合成 `ReferenceSchema`。后续 `ProcessObjects(...) -> VisitMembers(...)` 只是消费这张 schema，而不会重新分析类定义。

#### Cluster 是怎么建立的

```cpp
void UObjectBaseUtility::CreateCluster()
{
	FUObjectItem* RootItem = GUObjectArray.IndexToObject(InternalIndex);
	if (RootItem->GetOwnerIndex() != 0 || RootItem->HasAnyFlags(EInternalObjectFlags::ClusterRoot))
	{
		return;
	}

	const int32 ClusterIndex = GUObjectClusters.AllocateCluster(InternalIndex);
	FUObjectCluster& Cluster = GUObjectClusters[ClusterIndex];

	FClusterReferenceProcessor Processor(InternalIndex, Cluster, GetOutermost());
	FGCArrayStruct ArrayStruct;
	TArray<UObject*> ObjectsToProcess = { static_cast<UObject*>(this) };
	ArrayStruct.SetInitialObjectsUnpadded(ObjectsToProcess);
	CollectReferences(Processor, ArrayStruct);

	RootItem->SetClusterIndex(ClusterIndex);
	RootItem->SetFlags(EInternalObjectFlags::ClusterRoot);

	if (Cluster.Objects.Num() >= GUObjectClusters.GetMinClusterSize())
	{
		Cluster.Objects.Sort();
		Cluster.ReferencedClusters.Sort();
		Cluster.MutableObjects.Sort();
	}
	else
	{
		for (int32 ClusterObjectIndex : Cluster.Objects)
		{
			GUObjectArray.IndexToObjectUnsafeForGC(ClusterObjectIndex)->SetOwnerIndex(0);
		}
		GUObjectClusters.FreeCluster(ClusterIndex);
	}
}
```

`UObjectBaseUtility::CreateCluster()` 会以当前对象为 root，借助 `FClusterReferenceProcessor` 先收出 cluster 内部对象以及跨 cluster 的引用关系。成功建立后，核心结果会写入 `Cluster.Objects`、`Cluster.ReferencedClusters` 和 `Cluster.MutableObjects`；如果最终成员数量不足最小阈值，这个 cluster 会被直接丢弃。

```cpp
void CreateClustersFromPackage(FLinkerLoad* PackageLinker, TArray<UObject*>& OutClusterObjects)
{
	if (CanCreateObjectClusters())
	{
		for (FObjectExport& Export : PackageLinker->ExportMap)
		{
			if (Export.Object && Export.Object->CanBeClusterRoot())
			{
				OutClusterObjects.Add(Export.Object);
			}
		}
	}
}
```

包加载阶段会先从 export 中筛出可以作为 cluster root 的对象，再由这些对象继续调用 `CreateCluster()`。后续 `MarkClusteredObjectsAsReachable(...)` 和 `DissolveUnreachableClusters(...)` 操作的就是这里建立出来的 cluster 数据。

### PostCollectGarbageImpl

```cpp
template<bool bPerformFullPurge>
void PostCollectGarbageImpl(EObjectFlags KeepFlags)
{
	if (!GIsIncrementalReachabilityPending)
	{
		TConstArrayView<TUniquePtr<FWorkerContext>> AllContexts = ContextPool.PeekFree();

		UpdateGCHistory(AllContexts);
		if (GUObjectClusters.ClustersNeedDissolving())
		{
			GUObjectClusters.DissolveClusters();
		}

		const EGatherOptions GatherOptions = GetObjectGatherOptions();
		DissolveUnreachableClusters(GatherOptions);

		if (GReachabilityState.GetNumIterations() > 1)
		{
			ClearWeakReferences<true>(AllContexts);
		}
		else
		{
			ClearWeakReferences<false>(AllContexts);
		}

		GGatherUnreachableObjectsState.Init();
		if (bPerformFullPurge || !GAllowIncrementalGather || !FGCFlags::IsIncrementalGatherUnreachableSupported())
		{
			GatherUnreachableObjects(GatherOptions, 0.0);
		}
	}

	UnlockUObjectHashTables();
	ReleaseGCLock();

	if (!GIsIncrementalReachabilityPending)
	{
		FCoreUObjectDelegates::PostReachabilityAnalysis.Broadcast();

		if (bPerformFullPurge || !GIncrementalBeginDestroyEnabled)
		{
			UnhashUnreachableObjects(false);
		}

		GObjPurgeIsRequired = true;
		if (bPerformFullPurge)
		{
			IncrementalPurgeGarbage(false);
		}
		...
	}
}
```

`PostCollectGarbageImpl(...)` 负责把 reachability 阶段得到的“不可达结果”转换成销毁阶段可以直接消费的输入。它会收尾所有 `FWorkerContext`，处理 cluster 和弱引用，然后把散落在对象数组中的垃圾候选对象整理成 `GUnreachableObjects`，后面的 `UnhashUnreachableObjects(...)` 和 `IncrementalDestroyGarbage(...)` 都围绕这张表继续推进。

```cpp
static void DissolveUnreachableClusters(EGatherOptions Options)
{
	ParallelFor(..., [&ThreadIterators, &ClusterArray](int32 ThreadIndex)
	{
		while (ThreadState.Index <= ThreadState.LastIndex)
		{
			FUObjectCluster& Cluster = ClusterArray[ClusterIndex];
			FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[Cluster.RootIndex];
			if (FGCFlags::IsMaybeUnreachable_ForGC(RootItem))
			{
				for (int32 ObjectIndex : Cluster.Objects)
				{
					FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ObjectIndex];
					if (!ObjectItem->HasAnyFlags(EInternalObjectFlags::ReachableInCluster))
					{
						// cluster root 已经不可达时，这些成员要重新回到普通 MaybeUnreachable 流程。
						FGCFlags::SetMaybeUnreachable_ForGC(ObjectItem);
					}
				}
				// 这一轮先记下要释放的 cluster，下半段统一 FreeCluster。
				ThreadState.Payload.Add(ClusterIndex);
			}
		}
	});

	GatherClustersState.Finish(ClustersToDestroy);
	for (int32 ClusterIndex : ClustersToDestroy)
	{
		// cluster 元数据在这里真正释放掉。
		GUObjectClusters.FreeCluster(ClusterIndex);
	}
}
```

`DissolveUnreachableClusters(...)` 会把 root 已经不可达的 cluster 解散，并把其中未带 `ReachableInCluster` 标记的对象重新打回普通 `MaybeUnreachable` 流程。这样后面的 `GatherUnreachableObjects(...)` 才能完整收集这些对象。

```cpp
template <bool bGatheredWithIncrementalReachability>
static void ClearWeakReferences(TConstArrayView<TUniquePtr<FWorkerContext>> Contexts)
{
	for (const TUniquePtr<FWorkerContext>& Context : Contexts)
	{
		for (FWeakReferenceInfo& ReferenceInfo : Context->WeakReferences)
		{
			if (ReferencedObject && FGCFlags::IsMaybeUnreachable_ForGC(ReferencedObject))
			{
				if constexpr (bGatheredWithIncrementalReachability)
				{
					// 增量可达性下不能直接清空，先记下 owner，后面再统一修正。
					ObjectsThatNeedWeakReferenceClearing.Add(ReferenceInfo.ReferenceOwner);
				}
				else
				{
					// 非增量模式可以直接把弱引用置空。
					*ReferenceInfo.Reference = nullptr;
				}
			}
		}
	}
	...
}
```

`ClearWeakReferences(...)` 接在 cluster dissolve 之后执行，是为了让弱引用清理基于修正后的垃圾集合进行。非增量模式可以直接把弱引用置空；增量 reachability 下则先记录 owner，后面再统一修正。

```cpp
bool GatherUnreachableObjects(EGatherOptions Options, double TimeLimit)
{
	GUnreachableObjects.Reset();
	GUnreachableObjectIndex = 0;
	...
	ParallelFor(..., [&ThreadIterators](int32 ThreadIndex)
	{
		while (Iterator.Index <= Iterator.LastIndex)
		{
			FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[Iterator.Index++];
			if (FGCFlags::IsMaybeUnreachable_ForGC(ObjectItem))
			{
				// 这里把“候选垃圾”正式提升成 unreachable，并收进统一列表。
				FGCFlags::SetUnreachable(ObjectItem);
				Iterator.Payload.Add({ ObjectItem });
			}
		}
	});
	// 线程局部结果最后汇总成全局 GUnreachableObjects。
	GGatherUnreachableObjectsState.Finish(GUnreachableObjects);
	NotifyUnreachableObjects(UnreachableObjectItemsView);
}
```

`GatherUnreachableObjects(...)` 把散落在 `GUObjectArray` 里的 `MaybeUnreachable` 对象整理成全局顺序表 `GUnreachableObjects`。从这一层开始，后面的销毁流程不再需要重新扫描整个对象数组去查找垃圾对象。

```cpp
bool UnhashUnreachableObjects(bool bUseTimeLimit, double TimeLimit)
{
	if (GGatherUnreachableObjectsState.IsPending())
	{
		// 如果前面的 gather 还没收完，先把不可达对象列表补齐。
		if (GatherUnreachableObjects(GatherOptions, bUseTimeLimit ? GatherTimeLimit : 0.0))
		{
			return true;
		}
	}

	while (GUnreachableObjectIndex < GUnreachableObjects.Num())
	{
		FUObjectItem* ObjectItem = GUnreachableObjects[GUnreachableObjectIndex++].ObjectItem;
		UObject* Object = static_cast<UObject*>(ObjectItem->GetObject());
		// 真正进入销毁生命周期的第一步：先发 BeginDestroy。
		Object->ConditionalBeginDestroy();
	}

	return false;
}
```

`UnhashUnreachableObjects(...)` 会为 `GUnreachableObjects` 中的对象逐个调用 `ConditionalBeginDestroy()`，对象在这里正式进入销毁生命周期。下一步 `IncrementalDestroyGarbage(...)` 接手时，不再负责判定垃圾对象，而是负责继续推进这些对象从 `BeginDestroy` 到 `FinishDestroy` 再到真正释放的过程。

### IncrementalDestroyGarbage

```cpp
bool IncrementalDestroyGarbage(bool bUseTimeLimit, double TimeLimit)
{
	if (!GObjFinishDestroyHasBeenRoutedToAllObjects)
	{
		while (GObjCurrentPurgeObjectIndex < GUnreachableObjects.Num())
		{
			FUObjectItem* ObjectItem = GUnreachableObjects[GObjCurrentPurgeObjectIndex].ObjectItem;
			if (ObjectItem->IsUnreachable())
			{
				UObject* Object = static_cast<UObject*>(ObjectItem->GetObject());
				if (Object->IsReadyForFinishDestroy())
				{
					// 当前帧已经 ready，就直接补发 FinishDestroy。
					Object->ConditionalFinishDestroy();
				}
				else
				{
					// 还没 ready 的对象先进入 PendingDestruction，后面反复轮询。
					GGCObjectsPendingDestruction.Add(Object);
					GGCObjectsPendingDestructionCount++;
				}
			}
			++GObjCurrentPurgeObjectIndex;
			...
		}

		while (GGCObjectsPendingDestructionCount > 0)
		{
			int32 CurPendingObjIndex = 0;
			while (CurPendingObjIndex < GGCObjectsPendingDestructionCount)
			{
				UObject* Object = GGCObjectsPendingDestruction[CurPendingObjIndex];
				if (Object->IsReadyForFinishDestroy())
				{
					// 之前没 ready 的对象，一旦 ready 就立刻 FinishDestroy 并移出 pending。
					Object->ConditionalFinishDestroy();
					GGCObjectsPendingDestruction[CurPendingObjIndex] = GGCObjectsPendingDestruction[GGCObjectsPendingDestructionCount - 1];
					GGCObjectsPendingDestructionCount--;
				}
				else
				{
					CurPendingObjIndex++;
				}
				...
			}
			...
		}

		if (GGCObjectsPendingDestructionCount == 0)
		{
			// 所有不可达对象都已经走完 FinishDestroy，后面才允许真正 delete。
			GGCObjectsPendingDestruction.Empty(256);
			GObjFinishDestroyHasBeenRoutedToAllObjects = true;
			GObjCurrentPurgeObjectIndexNeedsReset = true;
		}
	}

	if (GObjFinishDestroyHasBeenRoutedToAllObjects)
	{
		if (GObjCurrentPurgeObjectIndexNeedsReset)
		{
			// delete 阶段第一次进入时，先初始化 purge 游标。
			GUObjectPurge.Begin();
			GObjCurrentPurgeObjectIndexNeedsReset = false;
		}

		// 真正删除对象、释放槽位和相关存储是在这里推进的。
		GUObjectPurge.DestroyObjects(bUseTimeLimit, TimeLimit, GCStartTime);
		if (GUObjectPurge.IsFinished())
		{
			// 整轮 purge 彻底结束后，清掉全局 purge 状态，等待下一轮 GC。
			GObjFinishDestroyHasBeenRoutedToAllObjects = false;
			GObjPurgeIsRequired = false;
			GObjCurrentPurgeObjectIndexNeedsReset = true;
			bCompleted = true;
		}
	}
}
```

`IncrementalDestroyGarbage(...)` 分成两段推进销毁：第一段遍历 `GUnreachableObjects`，把已经 `IsReadyForFinishDestroy()` 的对象直接推进到 `ConditionalFinishDestroy()`，其余对象放入 `GGCObjectsPendingDestruction`；第二段持续轮询 pending 列表，直到所有对象都完成 `FinishDestroy()`。只有 `GObjFinishDestroyHasBeenRoutedToAllObjects` 被置为 true 之后，`GUObjectPurge.DestroyObjects(...)` 才会开始真正删除对象、释放槽位和回收存储。