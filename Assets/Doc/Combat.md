# 战斗机制与数值 (Combat Mechanics)

本文档详细描述了本项目中战斗系统的核心逻辑实现，涵盖了从回合初始化、数值计算到特殊状态处理的完整生命周期。

---

## 1. 回合与先攻 (Round & Initiative)
回合管理器 (`Round.cs`) 驱动战斗流转。每回合开始时，系统会为所有活着的单位掷骰决定行动顺序。

### 1.1 先攻计算 (Initiative Roll)
<font color="#3498db">**核心公式：**</font> `Initiative = Speed + Random(0, 3) + RandomDouble()`

```csharp
// Assets/Scripts/Mechanics/Battle/Round.cs
// 每回合对所有单位进行先攻排序
OrderedUnits = new List<FormationUnit>(OrderedUnits.OrderByDescending(unit => 
    unit.Character.Speed + RandomSolver.Next(0, 3) + RandomSolver.NextDouble()));
```

- **受惊修正**：在 <font color="#e67e22">**HeroesSurprised**</font> 状态下，英雄方的速度会受到极大惩罚（-100）：
```csharp
// Round.cs: 受惊时的特殊排序逻辑
if (RaidSceneManager.BattleGround.SurpriseStatus == SurpriseStatus.HeroesSurprised)
    OrderedUnits = new List<FormationUnit>(OrderedUnits.OrderByDescending(unit => unit.Character.IsMonster ?
        unit.Character.Speed + RandomSolver.Next(0, 3) + RandomSolver.NextDouble() :
        unit.Character.Speed + RandomSolver.Next(0, 3) + RandomSolver.NextDouble() - 100)); // 英雄方 Initiative 大幅降低
```

---

## 2. 核心结算算法 (BattleSolver)

### 2.1 命中与闪避 (Hit & Dodge)
<font color="#3498db">**判定逻辑：**</font> 系统通过一个随机浮点数 `roll` 与计算出的 `hitChance` 进行对比。

```csharp
// Assets/Scripts/Mechanics/Battle/BattleSolver.cs
// 命中率计算：精度 - 闪避，且上限锁死在 95%
float accuracy = skill.Accuracy + performer.Accuracy;
float hitChance = Mathf.Clamp(accuracy - target.Dodge, 0, 0.95f); 
float roll = (float)RandomSolver.NextDouble();

if (roll > hitChance)
{
    // 区分是自己“打偏” (Miss) 还是对方“躲闪” (Dodge)
    if (roll > Mathf.Min(accuracy, 0.95f))
        SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Miss));
    else
        SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Dodge));
}
```

### 2.2 伤害计算 (Damage & PROT)
<font color="#3498db">**公式分析：**</font> 英雄伤害受武器区间和技能修正影响，怪物伤害则基于技能基础值和怪物倍率。

```csharp
// BattleSolver.cs: 伤害计算与防御减免
float initialDamage = performer is Hero ?
    Mathf.Lerp(performer.MinDamage, performer.MaxDamage, (float)RandomSolver.NextDouble()) * (1 + skill.DamageMod) :
    Mathf.Lerp(skill.DamageMin, skill.DamageMax, (float)RandomSolver.NextDouble()) * performer.DamageMod;

// 应用防御 (Protection) 减免：damage = initial * (1 - PROT)
int damage = Mathf.CeilToInt(initialDamage * (1 - target.Protection));
```

---

## 3. 拖延惩罚机制 (Stalling)
为了防止玩家利用弱小怪物无限刷血（Stalling），系统会在 `RaidSceneManager` 中检测战斗进度。

<font color="#e74c3c">**惩罚逻辑：**</font>
- **第 3 回合**：触发全队压力伤害 `stall_stress`。
- **第 4 回合**：强制召唤增援并重置。

```csharp
// Assets/Scripts/Managers/RaidSceneManager.cs: 协程驱动的拖延处理
if (BattleGround.MonsterFormation.IsStallingActive)
{
    BattleGround.StallingRoundNumber++;
    if (BattleGround.StallingRoundNumber == 3)
    {
        var stallEffect = DarkestDungeonManager.Data.Effects["stall_stress"];
        for (int i = 0; i < BattleGround.HeroParty.Units.Count; i++)
            stallEffect.ApplyIndependent(BattleGround.HeroParty.Units[i]); // 施加压力
    }
    else if (BattleGround.StallingRoundNumber == 4)
    {
        // 从当前区域的副本池中强制刷怪
        var stallSet = RandomSolver.ChooseByRandom(Raid.Dungeon.DungeonMash.StallEncounters);
        // ... 调用 BattleGround.SummonUnit 进行召唤
    }
}
```

---

## 4. 特殊状态 (Special States)

### 4.1 守护 (Guard)
守护机制通过 `GuardedStatusEffect` 和 `GuardStatusEffect` 互相引用实现：
```csharp
// Assets/Scripts/Raid/Battle/BattleGround.cs
// 加载保存的守护关联
var guardUnit = FindById(heroGuardedStatus.GuardCombatId);
if (guardUnit != null)
{
    heroGuardedStatus.Guard = guardUnit;
    var guardStatus = guardUnit.Character[StatusType.Guard] as GuardStatusEffect;
    guardStatus.Targets.Add(HeroParty.Units[i]); // 守护者持有受援者列表
}
```

### 4.2 控制 (Control)
精神控制通过动态修改单位的 `Team` 和父节点来实现：
```csharp
// BattleGround.cs: ControlUnit 核心逻辑
prisoner.Team = controller.Team; // 阵营反转
prisoner.transform.SetParent(controller.Party.transform, true); // 挂载到对方 Party 容器
prisoner.Formation = controller.Formation; // 关联到对方阵型
```

---
[返回主页](Index.md)
