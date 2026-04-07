# 技能系统深度技术文档 (Skill System Architecture)

本系统采用**数据驱动 (Data-Driven)** 设计，核心逻辑在于将技能的定义参数与其实际的战斗结算高度解耦。通过一套健壮的指令词解析引擎（Token Parser）和战斗结算器（`BattleSolver`），所有机制均实现了可配置化和灵活扩展。

---

## 1. 技能类型架构

所有的技能全部统一继承自抽象基类 `<span style="color:#00BFFF;">Skill</span>`。

```csharp
public abstract class Skill
{
    public string Id { get; set; }
}
```

### 1.1 战斗技能 (`CombatSkill`)

战斗技能是核心副本战斗循环中的主力组件。系统不对每个技能进行硬编码，而是通过读取按词法令牌（Tokens）解析的配置流，将数据灌入 `CombatSkill` 实例，从而实现高度动态的技能配置。

#### 1. 技能的数据定义与配置设计
在底层数据表中，技能是一串以 `.` 开头的指令与参数。通过 `LoadData()` 解析流，能够定义以下维度：
- **基本属性**：如 `.atk`（精度 `Accuracy`）、`.dmg`（伤害修正 `DamageMod`）、`.crit`（暴击修正 `CritMod`）。
- **目标与可用性判定**：
  - <span style="color:#32CD32;">LaunchRanks</span>：施法者站位限制（如 `.launch 1234` 表示任意位置可放）。
  - <span style="color:#32CD32;">TargetRanks</span>：技能打击位置（如 `.target ~123` 表示打击敌方前三排，`~`代表群体 AOE；`@12` 代表对自己或者友军生效）。
  - <span style="color:#32CD32;">can_miss</span> 与 <span style="color:#32CD32;">is_crit_valid</span>：控制技能是否为必中技能（如状态辅助技能通常设为 `false`）以及是否允许暴击。
- **高级控制参数**：
  - `.per_turn_limit` / `.per_battle_limit`：限制技能在单回合或单场战斗中的最大使用次数。
  - `.is_continue_turn`：配置技能释放后是否允许该角色继续行动（不消耗当前回合）。
  - `.extra_targets_chance`：技能附带额外打击目标的几率配置。
- **治疗与位移组件**：
  - `.heal`：绑定 <span style="color:#FF69B4;">HealComponent</span>，定义治疗量的上下限（不产生直接伤害）。
  - `.move`：绑定 <span style="color:#FF69B4;">MoveComponent</span>，实现攻击附带的向前拉（Pull）或向后推（Push）强制位移效果。
- **模式切换 (Modes)**：
  - 针对变身类职业（如狗人 Abomination），使用 `.valid_modes` 指定可用的形态，同时通过 `.human_effects` 和 `.beast_effects`，为同个技能在不同形态下配置互不干扰的独立附加效果字典 <span style="color:#9370DB;">ModeEffects</span>。

**源码解析：动态类别判定 (IsBuffSkill)**
系统避免了对增益技能的硬编码标记，而是通过检查技能挂载的特效类型动态返回是否为 Buff：
```csharp
public bool IsBuffSkill
{
    get
    {
        return Effects.Find(effect => effect.SubEffects.Find(subEffect =>
            subEffect.Type == EffectSubType.Buff || subEffect.Type == EffectSubType.StatBuff) != null) != null;
    }
}
```

#### 2. 技能的合法性判定与目标获取
当玩家在 UI 上点击某个技能图标时，系统首先需要判定该技能此时**能否被使用**（按钮是否亮起），并**获取合法的目标**：
1. **施法者站位判定**：`LaunchRanks.IsLaunchableFrom(performer.Rank)` 检测施法者当前在队伍中的相对位置是否满足 `.launch` 限制。
2. **目标存在性与合法性判定**：
   - 调用 `HasAvailableTargets` 检测场上（敌方或友方队伍）是否存在活着的实体。
   - `GetAvailableTargets` 负责精细化提取列表：
     - 若为 **自我施法 (`IsSelfTarget`)**，仅返回自身。
     - 若为 **友军目标 (`IsSelfFormation`)**，过滤存活的友军，并额外检查自身是否允许作为目标 (`IsSelfValid`)，以及目标身上是否带有特殊的排斥修饰符 (`IsValidFriendlyTarget`，例如某些特殊状态下不可被作为友方技能目标)。
     - 若为 **敌军目标**，筛选对方存活且满足 `TargetRanks` 站位定义的敌人。
3. **自身状态约束**：还要检测施法者身上是否挂载了阻断状态（例如通过 `performer.CombatInfo.BlockedHealUnitIds` 判断是否被禁疗），以及是否达到了该技能的 `LimitPerTurn` / `LimitPerBattle` 限制。

**源码解析：精细化目标获取提取**
```csharp
public List<FormationUnit> GetAvailableTargets(FormationUnit performer, FormationParty friends, FormationParty enemies)
{
    if (TargetRanks.IsSelfTarget) return new List<FormationUnit>(new[] { performer });

    if (TargetRanks.IsSelfFormation)
        return friends.Units.FindAll(unit =>
            ((unit == performer && TargetRanks.IsTargetableUnit(unit) && IsSelfValid) ||
            (unit != performer && TargetRanks.IsTargetableUnit(unit))) && 
            (unit.Character.BattleModifiers == null || unit.Character.BattleModifiers.IsValidFriendlyTarget));

    return enemies.Units.FindAll(unit => unit != performer && TargetRanks.IsTargetableUnit(unit));
}
```

#### 3. 技能的执行流转与目标收集
如果技能判定合法，玩家点击目标或按下确认后的执行过程如下：
1. **生成目标遮罩**：UI 会高亮被 `TargetRanks` 囊括的所有目标，并调用 `BattleSolver.GetSkillAvailableTargets` 生成合法的可点击区域（Overlay）。
2. **收集最终目标**：根据是单体（点击单个单位）还是多目标（点击任意合法单位即选中受 AOE 影响的所有目标），将受击者封装入 `SkillTargetInfo`，交由执行器。
3. **进入核心结算器**：进入 `<span style="color:#FF4500;">BattleSolver.ExecuteSkill</span>`，依次进行：应用临时加成 -> 强制位移 -> 精度判定 -> 伤害结算 -> 调用特效。

#### 4. 特效 (Effect) 的叠加与挂载设计
一个技能之所以能产生丰富的表现，全靠后续的**效果分发**。技能在 `CombatSkill` 内部维护了 `List<Effect> Effects` 列表：
- **挂载阶段**：伤害判定结束后，系统会统一调用 `BattleSolver.ApplyEffects()` 遍历特效引用。
- **效果触发判定**：每个 `Effect` 内部会根据配置决定是否执行（如 `.on_miss false` 意为如果判定 Miss，中止该效果）。
- **特效组合叠加机制**：一个技能可以组合任意多个效果，如“放血打击”可同时叠加：
  1. `BleedEffect`（挂载持续流血 Dot）。
  2. `CombatStatBuffEffect`（挂载减益减速）。
  3. `PushEffect`（强制击退 1 格）。

**源码解析：统一特效分发执行**
```csharp
public static void ApplyEffects(FormationUnit performerUnit, FormationUnit targetUnit, CombatSkill skill)
{
    // 应用当前形态 (如人/野兽) 的专属特效
    if(skill.ValidModes.Count > 1 && performerUnit.Character.Mode != null)
        foreach (var effect in skill.ModeEffects[performerUnit.Character.Mode.Id])
            effect.Apply(performerUnit, targetUnit, SkillResult);

    // 应用技能自身的基础特效
    foreach (var effect in skill.Effects)
        effect.Apply(performerUnit, targetUnit, SkillResult);
}
```

### 1.2 扎营技能 (`CampingSkill`)
跳出了传统空间网格，而是围绕 **时间** 和 **目标条件** 设计的副本外辅助系统。
- **目标选择**：使用枚举 `CampTargetType`，定义其作用域（如 `Individual` 单体队友、`Self` 自己、`PartyOther` 除自己以外全队）。
- **前置需求**：带有严格的 `CampEffectRequirement`。比如必须携带某种信仰（`Religious`），或者处于折磨状态（`Afflicted`）才可触发特定的恢复效果。
- **时间成本**：受到 `RaidSceneManager.Raid.CampingTimeLeft` 限制，通过 `TimeCost` 点数削减扎营剩余时间。

### 1.3 位移技能 (`MoveSkill`)
专门响应战斗过程中的手动换位指令，仅提供 `MoveForward`（前移格数）和 `MoveBackward`（后退格数）的简易封装。

---

## 2. 战斗引擎结算主脑：`BattleSolver`

战斗技能真正发挥作用的地方在 `<span style="color:#FF4500;">BattleSolver.ExecuteSkill</span>`。

### 2.1 第一阶段：应用临时条件 (ApplyConditions)
调用 `performerUnit.Character.ApplyAllBuffRules`，加载单次攻击的被动“条件”加成。

### 2.2 第二阶段：位移前置 (Move)
处理自带的 `<span style="color:#32CD32;">Move</span>` 组件强制位移。

### 2.3 第三阶段：分类结算逻辑

**【A. 治疗/辅助技能】**
1. **基础治疗计算**：附加 `HpHealPercent.ModifiedValue`（施法者治疗加成）。
2. **暴击判定**：
   - 暴击治疗量变为 **1.5 倍**。并触发全队减压：
     `DarkestDungeonManager.Data.Effects["crit_heal_stress_heal"].ApplyIndependent(targetUnit)`。

**【B. 伤害技能】**
1. **精度判定 (Accuracy & Dodge)**
   - 命中公式：`<span style="color:#ADD8E6;">hitChance = Mathf.Clamp(SkillAccuracy + PerformerAccuracy - TargetDodge, 0, 0.95f)</span>`
2. **伤害区间投点 (Damage Roll)**
   - 英雄伤害公式：`<span style="color:#ADD8E6;">Lerp(MinDmg, MaxDmg) * (1 + DamageMod)</span>`
3. **减伤计算 (Protection)**
   - 护甲免伤公式：`<span style="color:#ADD8E6;">damage = Mathf.CeilToInt(initialDamage * (1 - target.Protection))</span>`
4. **暴击判定翻倍**：不仅伤害 1.5 倍，英雄暴击还会触发全局 `Stress 2` 特效进行减压/增压处理。

**源码解析：伤害结算与暴击判定片段**
```csharp
// 命中判定（最高95%）
float hitChance = Mathf.Clamp(accuracy - target.Dodge, 0, 0.95f);
float roll = (float)RandomSolver.NextDouble();

if (roll > hitChance && skill.CanMiss != false)
{
    // 未命中（根据 roll 区分自己打偏还是对方闪避）
    if (roll > Mathf.Min(accuracy, 0.95f))
        SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Miss));
    else
        SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, SkillResultType.Dodge));
    ApplyEffects(performerUnit, targetUnit, skill);
    return;
}

// ...省略伤害区间的 Lerp 计算...

// 暴击判定
if (skill.IsCritValid && RandomSolver.CheckSuccess(critChance))
{
    int critDamage = target.TakeDamage(damage * 1.5f); // 伤害翻倍
    SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, critDamage, SkillResultType.Crit));
    ApplyEffects(performerUnit, targetUnit, skill);
    return;
}

// 正常伤害结算
int finalDamage = target.TakeDamage(damage);
SkillResult.AddResultEntry(new SkillResultEntry(targetUnit, finalDamage, SkillResultType.Hit));
ApplyEffects(performerUnit, targetUnit, skill);
```

---

## 3. 技能效果器 (Effect) 引擎

挂 Buff、流血、击晕的核心组件。

### 3.1 `Effect` 容器
包含 `<span style="color:#9370DB;">TargetType</span>`（作用域如 `Performer`, `Global`）和执行过滤参数（如 `.on_miss false`）。

### 3.2 子效应 (`SubEffect`) 多态体系
真正执行操作的单元，常见实现包括 `<span style="color:#FF4500;">StressEffect</span>`、`<span style="color:#FF4500;">BleedEffect</span>`、`<span style="color:#FF4500;">StunEffect</span>`。

**【事件队列注入 (Event Queue)】**
在 `SubEffect.Apply` 中，除非特效配置了强行执行，否则都会以 `EffectEvent` 的形式被压入目标队列，防止特效迸发导致的动画或数据表现死锁。

**源码解析：事件安全排队机制**
```csharp
public virtual void Apply(FormationUnit performer, FormationUnit target, Effect effect)
{
    if (effect.BooleanParams[EffectBoolParams.Queue] == false)
        ApplyInstant(performer, target, effect);
    else
        // 将事件封装进队列以按序依次播放表现层动画并结算实质伤害
        target.EventQueue.Add(new EffectEvent(performer, target, effect, this));
}
```

---
[返回主页](Index.md)