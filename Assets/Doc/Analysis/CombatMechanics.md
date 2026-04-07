# Darkest Dungeon Unity 战斗机制技术文档

本文档详细描述了本项目中战斗系统的核心逻辑实现，涵盖了从回合初始化到伤害结算的完整生命周期。

## 1. 回合初始化与行动顺序 (Initiative & Round Setup)

战斗的节奏由 `Round.cs` 类管理，并通过 `RaidSceneManager.BattleRound` 协程驱动。

- **先攻权计算**：每回合开始时，系统会为所有存活单位计算先攻值。
  - **公式**：`最终先攻 = 基础速度 (Speed) + Random(0, 3) + RandomDouble()`。
- **行动排序**：
  - 单位按先攻值**降序**排列在 `OrderedUnits` 列表中。
  - **惊吓机制 (`SurpriseStatus`)**：
    - 若英雄被惊吓，其先攻计算会受到 `-100` 的惩罚，确保怪物先行。
    - 若怪物被惊吓，其先攻计算会受到 `-100` 的惩罚（需满足可被惊吓条件）。
- **行动提取**：`BattleRound` 循环中每次提取列表首位 `OrderedUnits[0]`，并将其从列表中移除，进入该单位的独立回合。

## 2. 英雄技能选择流程 (Skill Selection)

当轮到英雄行动时，交互逻辑按以下顺序展开：

1. **界面激活**：`RaidCombatSkillsPanel` 显示当前英雄装备的 4 个战斗技能。
2. **点击技能**：玩家点击 `BattleSkillSlot`。
3. **可用性检查 (`BattleSolver.IsSkillUsable`)**：
   - **站位检查**：技能必须在当前 `Rank` 下可释放（`LaunchRanks`）。
   - **目标检查**：战场上必须存在符合该技能 `TargetRanks` 的合法目标。
4. **目标高亮**：选定技能后，`RaidSceneManager` 调用 `GetSkillAvailableTargets`，并通过 `FormationOverlay` 高亮所有可被点击的目标。

## 3. 目标选择机制 (Target Selection)

玩家通过点击场景中单位上方的透明遮罩（`FormationOverlaySlot`）来确定攻击目标。

- **点击判定**：`FormationOverlaySlot.OnPointerClick` 检查目标是否 `IsTargetable`。
- **多目标处理**：
  - 若技能标记为 `IsMultitarget`（如群体攻击），点击其中一个合法目标将自动选中该技能定义的所有范围目标。
  - 最终目标列表由 `BattleSolver.SelectSkillTargets` 确定，并传递给结算引擎。

## 4. 伤害结算与效果 (Damage & Execution)

核心结算逻辑位于 `BattleSolver.ExecuteSkill`，包含以下步骤：

### A. 命中判定 (Accuracy Check)
- **公式**：`命中率 = 技能基础精度 + 施法者精度修饰 - 目标闪避 (Dodge)`。
- **限制**：最终命中率上限通常为 **95%**（除非技能具有必中属性 `CanMiss == false`）。
- **随机判定**：随机数大于命中率则触发 `Miss` 或 `Dodge`。

### B. 伤害计算 (Damage Calculation)
- **英雄伤害**：`武器基础伤害区间随机值 * (1 + 技能伤害修正 %)`。
- **怪物伤害**：`技能伤害区间随机值 * 怪物属性伤害修正 %`。
- **防御减免**：`实际伤害 = 向上取整(初始伤害 * (1 - 目标防御 Protection))`。
- **暴击判定**：
  - 若触发暴击（`CritChance` 判定成功），伤害提升至 **1.5 倍**。

### C. 效果应用 (Effect Application)
- **治疗逻辑**：基础治疗量同样受治疗加成影响，且可以暴击（1.5倍治疗并减压）。
- **状态附加**：计算流血、腐蚀、眩晕等的抗性判定。
- **位移处理**：处理技能自带的向前/向后移动效果。

## 5. 循环与下一个单位 (Next Unit Action)

- **单体回合结束**：技能执行完毕或玩家选择跳过/移动后，调用 `PostHeroTurn` 或 `PostMonsterTurn`。
- **主循环驱动**：`BattleRound` 协程检测 `OrderedUnits`。
  - 若列表不为空：继续提取下一个 `unit` 并开启其回合。
  - 若列表为空：当前大回合结束，触发 `ExecuteRoundAdvance` 增加回合计数，并重新开始第一阶段。
- **战斗结束判定**：在每次单位行动前后，系统都会通过 `BattleGround.IsBattleEnded()` 检查是否所有敌方已死亡或我方撤退。
