# 地牢随机生成系统 (Dungeon Generation System)

本项目实现了一套基于图论和网格分布的随机地牢生成算法，旨在复现《暗黑地牢》经典的“房间-走廊”网格结构。

---

## 1. 核心数据结构

地牢生成逻辑主要集中在 `DungeonGenerator.cs` 中，使用了两个内部辅助类：

- **GenRoom (房间节点)**: 存储房间在网格中的坐标 (`GridX`, `GridY`) 及其四周的连接通路 (`Left`, `Right`, `Top`, `Bot`)。
- **GenHall (走廊边)**: 存储连接两个 `GenRoom` 的通路状态 (`Exists`) 及其两端的房间引用。

---

## 2. 生成流程分析

地牢生成是一个**约束满足 (Constraint Satisfaction)** 过程，分为以下几个阶段：

### 2.1 网格初始化
系统根据 `DungeonGenerationData` (JSON) 定义的 `GridSizeX/Y` 初始化一个二维网格。
```csharp
int xSize = genData.GridSizeX;
int ySize = genData.GridSizeY;
GenRoom[,] areaGrid = new GenRoom[xSize, ySize];
```

### 2.2 随机游走与节点布置
1. **起始点选择**: 随机选择一个网格点作为入口。
2. **连接扩散**: 从当前房间向四周随机扩散，直到达到 `BaseRoomNumber` (基础房间数) 的限制。
3. **连通性校验**: 确保所有房间都可通过走廊相互到达，避免孤岛节点。

### 2.3 房间类型与内容填充
地牢生成后，系统会对每个房间和走廊分配内容：
- **Room Types**: 入口 (Entrance)、战斗房 (Battle)、奇物房 (Curio)、宝箱房 (Treasure)、BOSS 房。
- **Contents**: 随机刷出怪物组合 (`MonsterMash`)、奇物、陷阱或障碍物。

---

## 3. 约束配置 (DungeonGenerationData)

地牢的大小和复杂度由 JSON 配置文件决定：
- **GridSize**: 定义网格边界（如 5x5, 7x7）。
- **BaseRoomNumber**: 必须生成的最小房间数。
- **BaseCorridorNumber**: 必须生成的最小走廊数。
- **MonsterFrequency**: 不同难度下的怪物遭遇频率。

---

## 4. 关键算法逻辑：先宽搜索 (BFS) 的运用

生成器内部使用类似 BFS 的逻辑来计算每个房间到入口的最短路径 (`MinPath`)，这对于确定 BOSS 房（通常放置在离入口最远的位置）和任务目标分布至关重要。

```csharp
// 伪代码示例：计算最短路径
while(queue.Count > 0)
{
    var current = queue.Dequeue();
    foreach(var neighbor in current.Neighbors)
    {
        if(neighbor.MinPath > current.MinPath + 1)
        {
            neighbor.MinPath = current.MinPath + 1;
            queue.Enqueue(neighbor);
        }
    }
}
```

---

## 5. 地图显示与侦察 (Scouting)

地牢生成器生成的 `Dungeon` 对象会被 `RaidSceneManager` 持有，并通过 UI 渲染在小地图面板上。
- **Knowledge 状态**:
    - `Hidden`: 玩家尚未知晓该节点。
    - `Scouted`: 通过侦察机制揭露了内容（显示图标但颜色暗淡）。
    - `Visited`: 玩家已亲自进入该节点。

---
[返回主页](Index.md)
