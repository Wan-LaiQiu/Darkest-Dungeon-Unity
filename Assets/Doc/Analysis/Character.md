# 角色与属性系统 (Character & Attributes)

本文档深入分析了 Darkest Dungeon Unity 项目中的角色架构、属性计算、Buff 机制以及英雄与怪物的差异化实现。

---

## 1. Character 基类架构

`Character` 是所有战斗实体的核心，通过组合而非继承的方式集成了大量的属性和状态修饰符。

### 1.1 核心属性访问器
```csharp
public class Character
{
    // 基础战斗数值通过 GetSingleAttribute 动态计算（包含 Buff 修正）
    public float Speed => Mathf.Clamp(GetSingleAttribute(AttributeType.SpeedRating).ModifiedValue, 0, float.MaxValue);
    public float Crit => Mathf.Clamp(GetSingleAttribute(AttributeType.CritChance).ModifiedValue, 0, 1);
    public float Accuracy => Mathf.Clamp(GetSingleAttribute(AttributeType.AttackRating).ModifiedValue, -1, 2);
    public float Dodge => GetSingleAttribute(AttributeType.DefenseRating).ModifiedValue;
    public float Protection => Mathf.Clamp(GetSingleAttribute(AttributeType.ProtectionRating).ModifiedValue, 0, 1);
    
    // 生命值与压力值使用 PairedAttribute (Current/Max)
    public PairedAttribute HitPoints { get; set; }
    public PairedAttribute Stress { get; set; }
}
```

### 1.2 战斗状态标记
`Character` 类中包含大量的虚属性，用于描述实体的当前状态：
- `AtDeathsDoor`: 是否处于死亡之门（仅英雄）。
- `IsStressed`: 压力是否超过 0。
- `IsOverstressed`: 压力是否达到 100（触发判定线）。
- `IsVirtued / IsAfflicted`: 是否已获得美德或折磨特质。
- `IsMonster`: 用于快速区分英雄与怪物逻辑。

---

## 2. 属性系统 (Attribute System)

项目使用了高度解耦的属性类来管理数值及其变动。

### 2.1 SingleAttribute (单值属性)
用于速度、闪避、伤害等。
- **BaseValue**: 基础值（来自 JSON 配置）。
- **BuffValue**: 所有生效 Buff 的累加值。
- **ModifiedValue**: `BaseValue + BuffValue`。

### 2.2 PairedAttribute (成对属性)
用于生命值和压力值。
- **CurrentValue**: 当前剩余量。
- **MaximumValue**: 动态计算的最大上限。

---

## 3. Buff 与 规则系统 (Buff & Rules)

Buff 是游戏机制的核心，通过 `BuffInfo` 进行管理。

### 3.1 Buff 数据结构
```csharp
public class Buff
{
    public string Id { get; set; }
    public BuffType Type { get; set; }              // Stat (属性), Resistance (抗性), ResolveXP (经验)
    public AttributeType AttributeType { get; set; } // 影响的具体属性
    public float ModifierValue { get; set; }         // 修正值 (例如 0.1 表示 +10%)
    
    public BuffRule RuleType { get; set; }           // 触发规则
    public string RuleData { get; set; }             // 规则参数 (如特定怪物类型 ID)
    
    public BuffDurationType DurationType { get; set; } // 持续时间类型 (Combat, Round, Permanent)
    public int DurationAmount { get; set; }            // 持续数值
}
```

### 3.2 动态触发规则 (BuffRule)
Buff 并不总是生效，它会根据 `BuffRule` 进行实时判定：
- `Always`: 始终生效。
- `InRank`: 英雄处于特定排位（1, 2, 3, 4）时生效。
- `LightAbove / LightBelow`: 依赖 `TorchMeter` 的光照水平。
- `HpAbove / HpBelow`: 依赖当前生命值百分比。
- `AtDeathsDoor`: 仅在濒死状态生效。
- `EnemyType`: 仅在针对特定种族（人类、不死等）的攻击时生效。

---

## 4. 英雄系统 (Hero)

`Hero` 类扩展了 `Character`，增加了大量持久化和城镇相关的逻辑。

### 4.1 怪癖与疾病 (Quirks & Diseases)
英雄拥有动态的特质列表：
```csharp
public class Hero : Character
{
    public List<Quirk> Quirks { get; private set; }
    public List<Disease> Diseases { get; private set; }
    
    // 获取所有怪癖提供的 Buff 集合
    public IEnumerable<Buff> QuirkBuffs => Quirks.SelectMany(q => q.Buffs);
}
```

### 4.2 意志等级 (Resolve)
```csharp
public class Resolve
{
    public int Level { get; set; }      // 0-6 级
    public int Experience { get; set; } // 当前经验
    
    // 升级所需的经验阈值表
    public static readonly int[] LevelThresholds = { 0, 2, 8, 14, 24, 38, 52 };
}
```

---

## 5. 怪物系统 (Monster)

怪物侧重于 AI 行为定义。

### 5.1 怪物大脑 (MonsterBrain)
每个怪物都有一个 `MonsterBrain` 配置文件（JSON），定义了其攻击偏好。
- **Skill Selection**: 根据目标英雄的状态（血量低、压力高、排位）分配技能权重。
- **Targeting**: 定义了该怪物倾向于攻击哪些位置的英雄。

### 5.2 怪物类型 (MonsterType)
怪物可以同时拥有多个标签：
- `Human`, `Beast`, `Eldritch`, `Unholy`, `Carcass`, `Staking`.
- 许多英雄技能（如十字军的 `Smite`）会对特定类型（`Unholy`）产生伤害加成。

---
[返回主页](Index.md)
