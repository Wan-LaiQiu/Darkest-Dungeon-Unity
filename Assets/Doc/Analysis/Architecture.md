# 核心架构与系统 (Core Architecture & Systems)

本项目采用了**单例驱动 (Singleton-Driven)** 和 **数据驱动 (Data-Driven)** 的架构，通过一个全局的核心管理器协调各个子系统的运作。

---

## 1. DarkestDungeonManager (核心入口)

`DarkestDungeonManager` 是整个项目的控制中心，挂载在 `DontDestroyOnLoad` 的 GameObject 上。

### 1.1 单例访问模式
```csharp
public class DarkestDungeonManager : MonoBehaviour
{
    public static DarkestDungeonManager Instanse { get; private set; }
    
    // 全局快捷访问器
    public static Campaign Campaign => Instanse.campaign;
    public static DarkestDatabase Data => Instanse.database;
    public static RaidManager RaidManager => Instanse.RaidingManager;
}
```

### 1.2 材质与状态管理
管理器还统一持有了用于处理 UI 状态的材质集，这是实现暗黑风格视觉效果（如置灰、高亮）的关键：
- `sharedHighlight`: 选中项的高亮发光效果。
- `sharedGrayscale`: 未激活或禁用项的黑白效果。
- `sharedDeactivated`: 暗淡效果。

---

## 2. DarkestDatabase (数据仓库)

项目的大量静态数据都存储在 JSON/CSV 中，并通过 `DarkestDatabase` 进行统一加载。

### 2.1 延迟初始化与加载
```csharp
public void Load()
{
    // 按照依赖顺序依次加载
    LoadSprites();          // 加载公共资源
    LoadJsonUpgrades();     // 加载升级树
    LoadJsonBuildings();    // 加载城镇建筑
    LoadJsonHeroClasses();  // 加载英雄数据
    // ...
}
```

---

## 3. SaveLoadManager (存档持久化)

负责将玩家的 `Campaign` 进度序列化为二进制或 JSON 文件。

### 3.1 存档结构 (SaveCampaignData)
```csharp
public class SaveCampaignData
{
    public int CurrentWeek;                 // 当前周数
    public List<SaveHeroData> Heroes;      // 所有英雄的详细状态
    public List<string> CompletedPlot;     // 已完成的剧情 ID
    public Dictionary<string, int> EstateCurrencies; // 庄园货币
}
```

---

## 4. 关键系统交互流程

1. **初始化阶段**: `DarkestDungeonManager` 启动并调用 `DarkestDatabase.Load()`。
2. **战役选择**: 玩家进入 `CampaignSelectionWindow`，通过 `SaveLoadManager` 加载存档。
3. **城镇阶段 (Estate)**: 玩家在城镇中通过 `Campaign` 对象管理资源和英雄。
4. **探险准备**: 玩家在 `QuestSelectionWindow` 选择任务并配置队伍。
5. **探险阶段 (Raid)**: 加载地牢场景，`RaidSceneManager` 接管战斗与探索逻辑。
6. **结算与返回**: 任务结束后，更新 `Campaign` 数据并保存，返回城镇场景。

---
[返回主页](Index.md)
