
- Actor是如何BeginPlay的？
- 服务端创建的Actor同步到客户端后如何BeginPlay？
- 部分核心的Actor类的BeginPlay顺序，比如GameState与Character谁先BeginPlay？

#### 各类Actor
![[Actor的BeginPlay时序剖析 2026-09-10 15.10.21.excalidraw|800]]
![[Actor的BeginPlay时序剖析 2026-09-10 15.18.21.excalidraw|800]]
#### Actor生命周期
![[Pasted image 20260910153450.png]]
##### GameMode组件初始化
![[Actor的BeginPlay时序剖析 2026-09-10 16.04.56.excalidraw|800]]
##### PlayerContorller组件初始化
![[Actor的BeginPlay时序剖析 2026-09-10 16.23.29.excalidraw|800]]

#### 启动流程

##### Standalone
![[Actor的BeginPlay时序剖析 2026-09-10 17.07.27.excalidraw|800]]
##### DedicateServer
- **Server**
![[Pasted image 20260910171102.png]]
- **Client**
![[Pasted image 20260910171138.png]]
![[Pasted image 20260910171159.png]]
![[Pasted image 20260910171216.png]]
![[Pasted image 20260910171417.png]]
`AGameStateBase::HandleBeginPlay`是由GameMode来调用的，而客户端不存在GameMode，也就是说客户端无法在LoadMap流程里BeginPlay！那客户端到底是怎么BeginPlay的呢？冷静思考一下，虽然客户端没有GameMode，但是有GameState呀，而且服务端为什么要绕到GameState来处理BeginPlay呢？也许答案就藏在GameState里，于是跟着源码果然发现了线索：GameState里有一个可复制的变量bReplicatedHasBegunPlay，它有一个同步回调函数`OnRep_ReplicatedHasBegunPlay`，逻辑竟然与`AGameStateBase::HandleBeginPlay`一模一样！大概读者已经能猜到了，当GameState的这个属性同步下来时就会触发客户端的BeginPlay，也就是说客户端的BeginPlay依赖于GameState的属性同步的时机。
![[Pasted image 20260910171521.png]]![[Pasted image 20260910171526.png]]

- **ListenServer**
![[Pasted image 20260910171615.png]]

### BeginPlay时序

- World->BeginPlay之前创建的Actor(静态资源类 or 动态创建)，按World中的Actor的顺序BeginPlay。这里的顺序是经SortActorList排序后的顺序，对应上文Standalone模式里提到的排序规则
- World->BeginPlay后动态创建的Actor，立即BeginPlay。动态创建的Actor通过PostActorConstruction方法触发组件初始化的流程，同时会判断World是否已经BeginPlay了，是的话就让自己直接BeginPlay
- Actor的Component比Actor自身更早BeginPlay。

## Standalone

对Standalone模式来说既是客户端也是服务端，所以在LoadMap流程中会把GameMode及PlayerController这些类都创建出来，最后整个World统一BeginPlay，将这个过程简化一下：

- LoadPackage加载资源(静态资源Actor)
    
- SetGameMode：创建GameMode
    
- PreInitializeComponents(GameMode)：创建GameState
    
- Login：创建PlayerController
    
- PostInitializeComponents(PlayerController)：创建PlayerState
    
- PostLogin：创建Character
    
- World开启BeginPlay
    

所有的Actor都在World开启BeginPlay之前创建出来，所以这些Actor的BeginPlay时序：

- GameMode->BeginPlay
    
- GameState->BeginPlay
    
- PlayerController->BeginPlay
    
- PlayerState->BeginPlay
    
- Character->BeginPlay

## DedicateServer

由于DS模式下服务端与客户端LoadMap的流程不同，并且除GameMode外其他的Actor是由服务端同步给客户端的，所以这些Actor的BeginPlay时序在服务端与客户端有较大的不同。

**Server**

刚启动Server进程时，由于客户端未连入不会创建玩家的链接和角色，此时服务端只有GameMode和GameState：

- LoadPackage加载资源(静态资源Actor)
    
- SetGameMode：创建GameMode
    
- PreInitializeComponents(GameMode)：创建GameState
    
- World开启BeginPlay
    

即只有GameMode和GameState会开始BeginPlay：

- GameMode->BeginPlay
    
- GameState->BeginPlay
    

当客户端登录成功后，服务端会创建PlayerController、PlayerState与Character，此时服务端的World已经BeginPlay了，所以这些Actor一创建出来就会BeginPlay。同时有一点要注意，PlayerState是作为PlayerController的组件创建出来的，所以PlayerState比PlayerController更早BeginPlay：

- Login：创建PlayerController
    
- PostInitializeComponents(PlayerController)：创建PlayerState
    
- PlayerState->BeginPlay
    
- PlayerController->BeginPlay
    
- PostLogin：创建Character
    
- Character->BeginPlay
    

**Client**

客户端的PlayerController、PlayerState和Character依赖网络复制，当Actor同步下来后再创建到当前World中，而网络复制的无序性导致无法确定这些Actor的创建顺序，另外上文也解释了客户端的BeginPlay取决于GameState的属性同步的时机，所以客户端的BeginPlay时序具有随机性。下面以其中一种复制顺序为例展示对应的BeginPlay时序：

- 复制PlayerController下来并创建
    
- 复制Character下来并创建
    
- 复制PlayerState下来并创建
    
- 复制GameState下来并创建
    

示例中GameState是最后复制下来的，而GameState的属性同步触发World的BeginPlay，所以这种情况下的BeginPlay时序：

- PlayerController->BeginPlay
    
- Character->BeginPlay
    
- PlayerState->BeginPlay
    
- GameState->BeginPlay
    

再次声明，上边的示例只是网络复制无序性下的一种可能场景，假如GameState是第二个复制下来的Actor，那么BeginPlay的时序也会相应发生变化。