# 突袭与地牢探索 (Raid & Exploration)

## 1. RaidSceneManager (突袭管理器)
`RaidSceneManager` 是地牢关卡的主要管理器，负责关卡加载、场景切换、任务目标检查和全局事件处理。

- **核心场景状态 (DungeonSceneState)**：
  - 房间 (`Room`)、走廊 (`Hall`)。
- **任务目标处理**：
  - 记录已探索房间数、战斗胜利数、已采集物品等。
  - 完成目标后触发任务完成窗口 (`QuestCompletionWindow`)。
- **全局状态维护**：
  - 光照强度 (`TorchMeter`)、侦察进度 (`Scouting`)、扎营过程 (`Camping`)。
  - 维护 判定队列 (ResolveCheck, HeartAttack, DeathDoorEnter)。

## 2. 地牢与生成 (Dungeon & DungeonGenerator)
地牢是以节点（房间）和边（走廊）构成的图。

- **房间 (DungeonRoom)**：
  - 包含 战斗、奇物 (`Curio`)、障碍物、门。
  - 房间知识状态 (`Knowledge`): 隐藏、已侦察、已访问。
- **走廊 (Hallway / HallSector)**：
  - 由多个 扇区 (`Sector`) 组成。
  - 包含 战斗、奇物、陷阱、障碍物、饥饿检查点 (`HungerCheck`)。
- **地牢生成器 (DungeonGenerator)**：
  - 根据 `DungeonGenerationData` 生成随机地牢。
  - 管理房间连接、关键目标放置、资源（饰品/传家宝）分布。

## 3. 队伍控制 (RaidPartyController)
负责玩家在房间和走廊中的移动表现。

- **机制**：
  - 在走廊中采用横版卷轴移动方式。
  - 处理 队伍排位 (`Ranks`) 和 队伍灯光/摄像机焦点。
  - 触发 陷阱判定、饥饿判定 和 随机战斗加载。

## 4. 探索机制
- **光照 (TorchMeter)**：
  - 消耗 火把 (`Torch`)。
  - 光照水平（100-75, 74-51, 50-26, 25-1, 0）。
  - 低光照增加压力、怪物伤害和掉落，高光照增加侦察概率、英雄命中和怪物惊喜概率。
- **侦察 (Scouting)**：
  - 进入新房间或走廊扇区时可能触发。
  - 揭示地图节点内容（战斗、奇物、陷阱）。
- **扎营 (Camping)**：
  - 在特定长度的任务中使用 薪柴 (`Firewood`) 触发。
  - 进食阶段：消耗食物，根据摄入量恢复生命或承受压力损害。
  - 技能阶段：消耗扎营点数 (`TimeCost`) 使用各职业的扎营技能，提供长期 Buff、压力缓解或额外侦察。

---
[返回主页](Index.md)
