### 触发入口
![[Level Travel 2026-09-09 16.20.49.excalidraw|800]]
### Hard Travel

#### UEngine::loadMap()

- 旧世界清理
![[Level Travel 2026-09-09 17.21.28.excalidraw|600]]
- 新世界加载
![[Level Travel 2026-09-09 17.39.42.excalidraw|500]]
- 新世界初始化与开始运行
![[Level Travel 2026-09-10 09.38.42.excalidraw|800]]
### Seamless Travel

无缝旅行是为多人游戏设计的不断开网络连接的地图切换方案；通过FSeamlessTravelHandler结构处理切换状态机。

![[Level Travel 2026-09-10 10.46.07.excalidraw|800]]
#### 相关类图
![[Level Travel 2026-09-10 11.06.10.excalidraw|800]]
#### 无缝旅行 vs 非无缝旅行
|特性|非无缝旅行 (Hard Travel)|无缝旅行 (Seamless Travel)|
|---|---|---|
|**网络连接**|断开重连|保持连接|
|**Actor 迁移**|全部销毁|可选择性保留|
|**过渡地图**|无|有（TransitionMap）|
|**加载方式**|同步 `LoadPackage`|异步 `LoadPackageAsync`|
|**阻塞性**|阻塞主线程|非阻塞，分帧完成|
|**适用场景**|单机/首次加入服务器|多人游戏换图|
|**GameMode**|重新创建|可迁移保留|
|**PlayerController**|重新创建|迁移保留|
#### 注意事项与最佳实践

1. **切图期间是阻塞的**：`LoadMap()` 是同步执行的，会冻结主线程。建议使用 Loading Screen 或 Movie 遮挡。  
2. **`GameInstance` 是唯一不被销毁的对象**：它在整个游戏生命周期存在，是跨关卡传递数据的最佳载体。  
3. **多人游戏优先使用 Seamless Travel**：避免断开网络连接导致的重连延迟。需要在 `GameMode` 中设置 `bUseSeamlessTravel = true`。  
4. **配置过渡地图**：在项目设置 → Maps & Modes → `TransitionMap` 中配置，无缝旅行需要它作为中间缓冲。  
5. **Actor 持久化**：在无缝旅行中，重写 `GetSeamlessTravelActorList()` 来指定需要迁移的 Actor。  
6. **`TRAVEL_Absolute` vs `TRAVEL_Relative`**：Absolute 会完全重置 URL（包括 Options），Relative 会保留之前的 Options 并追加新的。