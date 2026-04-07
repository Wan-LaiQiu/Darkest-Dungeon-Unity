# 技能系统深度技术文档 (Skill System Architecture)

本系统采用**数据驱动 (Data-Driven)** 设计，核心逻辑在于将技能的定义参数与其实际的战斗结算高度解耦。通过一套健壮的指令词解析引擎（Token Parser）和战斗结算器（`BattleSolver`），所有机制均实现了可配置化和灵活扩展。

---

## 目录
1. [核心数据结构](#1-核心数据结构)
2. [技能解析流程](#2-技能解析流程)
3. [技能执行流程](#3-技能执行流程)
4. [效果叠加机制](#4-效果叠加机制)
5. [伤害计算流程](#5-伤害计算流程)
6. [实战设计示例](#6-实战设计示例)

---

## 1. 核心数据结构

### 1.1 技能抽象基类 `Skill`

```csharp
public abstract class Skill
{
    public string Id { get; set; }
}
```

所有技能均继承自 `Skill`，仅包含技能唯一标识。

### 1.2 战斗技能 `CombatSkill`

**核心属性分类：**

| 类别 | 属性 | 说明 |
|------|------|------|
| **伤害属性** | `Accuracy`, `DamageMin`, `DamageMax`, `DamageMod`, `CritMod` | 精度、伤害区间、伤害修正、暴击修正 |
| **目标属性** | `LaunchRanks`, `TargetRanks` | 施法者站位限制、技能打击位置 |
| **能力标志** | `IsCritValid`, `IsSelfValid`, `CanMiss` | 是否可暴击、是否可对自己生效、是否可能未命中 |
| **效果组件** | `Effects`, `ModeEffects` | 技能附带的特效列表 |
| **辅助组件** | `Heal`, `Move` | 治疗组件、位移组件 |
| **使用限制** | `LimitPerTurn`, `LimitPerBattle`, `IsContinueTurn` | 每回合/每战斗次数限制、是否继续行动 |

**关键方法：**
```csharp
// 获取可用的目标列表
public List<FormationUnit> GetAvailableTargets(FormationUnit performer, 
    FormationParty friends, FormationParty enemies)

// 检测是否存在可用目标
public bool HasAvailableTargets(FormationUnit performer, 
    FormationParty friends, FormationParty enemies)

// 动态判断是否为增益技能
public bool IsBuffSkill
{
    get
    {
        return Effects.Find(effect => effect.SubEffects.Find(subEffect =>
            subEffect.Type == EffectSubType.Buff || 
            subEffect.Type == EffectSubType.StatBuff) != null) != null;
    }
}
```

### 1.3 效果系统 `Effect`

```csharp
public class Effect
{
    public string Name { get; private set; }
    public EffectTargetType TargetType { get; private set; }  // 作用目标类型
    public Dictionary<EffectBoolParams, bool?> BooleanParams { get; private set; }  // 布尔参数
    public Dictionary<EffectIntParams, int?> IntegerParams { get; private set; }    // 整数参数
    public List<SubEffect> SubEffects { get; private set; }  // 子效应列表
}
```

**目标类型 (`EffectTargetType`)：**
- `Target` - 技能主目标
- `Performer` - 施法者自身
- `PerformersOther` - 施法者所在队伍的其他成员
- `TargetGroup` - 目标所在队伍的全部成员
- `Global` - 全局效果（如火把值变化）

### 1.4 子效应抽象 `SubEffect`

```csharp
public abstract class SubEffect
{
    public abstract EffectSubType Type { get; }
    public virtual bool Fusable { get { return false; } }  // 是否可融合

    public virtual void Apply(FormationUnit performer, FormationUnit target, Effect effect);
    public abstract bool ApplyQueued(FormationUnit performer, FormationUnit target, Effect effect);
    public abstract bool ApplyInstant(FormationUnit performer, FormationUnit target, Effect effect);
}
```

**常见子效应实现：**
| 子效应类 | `EffectSubType` | 功能 |
|----------|-----------------|------|
| `BleedEffect` | Bleeding | 流血 DoT |
| `PoisonEffect` | Poison | 中毒 DoT |
| `StressEffect` | Stress | 压力伤害 |
| `StunEffect` | Stun | 眩晕 |
| `HealEffect` | Heal | 治疗 |
| `PushEffect` | - | 击退 |
| `PullEffect` | - | 拉近 |
| `GuardEffect` | Guard | 护盾 |

### 1.5 辅助组件

```csharp
// 治疗组件
public class HealComponent
{
    public int MinAmount { get; set; }
    public int MaxAmount { get; set; }
}

// 位移组件
public class MoveComponent
{
    public int Pushback { get; set; }   // 推后格数
    public int Pullforward { get; set; } // 拉前格数
}
```

### 1.6 站位定义 `FormationSet`

```csharp
public class FormationSet
{
    public bool IsMultitarget { get; private set; }   // 是否AOE
    public bool IsRandomTarget { get; private set; } // 是否随机目标
    public bool IsSelfFormation { get; private set; } // 是否友方目标
    public bool IsSelfTarget { get; private set; }    // 是否对自己施放
    public List<int> Ranks { get; private set; }      // 站位编号

    public bool IsLaunchableFrom(int rank, int size);  // 检测施法者站位
    public bool IsTargetableUnit(FormationUnit unit);   // 检测目标是否在范围内
}
```

**站位字符串解析规则：**
- `12` - 打击位置1和位置2
- `~12` - `~`表示AOE，打击位置1和2的所有单位
- `@12` - `@`表示友方目标
- 无前缀 - 敌方目标

---

## 2. 技能解析流程

### 2.1 数据文件格式

技能定义以 `.bytes` 或 `.txt` 文件存储，示例（Leper 的 chop 技能）：

```
combat_skill: .id "chop" .level 0 .type "melee" .atk 75% .dmg 0% .crit 2% 
              .launch 12 .target 12 .is_crit_valid True .generation_guaranteed true
```

**解析流程图：**

```
数据文件加载
    ↓
HeroClass.LoadData() 解析英雄数据块
    ↓
遇到 "combat_skill:" 标记
    ↓
提取技能数据列表 (combatData)
    ↓
new CombatSkill(combatData, true) 构造函数
    ↓
CombatSkill.LoadData() Token解析
    ↓
根据 Token 类型填充属性
    ↓
从 DarkestDungeonManager.Data.Effects 加载 Effect 引用
```

### 2.2 Token 解析详解

```csharp
private void LoadData(List<string> data, bool isHeroSkill)
{
    for (int i = 1; i < data.Count; i++)
    {
        switch (data[i])
        {
            case ".id":
                Id = data[++i];
                break;
            case ".atk":
                Accuracy = float.Parse(data[++i]) / 100;
                break;
            case ".dmg":
                if (isHeroSkill)
                    DamageMod = float.Parse(data[++i]) / 100;
                else
                {
                    DamageMin = float.Parse(data[++i]);
                    DamageMax = float.Parse(data[++i]);
                }
                break;
            case ".launch":
                LaunchRanks = new FormationSet(data[++i]);
                break;
            case ".target":
                TargetRanks = new FormationSet(data[++i]);
                break;
            case ".effect":
                while (++i < data.Count && data[i--][0] != '.')
                {
                    if (DarkestDungeonManager.Data.Effects.ContainsKey(data[++i]))
                        Effects.Add(DarkestDungeonManager.Data.Effects[data[i]]);
                }
                break;
            // ... 其他 Token
        }
    }
}
```

### 2.3 Effect 解析流程

效果在 `Effects.txt` 中定义：

```
effect: .name "Push 3A" .target "target" .push 3 .chance 100% 
        .on_hit true .on_miss false .can_apply_on_death true
```

```csharp
private void LoadData(List<string> data)
{
    for (int i = 1; i < data.Count; i++)
    {
        switch (data[i])
        {
            case ".push":
                SubEffects.Add(new PushEffect(int.Parse(data[++i])));
                break;
            case ".stun":
                SubEffects.Add(new StunEffect());
                break;
            case ".dotBleed":
                SubEffects.Add(new BleedEffect(int.Parse(data[++i])));
                break;
            case ".stress":
                SubEffects.Add(new StressEffect(int.Parse(data[++i])));
                break;
            case ".buff_ids":
                BuffEffect buffEffect = new BuffEffect();
                // ... 加载 Buff 列表
                SubEffects.Add(buffEffect);
                break;
            // ... 其他子效应
        }
    }
}
```

---

## 3. 技能执行流程

### 3.1 入口：`BattleSolver.ExecuteSkill`

```csharp
public static void ExecuteSkill(FormationUnit performerUnit, FormationUnit targetUnit, 
    CombatSkill skill, SkillArtInfo artInfo)
{
    // 1. 初始化结果容器
    SkillResult.Skill = skill;
    SkillResult.ArtInfo = artInfo;

    var target = targetUnit.Character;
    var performer = performerUnit.Character;

    // 2. 应用临时条件（Buff规则）
    ApplyConditions(performerUnit, targetUnit, skill);

    // 3. 处理强制位移
    if (skill.Move != null && !performerUnit.CombatInfo.IsImmobilized)
    {
        if (skill.Move.Pullforward > 0)
            performerUnit.Pull(skill.Move.Pullforward, false);
        else if (skill.Move.Pushback > 0)
            performerUnit.Push(skill.Move.Pushback, false);
    }

    // 4. 分支：治疗/辅助 vs 伤害
    if (skill.Category == SkillCategory.Heal || skill.Category == SkillCategory.Support)
    {
        ExecuteHeal(performerUnit, targetUnit, skill);
    }
    else
    {
        ExecuteDamage(performerUnit, targetUnit, skill);
    }
}
```

### 3.2 执行链路图

```
玩家选择技能 → 验证可用性 (IsSkillUsable)
    ↓
玩家选择目标 → 获取有效目标 (GetSkillAvailableTargets)
    ↓
点击确认 → SelectSkillTargets 生成 SkillTargetInfo
    ↓
BattleSolver.ExecuteSkill(performer, target, skill, artInfo)
    ├─ ApplyConditions()      应用 Buff 规则
    ├─ 处理 Move 组件         强制位移
    ├─ 分类处理
    │   ├─ ExecuteHeal()     治疗/辅助分支
    │   │   ├─ 精度判定（可选）
    │   │   ├─ 治疗量计算
    │   │   └─ ApplyEffects()
    │   └─ ExecuteDamage()   伤害分支
    │       ├─ 精度判定
    │       ├─ 伤害投点
    │       ├─ 护甲计算
    │       ├─ 暴击判定
    │       └─ ApplyEffects()
    ↓
结算结果存入 SkillResult
```

---

## 4. 效果叠加机制

### 4.1 效果分发入口

```csharp
public static void ApplyEffects(FormationUnit performerUnit, FormationUnit targetUnit, CombatSkill skill)
{
    // 1. 处理形态专属效果（如人/兽形态）
    if(skill.ValidModes.Count > 1 && performerUnit.Character.Mode != null)
        foreach (var effect in skill.ModeEffects[performerUnit.Character.Mode.Id])
            effect.Apply(performerUnit, targetUnit, SkillResult);

    // 2. 处理技能基础效果
    foreach (var effect in skill.Effects)
        effect.Apply(performerUnit, targetUnit, SkillResult);
}
```

### 4.2 Effect.Apply() 执行流程

```csharp
public void Apply(FormationUnit performer, FormationUnit target, SkillResult skillResult)
{
    // 1. 检测是否已应用过（apply_once）
    if (BooleanParams[EffectBoolParams.ApplyOnce].Value == true)
        if (skillResult.AppliedEffects.Contains(this))
            return;

    // 2. 检测未命中/闪避跳过
    if (BooleanParams[EffectBoolParams.OnMiss] == false)
        if (skillResult.Current.Type == SkillResultType.Miss || 
            skillResult.Current.Type == SkillResultType.Dodge)
            return;

    // 3. 检测死亡后是否可应用
    if (BooleanParams[EffectBoolParams.CanApplyAfterDeath] == false)
        if (skillResult.Current.IsZeroed)
            return;

    // 4. 根据目标类型分发
    switch(TargetType)
    {
        case EffectTargetType.Target:
            foreach (var subEffect in SubEffects)
                subEffect.Apply(performer, target, this);
            break;
        case EffectTargetType.Performer:
            foreach (var subEffect in SubEffects)
                subEffect.Apply(performer, performer, this);
            break;
        case EffectTargetType.PerformersOther:
            foreach(var unit in performer.Party.Units)
                if(unit != performer)
                    foreach (var subEffect in SubEffects)
                        subEffect.Apply(performer, unit, this);
            break;
        // ...
    }

    skillResult.AppliedEffects.Add(this);
}
```

### 4.3 SubEffect 队列机制

```csharp
public virtual void Apply(FormationUnit performer, FormationUnit target, Effect effect)
{
    if (effect.BooleanParams[EffectBoolParams.Queue] == false)
        ApplyInstant(performer, target, effect);  // 立即应用
    else
        target.EventQueue.Add(new EffectEvent(performer, target, effect, this));  // 排队
}
```

**排队机制的设计意图：** 防止特效迸发导致的动画或数据表现死锁，确保按序播放表现层动画并结算实质伤害。

### 4.4 可融合效果 (Fusable Effects)

部分效果支持融合叠加，例如 `StressEffect`：

```csharp
public override bool Fusable { get { return true; } }

// 融合多个同类型压力伤害
public override int Fuse(FormationUnit performer, FormationUnit target, Effect effect)
{
    // 计算融合后的总压力值
    float initialDamage = StressAmount;
    initialDamage *= (1 + performer.Character.GetSingleAttribute(AttributeType.StressDmgPercent).ModifiedValue);
    int damage = Mathf.RoundToInt(initialDamage * (1 + 
        target.Character.GetSingleAttribute(AttributeType.StressDmgReceivedPercent).ModifiedValue));
    if (damage < 1) damage = 1;
    return damage;
}
```

---

## 5. 伤害计算流程

### 5.1 完整伤害结算流程

```csharp
private static void ExecuteDamage(FormationUnit performerUnit, FormationUnit targetUnit, CombatSkill skill)
{
    var target = targetUnit.Character;
    var performer = performerUnit.Character;

    // ===== 第一步：精度判定 =====
    float accuracy = skill.Accuracy + performer.Accuracy;
    float hitChance = Mathf.Clamp(accuracy - target.Dodge, 0, 0.95f);
    float roll = (float)RandomSolver.NextDouble();

    // 检测不可被命中
    if (target.BattleModifiers != null && target.BattleModifiers.CanBeHit == false)
        roll = float.MaxValue;

    if (roll > hitChance)
    {
        if (!(skill.CanMiss == false || 
              (target.BattleModifiers != null && target.BattleModifiers.CanBeMissed == false)))
        {
            if (roll > Mathf.Min(accuracy, 0.95f))
                SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Miss));
            else
                SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Dodge));

            ApplyEffects(performerUnit, targetUnit, skill);  // Miss时仍可触发效果
            return;
        }
    }

    // ===== 第二步：伤害投点 =====
    float initialDamage;
    if (performer is Hero)
    {
        // 英雄：从角色属性计算伤害区间
        initialDamage = Mathf.Lerp(performer.MinDamage, performer.MaxDamage, 
            (float)RandomSolver.NextDouble()) * (1 + skill.DamageMod);
    }
    else
    {
        // 怪物：从技能定义计算
        initialDamage = Mathf.Lerp(skill.DamageMin, skill.DamageMax, 
            (float)RandomSolver.NextDouble()) * performer.DamageMod;
    }

    // ===== 第三步：护甲计算 =====
    int damage = Mathf.CeilToInt(initialDamage * (1 - target.Protection));
    if (damage < 0) damage = 0;

    // 检测是否可直接伤害
    if (target.BattleModifiers != null && target.BattleModifiers.CanBeDamagedDirectly == false)
        damage = 0;

    // ===== 第四步：暴击判定 =====
    if (skill.IsCritValid)
    {
        float critChance = performer.GetSingleAttribute(AttributeType.CritChance).ModifiedValue + skill.CritMod;
        if (RandomSolver.CheckSuccess(critChance))
        {
            int critDamage = target.TakeDamage(damage * 1.5f);  // 暴击伤害1.5倍
            SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, critDamage, SkillResultType.Crit));

            ApplyEffects(performerUnit, targetUnit, skill);

            // 英雄暴击触发全队减压
            if (targetUnit.Character.IsMonster == false)
                DarkestDungeonManager.Data.Effects["Stress 2"].ApplyIndependent(targetUnit);
            return;
        }
    }

    // ===== 第五步：正常伤害结算 =====
    damage = target.TakeDamage(damage);
    SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, damage, SkillResultType.Hit));
    ApplyEffects(performerUnit, targetUnit, skill);
}
```

### 5.2 伤害计算公式汇总

| 计算步骤 | 公式 | 说明 |
|----------|------|------|
| **命中判定** | `hitChance = Clamp(SkillAccuracy + PerformerAccuracy - TargetDodge, 0, 0.95)` | 上限95% |
| **英雄伤害** | `Lerp(MinDmg, MaxDmg, random) * (1 + DamageMod)` | 区间内插值 |
| **怪物伤害** | `Lerp(DamageMin, DamageMax, random) * DamageMod` | 使用技能固定区间 |
| **护甲减免** | `damage = CeilToInt(initialDamage * (1 - Protection))` | 向上取整 |
| **暴击伤害** | `damage * 1.5` | 固定1.5倍系数 |

### 5.3 治疗计算流程

```csharp
if (skill.Heal != null)
{
    // 基础治疗量
    float initialHeal = RandomSolver.Next(skill.Heal.MinAmount, skill.Heal.MaxAmount + 1) *
        (1 + performer.GetSingleAttribute(AttributeType.HpHealPercent).ModifiedValue);

    // 暴击治疗（1.5倍）
    if (skill.IsCritValid)
    {
        float critChance = performer[AttributeType.CritChance].ModifiedValue + skill.CritMod / 100;
        if (RandomSolver.CheckSuccess(critChance))
        {
            int critHeal = target.Heal(initialHeal * 1.5f, true);
            SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, critHeal, SkillResultType.CritHeal));
            // 暴击治疗触发全队减压
            DarkestDungeonManager.Data.Effects["crit_heal_stress_heal"].ApplyIndependent(targetUnit);
            return;
        }
    }

    int heal = target.Heal(initialHeal, true);
    SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, heal, SkillResultType.Heal));
}
```

---

## 6. 实战设计示例

### 6.1 设计目标：创建"重斩"技能

**设计需求：**
- 对单个敌人造成高伤害
- 附加25%护甲穿透
- 30%几率造成眩晕
- 目标流血3回合，每回合5点伤害

### 6.2 第一步：定义效果（Effects.txt）

```txt
effect: .name "Heavy Strike Armor Break" .target "target" 
        .protection_rating_add 25
        .on_hit true .queue true

effect: .name "Heavy Strike Stun" .target "target" 
        .stun 1 .chance 30%
        .on_hit true .queue true

effect: .name "Heavy Strike Bleed" .target "target" 
        .dotBleed 5 .duration 3 .chance 100%
        .on_hit true .queue true
```

### 6.3 第二步：定义技能（Hero.bytes）

```txt
combat_skill: .id "heavy_strike" .level 0 .type "melee" 
              .atk 85% .dmg 50% .crit 5% 
              .launch 12 .target 1 
              .is_crit_valid True
              .effect "Heavy Strike Armor Break" 
              "Heavy Strike Stun" 
              "Heavy Strike Bleed"
```

### 6.4 代码解析

**技能解析时：**
```csharp
case ".effect":
    while (++i < data.Count && data[i--][0] != '.')
    {
        if (DarkestDungeonManager.Data.Effects.ContainsKey(data[i]))
            Effects.Add(DarkestDungeonManager.Data.Effects[data[i]]);
    }
    break;
```

**执行时效果叠加：**
```
BattleSolver.ExecuteSkill()
    ├─ 精度判定（85% + 施法者精度 - 目标闪避）
    ├─ 伤害计算（Lerp(min, max) * 1.5 * 护甲减免）
    ├─ 暴击判定（5% + 暴击率）
    └─ ApplyEffects() 依次应用：
        ├─ Heavy Strike Armor Break → 护甲降低
        ├─ Heavy Strike Stun → 30%眩晕判定
        └─ Heavy Strike Bleed → 3回合流血，每回5伤
```

### 6.5 变身技能设计（Abomination 示例）

```txt
combat_skill: .id "transform" .level 0 .type "melee" 
              .atk 0% .dmg 0% .crit 0% 
              .launch 12 .target
              .valid_modes human beast
              .human_effects "Abomination Human Heal" "Abomination Human Stress Heal"
              .beast_effects "Abomination Beast Self Damage" "Abomination Beast Party Damage"
```

**形态效果分发：**
```csharp
if(skill.ValidModes.Count > 1 && performerUnit.Character.Mode != null)
    foreach (var effect in skill.ModeEffects[performerUnit.Character.Mode.Id])
        effect.Apply(performerUnit, targetUnit, SkillResult);
```

---

## 附录：关键文件索引

| 文件路径 | 说明 |
|----------|------|
| `Assets/Scripts/Mechanics/Skills/Skill.cs` | 技能抽象基类 |
| `Assets/Scripts/Mechanics/Skills/CombatSkill.cs` | 战斗技能实现 |
| `Assets/Scripts/Mechanics/Skills/Effect.cs` | 效果容器 |
| `Assets/Scripts/Mechanics/Skills/SubEffect.cs` | 子效应抽象 |
| `Assets/Scripts/Mechanics/Battle/BattleSolver.cs` | 战斗结算器 |
| `Assets/Scripts/Mechanics/Battle/FormationSet.cs` | 站位定义 |
| `Assets/Resources/Data/Mechanics/Effects.txt` | 效果数据定义 |
| `Assets/Resources/Data/Heroes/Info/*.bytes` | 英雄技能定义 |

---

[返回主页](Index.md)
