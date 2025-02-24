# 虚幻Tick

## 核心类

Tick系统涉及到以下几个核心类；

### FTickFunction

`TickFunction`是Tick系统管理、调度、执行的基本对象；

如果目标类需要Tick，则通用的做法是继承`FTickFunction`，实现一个特殊的子类，并让其作为目标类的成员变量，并在必要时注册它到Tick系统中；

`AActor`实现了`FActorTickFunction`，对应的成员变量为`PrimaryActorTick`；`UActorComponent`实现了`FActorComponentTickFunction`，对应的成员变量为`PrimaryComponentTick`；这两个类重载了`FTickFunction::ExecuteTick`函数，这个函数会在合适的时机被Tick系统调用，进而调用`AActor::Tick`和`UActorComponent::TickComponent`，执行实际的Tick逻辑；

### FTickTaskLevel

`TickTaskLevel`是`ULevel`在Tick系统内的唯一对应对象；

`TickTaskLevel`维护它所对应的`ULevel`内，所有`TickFunction`的列表，不同的`TickTaskLevel`相互独立；

这样的划分方式的好处是显而易见的，即当发生`ULevel`的整体卸载时，只需要把`TickTaskLevel`排除在外，所有其下的`TickFunction`都不会再执行，效率比较高；除此之外，较短的`TickFunction`列表也能提升调度效率；

`FTickTaskLevel`维护多种列表，对应`TickFunction`的不同状态，后面讲调度时会详细解释；

### FTickTaskSequencer

`TickFunction`按照执行时机位于物理模拟的阶段，被划分为了多个TickGroup；

```cpp
enum ETickingGroup
{
    TG_PrePhysics,
    TG_StartPhysics,
    TG_DuringPhysics,
    TG_EndPhysics,
    TG_PostPhysics,
    TG_PostUpdateWork,
    TG_LastDemotable,
    TG_NewlySpawned,
    TG_MAX,
}
```

`Sequencer`这个名字初看也许有些迷惑，它其实就是保证了不同TickGroup内的任务执行的先后顺序，对多线程任务进行同步，所以它实际承担的是执行器的工作；

### TaskGraph

`TaskGraph`是一个单例对象，它提供了一套支持依赖管理机制和异步机制的任务执行系统，在引擎内部被广泛使用，和Tick系统独立；

`TaskGraph`能够使用的线程包括`NamedThread`和`WorkerThread`，其中`NamedThread`就是`GameThread`, `RenderThread`, `RHIThread`这些，`WorkerThread`则是由`TaskGraph`创建，专用于处理异步任务的线程；

`TaskGraph`的实现较为复杂，我们这里只关注Tick系统使用到的接口；

- `FTaskGraphInterface::QueueTask` 把任务放入指定线程的队列里等待执行，对于`TickFunctionTask`来说就是`GameThread`或者`WorkerThread`；
- `FTaskGraphImplementation::WaitUntilTasksComplete` 传入一个任务列表，让指定线程处理这些任务直至全部处理完毕；
- `FTaskGraphImplementation::ProcessThreadUntilIdle` 指定线程处理队列内的所有任务，直至处理完毕；

后面`TickGraph`小节会大致讲一下相关实现原理；

### FTickTaskManager

`TickTaskManager`是一个单例对象，它驱动整个Tick系统的运行；

Tick系统和`TaskGraph`是单向引用关系，基本上就是Tick系统把`TickFunction`包装为`TickFunctionTask`，然后交付给`TaskGraph`执行；

![[TickSystem.drawio.png]]

## 执行流程

每一帧，引擎Loop都会对每个`UWorld`执行`UWorld::Tick`，这个函数就是Tick系统的执行流程，具体调用流程如下：

```
UWorld::Tick
    FTickTaskManager::StartFrame
        Set TickContext // 设置当前帧的Tick上下文，上下文会被其他函数共享
        FTickTaskSequencer::StartFrame // 重置TickTaskSequencer的状态
        FTickTaskManager::FillLevelList // 收集需要Tick的TickTaskLevel写入LevelList，仅考虑当前World内Visable的关卡
        if !bConcurrentQueue // 如果不启用并发入队(单线程模式)
            For each TickTaskLevel:
                FTickTaskLevel::StartFrame // 调度TickFunction，决定哪些TickFunction需要在当前帧执行
            For each TickTaskLevel:
                FTickTaskLevel::QueueAllTicks // 入队所有TickFunctions
                    FTickFunction::QueueTickFunction
                        FTickTaskSequencer::QueueTickTask
                            FTickTaskSequencer::StartTickTask
                                TGraphTask::CreateTask // 创建TGraphTask<FTickFunctionTask>
                                TGraphTask::FConstructor::ConstructAndHold
                                    placement new构造TickFunctionTask
                                    TGraphTask::Hold // Hold住这个Task，暂时不执行
                            FTickTaskSequencer::AddTickTaskCompletion 
                            // 把Task记录到HiPriTickTasks或TickTasks列表，把Task的CompletionEvent记录到TickCompletionEvents列表

        else // 如果启用并发入队
            For each TickTaskLevel:
                FTickTaskLevel::StartFrameParallel // 调度TickFunction，把需要执行的TickFunction放入AllTickFunctions列表
            ParallelFor each TickFunction:
                FTickFunction::QueueTickFunctionParallel
                    FTickTaskSequencer::QueueTickTaskParallel
    FTickTaskManager::RunTickGroup
        FTickTaskSequencer::ReleaseTickGroup
            FTickTaskSequencer::DispatchTickGroup
                For each HiPriTickTask: // 高优先级Task
                    TGraphTask::Unlock // 解锁这个Task
                        FBaseGraphTask::ConditionalQueueTask
                            FBaseGraphTask::QueueTask
                                FTaskGraphImplementation::QueueTask
                                    FNamedTaskThread::EnqueueFromThisThread // 把TGraphTask<FTickFunctionTask>放入对应线程的队列
                For each TickTask: // 低优先级Task
                    TGraphTask::Unlock // 解锁这个Task
            if bBlockTillComplete || SingleThreadedMode: // 如果不是TG_DuringPhysics阶段，或者为单线程模式
                FTaskGraphImplementation::WaitUntilTasksComplete // 等待当前Group及之前所有Group的TGraphTask<FTickFunctionTask>完成
            else: // 如果是TG_DuringPhysics阶段，并且为多线程模式
                FTaskGraphImplementation::ProcessThreadUntilIdle // 只等待所有GameThread的任务执行完毕，不等待物理异步任务
        TickGroup++
    FTickTaskManager::EndFrame
        FTickTaskSequencer::EndFrame
        For each TickTaskLevel:
            FTickTaskLevel::EndFrame // 再次调度TickFunction
        Reset TickContext
        Reset LevelList
```


对于`Dedicated Server`，Tick系统不启用多线程，整个流程都运行在`GameThread`内；

## TickFunction生命周期

`TickFunction`的生命周期基本上是和其Game对象绑定的，这里以`AActor`举例；

### 注册

Actor基类的BeginPlay函数会主动注册自己的`TickFunction`；

```
AActor::BeginPlay
    AActor::RegisterAllActorTickFunctions
        AActor::RegisterActorTickFunctions
            FTickFunction::RegisterTickFunction
```

如果勾选了`bAllowTickBeforeBeginPlay`，则会在组件注册之前就注册`TickFunction`，不再等待BeginPlay；

```
AActor::IncrementalRegisterComponents
    AActor::RegisterAllActorTickFunctions
        AActor::RegisterActorTickFunctions
            FTickFunction::RegisterTickFunction
```

在Actor销毁时，`UWorld`会主动解除`TickFunction`的注册；

```
UWorld::DestroyActor
    AActor::RegisterAllActorTickFunctions
        AActor::RegisterActorTickFunctions
            FTickFunction::UnRegisterTickFunction
```

### 启用

如果勾选了`bStartWithTickEnabled`，则会在注册`TickFunction`之前就调用`SetTickFunctionEnable`，把`TickState`设置为`Enabled`；

```cpp
PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartWithTickEnabled || PrimaryActorTick.IsTickFunctionEnabled());
```

如果希望注册后再启用，则需要手动调用启用的接口`AActor::SetActorTickEnabled`；

在这种情况下，考虑到`TickFunction`已经处于`TickTaskLevel`的列表内，所以需要先`Remove`再`Add`，来保证`TickTaskLevel`内部状态的正确性；

```cpp
void FTickFunction::SetTickFunctionEnable(bool bInEnabled)
{
	if (bRegistered && (bInEnabled == (TickState == ETickState::Disabled)))
	{
		check(TickTaskLevel);
		TickTaskLevel->RemoveTickFunction(this); // 从内部列表移除TickFunction
		TickState = (bInEnabled ? ETickState::Enabled : ETickState::Disabled);
		TickTaskLevel->AddTickFunction(this); // 把TickFunction加回内部列表
	}
    ...
}
```

### NewlySpawned

考虑到`TickFunction`执行时可能会Spawn新的Actor，这时就需要注册这个新Actor的`TickFunction`，新的`TickFunction`至少要等到当前Tick流程走完，才能被调度并执行，所以针对这种情况，需要一些特殊的处理；

`FTickTaskLevel`上有一个标志位`bTickNewlySpawned`，用于区分当前是否正在执行Tick流程。这个标志位在`FTickTaskLevel::StartFrame`时被设置为`true`，在`FTickTaskLevel::EndFrame`时被设置为`false`；

在标志位`bTickNewlySpawned`为`true`时，新`TickFunction`会被额外地添加到一个`NewlySpawnedTickFunctions`列表内缓存下来；

```cpp
void AddTickFunction(FTickFunction* TickFunction)
{
    if (TickFunction->TickState == FTickFunction::ETickState::Enabled)
    {
        AllEnabledTickFunctions.Add(TickFunction);
        if (bTickNewlySpawned)
        {
            NewlySpawnedTickFunctions.Add(TickFunction); // 把TickFunction缓存到NewlySpawned列表
        }
    }
    ...
}
```

在`FTickTaskManager::RunTickGroup`中，通过`ReleaseTickGroup`完成当前`TickGroup`的任务后，函数还会遍历所有`NewlySpawnedTickFunctions`列表，调用`QueueNewlySpawned`创建它们的任务并记录到列表。到了下一次`ReleaseTickGroup`时，这些任务就会被一起执行；

在处理`NewlySpawned`任务时，可能又会产出新的`NewlySpawnedTickFunctions`，因此函数使用一个带有最大迭代次数的大循环来循环处理，默认设置的最大次数为101次，超出这个次数的`TickFunction`会被丢弃；

```cpp
for (int32 Iterations = 0;Iterations < 101; Iterations++)
{
    int32 Num = 0;
    for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
    {
        Num += LevelList[LevelIndex]->QueueNewlySpawned(Context.TickGroup); // 创建所有NewlySpawnedTickFunctions的Task并记录它们
    }
    if (Num && Context.TickGroup == TG_NewlySpawned)
    {
        TickTaskSequencer.ReleaseTickGroup(TG_NewlySpawned, true); // 在TG_NewlySpawned Group循环处理剩余的NewlySpawnedTickFunctions
    }
    else
    {
        bFinished = true; // 没有剩余的NewlySpawnedTickFunctions了，跳出循环
        break;
    }
}
```

## TickFunction调度

调度逻辑主要由`TickTaskLevel`实现；

除去上面提到过的`NewlySpawned`，`TickTaskLevel`共维护四个列表来管理`TickFunction`的状态；

```cpp
/** Primary list of enabled tick functions **/  
TSet<FTickFunction*>                  AllEnabledTickFunctions;  
/** Primary list of enabled tick functions **/  
FCoolingDownTickFunctionList            AllCoolingDownTickFunctions;  
/** Primary list of disabled tick functions **/  
TSet<FTickFunction*>                  AllDisabledTickFunctions;  
/** Utility array to avoid memory reallocations when collecting functions to reschedule **/  
TArrayWithThreadsafeAdd<FTickScheduleDetails>           TickFunctionsToReschedule;
```

- `AllEnabledTickFunctions` 哈希表，一定会在下一帧被执行的`TickFunction`
- `AllCoolingDownTickFunctions` 链表，冷却中的`TickFunction`，冷却时间结束才会被执行
- `AllDisabledTickFunctions` 哈希表，不参与调度的`TickFunction`
- `TickFunctionsToReschedule` 线程安全数组，需要重新调度的`TickFunction`

顺带一提，`FTickFunction`也有一个枚举成员`TickState`，包括`Disabled`, `Enabled`, `CoolingDown`三种状态；但是这里的状态和上面四个列表所表达的状态不是一个语义；

### 状态跳转

![[TickStates.drawio.png]]

- 新的`TickFunction`注册后必定是`Enabled`或者`Disabled`状态
- `Enabled`状态的`TickFunction`，如果`TickInterval > 0`，则会在`FTickTaskLevel::QueueAllTicks`阶段入队等待执行，变为`ToReschedule`状态，等待调度；如果`TickInterval <= 0`则说明每帧都需要Tick，`TickFunction`会一直留在`Enabled`状态，且每帧都会入队；
- `ToReschedule`状态的`TickFunction`，在当前帧的Tick流程结束(`FTickTaskLevel::EndFrame`)时，会通过`ScheduleTickFunctionCooldowns`函数，进入`CoolDown`状态；
- `CoolDown`状态的`TickFunction`，如果冷却时间已结束，则会在下一次`FTickTaskLevel::QueueAllTicks`阶段入队后，重新变为`ToReschedule`状态，等待调度；
- 处于任意状态的`TickFunction`，如果调用了`SetTickFunctionEnable(false)`，都会变为`Disabled`状态，不再参与调度；

### CoolingDown链表

CoolingDown链表记录每个结点相对上一个结点的相对冷却时间，而不是记录每个结点自己的总冷却时间。从而在更新冷却时间时，不必更新每个结点，而只需更新链表头部结点。同样的，如果需要读取某个结点的冷却时间，则需要累加它及之前所有结点的相对冷却时间；

以下图为例，每次时间增长时，只需依次弹出链表头部的所有到期结点等待执行，并修改第一个非到期结点，就能以较小的代价更新整个链表；

![[CoolingDownLinkedList.drawio.png]]

### Schedule算法

`ScheduleTickFunctionCooldowns`函数会统一把所有`TickFunctionsToReschedule`内的`TickFunction`合入`AllCoolingDownTickFunctions`链表，具体流程分为两步；

1. 函数按照冷却时间对所有`ToReschedule`的`TickFunction`排序
2. 把`ToReschedule`和`CoolDown`链表合并

合并流程其实就是简单的双指针Merge，下图演示了一次完整的合并流程；

![[Reschedule.png]]

合并完毕后，`TickFunctionsToReschedule`会被清空，不会带到下一帧；

## TaskGraph

上面提到，Tick系统把所有当前帧需要执行的`TickFunction`包装为Task，入列`TaskGraph`执行。绕这个弯路的根本目的是复用`TaskGraph`的依赖管理机制和异步机制，这里就针对这两个特性挖掘一下`TaskGraph`的实现；

### 依赖处理机制

Tick系统支持指定为`TickFunction`指定`Prerequisites`，也就是可以保证当前`TickFunction`执行前，一定已经执行了某些指定的其他`TickFunction`。这个特性被游戏业务大量使用，如`ActorComponent`的Tick会等待其`ParentActor`的Tick，`SceneComponent`的Tick会等待其`AttachParent`的Tick，etc；

依赖处理机制离不开`FGraphEvent`这个辅助类。任务类`TGraphTask`中有一个成员变量`Subsequents`，它的类型是`FGraphEventRef`，可以理解为是这个任务专属的一个`FGraphEvent`的指针。`Subsequents`的语义是所有指定了本Task为`Prerequisites`的Tasks，在初始化Task时，`TGraphTask::SetupPrereqs`会遍历所有`Prerequisites`，并把本Task添加到它们的`Subsequents`中；

```cpp
void SetupPrereqs(const FGraphEventArray* Prerequisites, ENamedThreads::Type CurrentThreadIfKnown, bool bUnlock)  
{  
	...
	for (int32 Index = 0; Index < Prerequisites->Num(); Index++)  
	{         
		FGraphEvent* Prerequisite = (*Prerequisites)[Index];  
		if (Prerequisite == nullptr || !Prerequisite->AddSubsequent(this))  
		{
			AlreadyCompletedPrerequisites++;         
		}      
	}
	...
}
```

下图描述了一种依赖关系，`GraphTask_3`设置了`GraphTask_1`和`GraphTask_2`为`Prerequisites`，因此`GraphTask_1`的`Subsequents`会包含`GraphTask_3`；

![[GraphTask.drawio.png]]

Task执行完毕后，会主动通知`Subsequents`，调用它们的`ConditionalQueueTask`；

```
TGraphTask::ExecuteTask
	FGraphEvent::DispatchSubsequents
		FBaseGraphTask::ConditionalQueueTask
```

`ConditionalQueueTask`会尝试自减一个计数器`NumberOfPrerequistitesOutstanding`，如果计数器变为0，则说明所有`Prerequisites`都执行完毕了，现在可以放心地把任务入列到指定线程等待执行；

```cpp
void ConditionalQueueTask(ENamedThreads::Type CurrentThread, bool& bWakeUpWorker)  
{  
   if (NumberOfPrerequistitesOutstanding.Decrement()==0)  
   {
	    QueueTask(CurrentThread, bWakeUpWorker);  
	    bWakeUpWorker = true;  
   }
}
```


`NumberOfPrerequistitesOutstanding`的初始值是任务的前置依赖总数+1，这里多出的1用于给任务上锁；

如果使用`ConstructAndDispatchWhenReady`创建任务，则在构造时就会减去这个1，一旦依赖为0就会执行任务；

而如果使用`ConstructAndHold`创建任务，则必须手动调用`Unlock`减去这个1，即使依赖为0，不调用`Unlock`的话任务也不会执行。Tick系统正是利用了这个特性来控制不同`TickGroup`的执行顺序；

### 异步机制

`TaskGraph`通过创建`WorkerThread`来执行异步任务。如果标记`TickFunction`的`bRunOnAnyThread`标记为`true`，产出的对应`TickFunctionTask`的`ThreadToExecuteOn`就会是`AnyThread`，否则为`GameThread`；

对于`AnyThread`类型的任务，在入队时，`TaskGraph`会把它放入一个叫做`IncomingAnyThreadTasks`的队列，在这个队列存放的任务被所有`WorkerThread`共享。

`WorkerThread`的执行逻辑其实是非常简单的，它其实就是一个While循环，不断从队列中取出新的任务并执行。如果没有新的任务了，就把当前`WorkerThread`阻塞掉，等待新任务到来时被主动唤醒，就像只会工作和睡觉的996机器；

```cpp
// CallStack
// FTaskThreadAnyThread::Run
// 	FTaskThreadBase::Run
// 		FTaskThreadAnyThread::ProcessTasksUntilQuit
// 			FTaskThreadAnyThread::ProcessTasks

// 这里是经过简化后的 FTaskThreadAnyThread::ProcessTasks
uint64 ProcessTasks()
{
    uint64 ProcessedTasks = 0;
    verify(++Queue.RecursionGuard == 1);
    while (1)
    {
        FBaseGraphTask* Task = FindWork(); // Pop一个Task
        if (!Task)
        {
            const bool bIsMultithread = FTaskGraphInterface::IsMultithread();
            if (bIsMultithread)
            {
                Queue.StallRestartEvent->Wait(MAX_uint32, bCountAsStall); // 放弃cpu等待唤醒
                bDidStall = true;
            }
            if (Queue.QuitForShutdown || !bIsMultithread) // 退出线程
            {
                break;
            }
            continue;
        }

        Task->Execute(NewTasks, ENamedThreads::Type(ThreadId), true); // 执行任务
        ProcessedTasks++;
    }
    verify(!--Queue.RecursionGuard);
    return ProcessedTasks;
}
```

对于`GameThread`，它不是`TaskGraph`创建的，而是通过`FTaskGraphImplementation::AttachToThread`挂载到`TaskGraph`里的，因此相比`WorkerThread`有很大的不同；

首先，对于`GameThread`这一类`FNamedTaskThread`，每个线程有自己的两个队列，`MainQueue`和`LocalQueue`。使用额外的`LocalQueue`的主要目的是方便任务的递归创建，即允许`MainQueue`的任务`GameThreadTask_1`在执行时，创建新的任务`GameThreadTask_2`到`LocalQueue`里。Tick系统的所有任务都在`MainQueue`里；

其次，`FNamedTaskThread`不会主动执行任务，而是需要在线程的合适时机调用对应接口执行任务，具体来说就是`WaitUntilTasksComplete`和`ProcessThreadUntilIdle`这两个函数；

`WaitUntilTasksComplete`的语义是"等待指定的一批任务完成"。它以指定的任务列表作为`Prerequisites`，构造一个`FReturnGraphTask`的任务，然后调用`ProcessThreadUntilRequestReturn`函数；

```cpp
TGraphTask<FReturnGraphTask>::CreateTask(&Tasks, CurrentThread).ConstructAndDispatchWhenReady(CurrentThread);  
ProcessThreadUntilRequestReturn(CurrentThread);
```

`FReturnGraphTask`会在指定的任务列表都执行完毕后才开始执行，在`FReturnGraphTask::DoTask`，会主动调用`FTaskGraphInterface::Get().RequestReturn`，设置`QuitForReturn`标志位为`true`；

```cpp
void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)  
{  
   checkThreadGraph(ENamedThreads::GetThreadIndex(ThreadToReturnFrom) == ENamedThreads::GetThreadIndex(CurrentThread)); // we somehow are executing on the wrong thread.  
   FTaskGraphInterface::Get().RequestReturn(ThreadToReturnFrom);  
}
```

这时`ProcessThreadUntilRequestReturn`函数就可以从循环里跳出，正常返回到`GameThread`，我们的所有任务也执行完毕了；

`ProcessThreadUntilIdle`更简单，就是不断从队列里取出新的任务并执行，直到所有任务执行完毕；
