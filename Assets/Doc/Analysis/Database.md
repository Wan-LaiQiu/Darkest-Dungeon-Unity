# 数据库与数据加载系统 (Database & Data Loading)

本项目采用了高度解耦的**数据驱动 (Data-Driven)** 设计。游戏中的数值、规则、实体定义几乎全部存储在外部 JSON 和 CSV 文件中，由 `DarkestDatabase` 统一管理。

---

## 1. DarkestDatabase (核心仓库)

`DarkestDatabase` 是项目中所有静态数据的唯一访问点。它在游戏启动时通过 `DarkestDungeonManager` 初始化。

### 1.1 数据分布与目录
数据主要存储在 `Assets/Resources/Data/` 目录下：
- **Heroes/**: 各职业英雄的基础属性与技能定义。
- **Monsters/**: 怪物数值、AI 行为和技能。
- **Inventory/**: 物品、饰品数据。
- **Mechanics/**: 核心机制配置（地牢生成、城镇事件、Buff、效果）。
- **Dungeons/**: 地牢环境配置（背景图、装饰物、掉落表）。

### 1.2 核心加载流程
```csharp
public void Load()
{
    LoadSprites();          // 加载公共精灵图
    LoadJsonUpgrades();     // 加载建筑与英雄升级树
    LoadJsonBuildings();    // 加载城镇建筑配置
    LoadJsonBuffs();        // 加载全局 Buff 定义
    LoadJsonQuirks();       // 加载怪癖与疾病
    LoadEffects();          // 加载战斗效果原子
    LoadJsonHeroClasses();  // 加载英雄职业定义
    // ... 更多数据项
}
```

---

## 2. 数据解析机制 (JSON & Deserialization)

项目使用了自定义的 `JsonDarkestDeserializer` 和 Newtonsoft.Json 库。

### 2.1 英雄职业加载示例
```csharp
private void LoadJsonHeroClasses()
{
    HeroClasses = new Dictionary<string, HeroClass>();
    var heroFolders = Resources.LoadAll<TextAsset>(HeroesDirectory);
    foreach (var file in heroFolders)
    {
        HeroClass heroClass = JsonDarkestDeserializer.DeserializeHeroClass(file.text);
        HeroClasses.Add(heroClass.Id, heroClass);
    }
}
```

### 2.2 效果原子定义 (Effects.txt)
效果通常存储在纯文本或 CSV 中，每行代表一个效果原子的定义。
```text
// 示例：流血效果定义
effect: .id "bleed_1" .type "bleed" .chance 100% .dot_damage 1 .dot_duration 3
```
系统通过 `Regex` 匹配或 `String.Split` 快速解析这些紧凑的定义格式。

---

## 3. 实时访问模式

由于所有数据都存储在 `Dictionary<string, T>` 中，运行时可以通过 ID 快速检索：

```csharp
// 示例：获取特定怪物的配置
var monsterData = DarkestDungeonManager.Data.Monsters["skeleton_soldier"];

// 示例：获取特定的 Buff 定义
var buff = DarkestDungeonManager.Data.Buffs["speed_buff_1"];
```

---

## 4. 为什么要采用数据驱动？

1. **数值平衡**: 调整伤害、血量、概率只需修改 JSON，无需重新编译项目。
2. **内容扩展**: 新增一个英雄职业只需按格式编写 JSON 并提供 Spine 资源。
3. **多语言支持**: 文本 ID 与具体语言包解耦。
4. **存档兼容**: 存档只需记录 ID 和动态状态，静态属性始终从数据库读取。

---
[返回主页](Index.md)
