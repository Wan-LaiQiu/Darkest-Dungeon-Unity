# Campaign 系统分析文档

本文档详细分析 Darkest Dungeon Unity 项目中的 Campaign（战役）系统，包括周目推进机制、庄园建筑系统、任务生成逻辑和城镇事件系统。

---

## 1. Campaign 类（战役进度管理）

### 1.1 核心数据结构

```csharp
public class Campaign
{
    // 进度追踪
    public int CurrentWeek { get; set; }                    // 当前周目周数
    public int QuestsComleted { get; set; }                  // 已完成任务数
    
    // 核心子系统
    public Estate Estate { get; set; }                       // 庄园/建筑系统
    public RealmInventory RealmInventory { get; set; }        // 背包/饰品库存
    public List<Hero> Heroes { get; set; }                   // 当前可用英雄列表
    public List<Quest> Quests { get; set; }                 // 本周可用的任务
    public List<string> CompletedPlot { get; set; }         // 已完成的剧情任务
    
    // 事件系统
    public TownEvent TriggeredEvent { get; set; }             // 触发的随机事件
    public TownEvent GuaranteedEvent { get; set; }          // 保证触发的事件（如任务完成奖励）
    public EventModifiers EventModifiers { get; set; }       // 事件效果修饰符
    
    // 进度追踪
    public Dictionary<string, DungeonProgress> Dungeons { get; set; }  // 地牢进度
    public List<WeekActivityLog> Logs { get; set; }          // 每周活动日志
    
    // 叙述系统
    public Dictionary<string, int> NarrationRaidInfo { get; set; }     // 地牢叙述数据
    public Dictionary<string, int> NarrationTownInfo { get; set; }     // 城镇叙述数据
    public Dictionary<string, int> NarrationCampaignInfo { get; set; } // 战役叙述数据
}
```

### 1.2 状态机流程

#### ExecuteProgress() - 每周进度执行

```
每周末执行 ExecuteProgress()，流程如下：

1. 事件效果应用
   ├── TownEventDataType.IdleBuff      → 所有空闲英雄获得BUFF
   ├── TownEventDataType.IdleResolve    → 指定职业英雄获得经验
   └── TownEventDataType.InActivityBuff → 活动中英雄获得BUFF

2. 状态重置
   ├── TriggeredEvent = null
   ├── GuaranteedEvent = null
   └── EventModifiers.Reset()

3. 庄园进度执行
   └── Estate.ExecuteProgress()

4. 随机事件判定
   └── RandomSolver.CheckSuccess(EventsOption.Frequency[3])
       ├── 成功 → 随机选择一个可能事件触发
       └── 失败 → 本周无事件

5. 任务生成
   └── GenerateQuests()

6. 失踪英雄检查
   └── SearchMissingHeroes()
```

#### AdvanceNextWeek() - 周推进

```
新周开始时调用 AdvanceNextWeek()：

1. CurrentWeek++

2. 任务完成检查
   └── RaidManager.Status == RaidStatus.Success
       └── CheckGuarantees(Quest) → 触发保证事件

3. 传令官部署
   ├── TriggeredEvent != null 或 GuaranteedEvent != null
   │   └── Estate.RedeployCrier()
   └── 否则
       └── Estate.KickCrier()

4. 清除来源为Estate的BUFF
   └── Heroes[i].RemoveAllBuffsWithSource(BuffSourceType.Estate)
```

### 1.3 任务完成判定逻辑

```csharp
private void CheckGuarantees(Quest completedQuest)
{
    // 检查是否存在针对该任务类型的保证事件
    if (EventsOption.Frequency[0] > 0)
    {
        var eventGuarantee = DarkestDungeonManager.Data.EventDatabase.Guarantees.Find(guarantee =>
            guarantee.Dungeon == completedQuest.Dungeon && 
            guarantee.QuestType == completedQuest.Type);

        if (eventGuarantee != null)
        {
            // 设置保证事件
            GuaranteedEvent = DarkestDungeonManager.Data.EventDatabase.Events.Find(guarantEvent =>
                guarantEvent.Id == eventGuarantee.EventId);
            EventModifiers.IncludeEvent(GuaranteedEvent);
        }
    }
}
```

---

## 2. Estate 系统（庄园/建筑系统）

### 2.1 建筑类型枚举

```csharp
public enum BuildingType { 
    Abbey,           // 修道院 - 压力恢复
    Tavern,          // 酒馆 - 压力恢复
    Sanitarium,      // 疗养院 - 疾病/怪癖管理
    Blacksmith,      // 铁匠 - 武器/护甲升级
    Guild,           // 行会 - 技能升级
    CampingTrainer,   // 露营训练师 - 露营技能
    NomadWagon,      // 流浪商车 - 饰品商店
    StageCoach,      // 驿站马车 - 招募新英雄
    Graveyard,       // 墓地 - 死亡记录
    Statue           // 雕像
}
```

### 2.2 货币系统

```csharp
public Dictionary<string, int> Currencies { get; private set; }
// Key: "gold"    → 金币
// Key: "bust"    → 半身像（传家宝）
// Key: "deed"    → 契约（传家宝）
// Key: "portrait"→ 肖像（传家宝）
// Key: "crest"   → 纹章（传家宝）
```

### 2.3 升级树管理

```csharp
public Dictionary<string, UpgradePurchases> TownPurchases { get; private set; }
// 全镇共享的升级购买记录

public Dictionary<int, Dictionary<string, UpgradePurchases>> HeroPurchases { get; private set; }
// Key: RosterId → 每位英雄的升级记录
```

---

## 3. 各建筑详细机制

### 3.1 StageCoach（驿站马车）

**功能**：招募新英雄

**数据结构**：
```csharp
public class StageCoach : Building
{
    public int BaseRecruitSlots { get; set; }    // 基础招募栏位
    public int RecruitSlots { get; set; }        // 当前招募栏位
    public int BaseRosterSlots { get; set; }      // 基础阵容大小
    public int RosterSlots { get; set; }          // 当前阵容大小
    
    public List<SlotUpgrade> RecruitSlotUpgrades { get; set; }     // 招募栏位升级
    public List<SlotUpgrade> RosterSlotUpgrades { get; set; }     // 阵容升级
    public List<RecruitUpgrade> RecruitExperienceUpgrades { get; set; }  // 经验升级
    
    public List<Hero> Heroes { get; set; }       // 本周可招募的英雄
    public List<Hero> EventHeroes { get; set; }   // 事件英雄（如从墓地复活）
    public List<int> GraveIndexes { get; set; }   // 对应的死亡记录索引
    
    private int CurrentRecruitMaxLevel { get; set; }  // 当前招募最大等级
}
```

**招募算法** (RestockHeroes)：
```
1. 检查是否有新英雄等级升级
   └── 遍历 RecruitExperienceUpgrades
       ├── 检查 CurrentRecruitMaxLevel
       └── 按概率决定是否应用经验升级

2. 随机选择英雄职业
   └── heroClasses[Random.Range(0, count)]

3. 分配RosterId
   └── 从 rosterIds 列表中分配唯一ID

4. 生成英雄名称
   └── hero_name_{Random.Range(0, 556)}

5. 初始化购买信息
   └── GeneratePurchaseInfo(hero, estate)
       ├── 初始化武器/护甲升级树
       ├── 初始化技能升级树
       └── 初始化露营技能
```

**经验升级概率机制**：
```csharp
// RecruitUpgrade 结构
public class RecruitUpgrade : ITownUpgrade
{
    public int Level { get; set; }                    // 获得的等级
    public float Chance { get; set; }                // 触发概率
    public int ExtraPositiveQuirks { get; set; }     // 额外正面怪癖
    public int ExtraNegativeQuirks { get; set; }     // 额外负面怪癖
    public int ExtraCombatSkills { get; set; }       // 额外战斗技能
    public int ExtraCampingSkills { get; set; }     // 额外露营技能
}

// 判定逻辑：按等级从低到高遍历，一旦命中概率则应用
for (int j = 0; j <= RecruitExperienceUpgrades.Count - 1; j++)
{
    if (RecruitExperienceUpgrades[j].Level <= CurrentRecruitMaxLevel &&
        RandomSolver.CheckSuccess(RecruitExperienceUpgrades[j].Chance))
    {
        experienceUpgrade = RecruitExperienceUpgrades[j];
        break;
    }
}
```

**墓地复活机制** (RestockFromGrave)：
```
当特定事件触发时，可从墓地复活英雄：
1. 从 Graveyard.Records 获取死亡记录
2. 匹配英雄职业
3. 创建新英雄，保留原名和职业
4. 存入 EventHeroes 列表
```

---

### 3.2 Guild（行会）

**功能**：技能升级

**数据结构**：
```csharp
public class Guild : Building
{
    public List<DiscountUpgrade> DiscountUpgrades { get; private set; }
    public float Discount { get; private set; }  // 累计折扣百分比
}
```

**技能等级解锁机制**：
```
每个英雄职业有一套技能升级树：
- 树ID格式：{classId}.{skillId}，如 "crusader.smite"
- 每个技能最多3-4个等级
- 等级解锁前置条件：Resolve Level（决心等级）

// HeroUpgrade 结构
public class HeroUpgrade : TownUpgrade
{
    public int PrerequisiteResolveLevel { get; set; }  // 解锁需要的决心等级
}
```

**技能等级计算**：
```csharp
public int GetUpgradedSkillLevel(int rosterId, string classId, string skillId)
{
    if (HeroPurchases.ContainsKey(hero.RosterId))
        return HeroPurchases[hero.RosterId][classId + "." + skillId].PurchasedUpgrades.Count - 1;
    return -1;  // 未解锁任何等级
}
```

---

### 3.3 Blacksmith（铁匠）

**功能**：装备升级（武器/护甲）

**数据结构**：
```csharp
public class Blacksmith : Building
{
    public List<DiscountUpgrade> DiscountUpgrades { get; private set; }
    public float Discount { get; private set; }
}
```

**升级等级计算**：
```csharp
public int GetUpgradedWeaponLevel(int rosterId, string classId)
{
    if (HeroPurchases.ContainsKey(rosterId))
        return HeroPurchases[rosterId][classId + ".weapon"].PurchasedUpgrades.Count + 1;
    return 1;  // 基础等级为1
}

public int GetUpgradedArmorLevel(int rosterId, string classId)
{
    if (HeroPurchases.ContainsKey(rosterId))
        return HeroPurchases[rosterId][classId + ".armour"].PurchasedUpgrades.Count + 1;
    return 1;
}
```

**材料消耗机制**：
```
升级消耗由 UpgradeTree 定义：
- 武器/护甲各有独立的升级树
- 每级消耗不同的传家宝和金币
- 升级树结构：
  ├── weapon.weapon (武器升级)
  ├── blacksmith.armour (护甲升级)
  └── blacksmith.cost (成本降低)
```

---

### 3.4 Abbey & Tavern（修道院/酒馆）

**功能**：压力恢复

**共同基类**：
```csharp
public abstract class ActivityBuilding : Building
{
    public List<TownActivity> Activities { get; private set; }
}
```

**TownActivity 结构**：
```csharp
public class TownActivity
{
    public string Id { get; set; }
    public string TreeId { get; set; }          // 关联的升级树ID
    
    public int BaseSlots { get; set; }           // 基础栏位数
    public int NumberOfSlots { get; set; }       // 当前栏位数
    public int BaseStressHeal { get; set; }      // 基础压力恢复量
    public int StressHealAmount { get; set; }    // 当前压力恢复量
    
    public CurrencyCost BaseCost { get; set; }   // 基础费用
    public CurrencyCost ActivityCost { get; set; } // 当前费用
    
    public float SideEffectChance { get; set; }  // 副作用概率
    public List<TownEffect> SideEffects { get; set; }  // 可能的副作用
    
    public List<ActivitySlot> ActivitySlots { get; set; }  // 活动栏位
}
```

**活动类型枚举**：
```csharp
// Abbey 活动
ActivityType.MeditationStressHeal    // 冥想
ActivityType.PrayerStressHeal        // 祈祷
ActivityType.FlagellationStressHeal // 鞭笞

// Tavern 活动
ActivityType.BarStressHeal          // 酒吧
ActivityType.GambleStressHeal        // 赌博
ActivityType.BrothelStressHeal       // 妓院
```

**压力恢复流程** (ProvideActivity)：
```
每周活动结算时：
1. 遍历所有 ActivitySlot
2. 根据 Status 执行不同逻辑：
   ├── Caretaken/Crierd → 重置为 Available
   ├── Blocked/Checkout → 英雄状态恢复 Available
   └── Paid → 执行压力恢复
       ├── 扣除 StressHealAmount
       ├── 检查副作用 (RandomSolver.CheckSuccess(SideEffectChance))
       └── 应用随机副作用
```

**副作用类型** (TownEffectType)：
```csharp
TownEffectType.GoMissing          // 英雄失踪N周
TownEffectType.ActivityLock      // 锁定活动栏位
TownEffectType.AddQuirk          // 获得随机怪癖
TownEffectType.AddTrinket        // 获得随机饰品
TownEffectType.RemoveTrinket     // 失去已装备的饰品
TownEffectType.ApplyBuff         // 获得BUFF
TownEffectType.ChangeCurrency     // 获得/失去金币
```

---

### 3.5 Sanitarium（疗养院）

**功能**：疾病和怪癖的管理

**数据结构**：
```csharp
public class Sanitarium : Building
{
    public QuirkTreatmentActivity QuirkActivity { get; set; }      // 怪癖治疗
    public DiseaseTreatmentActivity DiseaseActivity { get; set; }  // 疾病治疗
}
```

#### 3.5.1 QuirkTreatmentActivity（怪癖治疗）

**治疗费用结构**：
```csharp
public class QuirkTreatmentActivity
{
    public CurrencyCost PositiveQuirkCost { get; set; }     // 锁定正面怪癖费用
    public CurrencyCost NegativeQuirkCost { get; set; }     // 移除负面怪癖费用
    public CurrencyCost PermNegativeQuirkCost { get; set; } // 永久移除负面怪癖费用
    
    public float QuirkTreatmentChance { get; set; }         // 治疗成功率
    
    public int BaseQuirkSlots { get; set; }                  // 基础治疗栏位
    public int QuirkSlots { get; set; }                      // 当前治疗栏位
    
    public List<TreatmentSlot> TreatmentSlots { get; set; } // 治疗栏位
}
```

**TreatmentSlot 结构**：
```csharp
public class TreatmentSlot : ActivitySlot
{
    public string TargetPositiveQuirk { get; set; }  // 要锁定的正面怪癖
    public string TargetNegativeQuirk { get; set; }  // 要移除的负面怪癖
}
```

**治疗操作**：
```
1. 移除负面怪癖 (RemoveQuirk)
   └── 从英雄怪癖列表中删除

2. 锁定正面怪癖 (LockQuirk)
   └── 防止在任务中获得新的怪癖
   └── 已锁定的怪癖不会被覆盖

3. 永久移除 (PermNegativeQuirkCost)
   └── 从所有可能怪癖池中排除
```

#### 3.5.2 DiseaseTreatmentActivity（疾病治疗）

**治疗费用结构**：
```csharp
public class DiseaseTreatmentActivity
{
    public CurrencyCost DiseaseTreatmentCost { get; set; }  // 单个疾病治疗费用
    public float CureAllChance { get; set; }               // 全治愈概率
    
    public int BaseDiseaseSlots { get; set; }               // 基础疾病治疗栏位
    public int DiseaseSlots { get; set; }                   // 当前疾病治疗栏位
}
```

**治疗流程**：
```
1. 检查 CureAllChance
   ├── 成功 → RemoveDiseases() 移除所有疾病
   └── 失败 → RemoveQuirk() 移除单个疾病
```

---

### 3.6 NomadWagon（流浪商车）

**功能**：神秘商人 - 饰品交易

**数据结构**：
```csharp
public class NomadWagon : Building
{
    public int BaseTrinketSlots { get; set; }        // 基础饰品栏位
    public int TrinketSlots { get; private get; set; } // 当前饰品栏位
    
    public List<Trinket> Trinkets { get; private set; }  // 可购买的饰品
    
    public List<SlotUpgrade> TrinketSlotUpgrades { get; private set; }  // 栏位升级
    public List<DiscountUpgrade> DiscountUpgrades { get; private set; }  // 折扣升级
    
    private List<GeneratedRarity> RarityTable { get; set; }  // 稀有度表
}
```

**稀有度表**：
```csharp
private List<GeneratedRarity> RarityTable { get; set; } = new List<GeneratedRarity>()
{
    new GeneratedRarity() { Chance = 1, RarityId = "very_common" },  // 非常普通
    new GeneratedRarity() { Chance = 1, RarityId = "common" },      // 普通
    new GeneratedRarity() { Chance = 1, RarityId = "uncommon" },     // 不常见
    new GeneratedRarity() { Chance = 1, RarityId = "rare" },         // 稀有
    new GeneratedRarity() { Chance = 1, RarityId = "very_rare" },    // 非常稀有
};
```

**饰品生成算法** (RestockTrinkets)：
```
每周开始时重新生成饰品：
1. 清空当前饰品列表
2. 获取所有饰品数据

3. 循环 N 次 (N = TrinketSlots):
   ├── 按概率选择稀有度 (RandomSolver.ChooseByRandom)
   ├── 从该稀有度列表随机选择饰品
   └── 添加到 Trinkets

4. 按价格降序排序
```

**饰品折扣机制**：
```
- DiscountUpgrades 提供百分比折扣
- 最终价格 = 原价 * (1 - Discount)
- 折扣可叠加
```

---

## 4. UpgradeTree 升级树系统

### 4.1 数据结构

```csharp
public class UpgradeTree
{
    public string Id { get; set; }                    // 升级树唯一ID
    public bool IsInstanced { get; set; }             // 是否为实例化（英雄专属）
    public List<string> Tags { get; set; }           // 标签，用于分类
    public List<TownUpgrade> Upgrades { get; set; }  // 升级项列表
}
```

### 4.2 升级项类型

#### TownUpgrade（城镇升级）
```csharp
public class TownUpgrade
{
    public string Code { get; set; }
    public List<CurrencyCost> Cost { get; set; }              // 费用
    public List<PrerequisiteReqirement> Prerequisites { get; set; }  // 前置条件
}
```

#### HeroUpgrade（英雄升级）
```csharp
public class HeroUpgrade : TownUpgrade
{
    public int PrerequisiteResolveLevel { get; set; }  // 解锁决心等级要求
}
```

#### 特殊升级类型
```csharp
// 栏位升级
public class SlotUpgrade : ITownUpgrade
{
    public int NumberOfSlots { get; set; }
}

// 费用降低升级
public class CostUpgrade : ITownUpgrade
{
    public CurrencyCost Cost { get; set; }
}

// 折扣升级
public class DiscountUpgrade : ITownUpgrade
{
    public float Percent { get; set; }
}

// 露营技能升级
public class StressUpgrade : ITownUpgrade
{
    public int StressHeal { get; set; }
}

// 治疗概率升级
public class ChanceUpgrade : ITownUpgrade
{
    public float Chance { get; set; }
}
```

### 4.3 升级状态枚举

```csharp
public enum UpgradeStatus { 
    Purchased,   // 已购买
    Available,   // 可购买
    Locked      // 未解锁
}
```

### 4.4 升级树配置示例

```
建筑升级树ID格式：
├── abbey.meditation          (冥想活动)
├── abbey.prayer               (祈祷活动)
├── abbey.flagellation         (鞭笞活动)
├── tavern.bar                 (酒吧活动)
├── tavern.gambling            (赌博活动)
├── tavern.brothel             (妓院活动)
├── sanitarium.cost            (治疗费用)
├── sanitarium.disease_quirk_cost  (疾病/怪癖费用)
├── sanitarium.slots           (治疗栏位)
├── blacksmith.weapon          (武器升级)
├── blacksmith.armour          (护甲升级)
├── blacksmith.cost             (升级费用降低)
├── guild.skill_levels         (技能等级)
├── guild.cost                  (技能费用降低)
├── nomad_wagon.numitems        (饰品数量)
├── nomad_wagon.cost            (饰品价格)
├── stage_coach.numrecruits     (招募数量)
├── stage_coach.rostersize      (阵容大小)
└── stage_coach.upgraded_recruits (升级招募)
```

---

## 5. TownEvent 城镇事件系统

### 5.1 事件语调枚举

```csharp
public enum TownEventTone
{
    Good,     // 好事件
    Bad,      // 坏事件
    Neutral   // 中性事件
}
```

### 5.2 事件数据类型

```csharp
public enum TownEventDataType
{
    EmbarkPartyBuff,              // 出征队伍BUFF
    IdleResolve,                  // 空闲经验
    BonusRecruit,                 // 额外招募
    InActivityBuff,               // 活动中BUFF
    ActivityLock,                 // 活动锁定
    ActivityCostChange,           // 活动费用变化
    ProvisionTypeCostChange,      // 补给费用变化
    ProvisionTypeAmountChange,    // 补给数量变化
    UpgradeTagDiscount,            // 升级折扣
    FreeActivity,                 // 免费活动
    DeadRecruit,                  // 死亡招募（复活）
    NoLevelRestriction,           // 无等级限制
    UpgradeTagFree,               // 免费升级标签
    IdleBuff,                     // 空闲BUFF
    PlotQuest                     // 剧情任务
}
```

### 5.3 TownEvent 结构

```csharp
public class TownEvent : ISingleProportion
{
    public string Id { get; set; }
    public TownEventTone Tone { get; set; }          // 事件语调
    public int Cooldown { get; set; }                 // 冷却周数
    public float Chance { get; set; }                 // 基础触发概率
    public float ChancePerNotRolled { get; set; }     // 未触发时的概率增量
    public int MinimumWeek { get; set; }              // 最小周数要求
    
    public int DeadHeroes { get; set; }               // 需要的死亡英雄数
    public Dictionary<int, int> LevelHeroes { get; set; }  // 需要的等级英雄数
    public Dictionary<string, string> Purchases { get; set; }  // 需要的已购买升级
    
    public List<TownEventData> Data { get; set; }    // 事件效果数据
}
```

### 5.4 事件触发机制

#### IsPossible 属性（触发条件检查）
```
1. 检查概率
   └── Chance == 0 或 activeCooldown > 0 → 不可触发

2. 检查周数
   └── MinimumWeek > CurrentWeek → 不可触发

3. 检查死亡英雄数
   └── DeadHeroes > Graveyard.Records.Count → 不可触发

4. 检查等级要求
   └── LevelHeroes[等级] > 满足该等级的英雄数 → 不可触发

5. 检查升级要求
   └── Purchases[树ID] != 购买记录 → 不可触发

6. 检查剧情条件
   └── Data包含PlotQuest且已完成 → 不可触发

全部满足 → 可能触发
```

#### 触发概率计算
```csharp
public float Chance
{
    get
    {
        // 最终概率 = 基础概率 + (未触发周数 * 每次增量)
        return baseChance + ChancePerNotRolled * notRolledAmount;
    }
}
```

#### 事件冷却机制
```
1. 事件被触发时：
   ├── activeCooldown = Cooldown
   └── notRolledAmount = 0

2. 事件未被触发时：
   ├── if (ChancePerNotRolled != 0) notRolledAmount++
   └── if (activeCooldown > 0) activeCooldown--
```

### 5.5 事件保证机制

```csharp
public class TownEventGuarantee
{
    public string Dungeon { get; set; }         // 地牢ID
    public string QuestType { get; set; }        // 任务类型
    public string EventId { get; set; }         // 保证触发的事件ID
}
```

**保证事件触发流程**：
```
任务完成后触发 CheckGuarantees():
1. 查找匹配的任务保证
2. 找到对应事件
3. 设置 GuaranteedEvent
4. 应用事件效果
```

### 5.6 EventModifiers 事件修饰符

```csharp
public class EventModifiers : IBinarySaveData
{
    public bool NoLevelRestrictions { get; set; }
    
    public Dictionary<string, bool> ActivityLocks { get; set; }            // 活动锁定
    public Dictionary<string, bool> FreeActivities { get; set; }          // 免费活动
    public Dictionary<string, float> ActivityCostModifiers { get; set; } // 活动费用修正
    public Dictionary<string, float> ProvisionCostModifiers { get; set; } // 补给费用修正
    public Dictionary<string, float> ProvisionAmountModifiers { get; set; }// 补给数量修正
    public Dictionary<string, float> UpgradeTagCostModifiers { get; set; } // 升级费用折扣
    public Dictionary<string, int> FreeUpgradeTags { get; set; }           // 免费升级标签
    
    public List<TownEventData> EventData { get; set; }                    // 事件数据列表
}
```

---

## 6. QuestGenerator 任务生成器

### 6.1 生成流程概述

```
QuestGenerator.GenerateQuests() 执行以下步骤：

1. GetQuestInfo()        → 收集可用地牢和任务数量
2. DistributeQuests()   → 分配任务到各难度
3. DistributeQuestTypes()→ 分配任务类型和长度
4. DistributeQuestGoals()→ 分配任务目标
5. DistributeQuestRewards() → 分配任务奖励
```

### 6.2 任务数量计算

```csharp
private static int GetQuestNumber(QuestGenerationData genData, Campaign campaign)
{
    int questNumberState;
    
    if (campaign.QuestsComleted <= 2)
        questNumberState = 0;
    else if (campaign.QuestsComleted <= 3)
        questNumberState = 1;
    else if (campaign.QuestsComleted <= 4)
        questNumberState = 2;
    else if (campaign.QuestsComleted <= 6)
        questNumberState = 3;
    else if (campaign.QuestsComleted <= 10)
        questNumberState = 4;
    else if (campaign.QuestsComleted <= 16)
        questNumberState = 5;
    else if (campaign.QuestsComleted <= 20)
        questNumberState = 6;
    else
        questNumberState = 7;
        
    return genData.QuestsPerVisit[questNumberState];
}
```

### 6.3 难度分配算法

```
根据可用英雄的决心等级分配任务难度：
1. 统计各难度等级的可用英雄数
   ├── Difficulty 1 → ResolveLevels[0]
   ├── Difficulty 3 → ResolveLevels[1]
   └── Difficulty 5 → ResolveLevels[2]

2. 按比例分配任务数量
   ├── difOnes = difOneAvailable / allAvailable * totalQuests
   ├── difTwos = difTwoAvailable / allAvailable * totalQuests
   └── difThrees = difThreeAvailable / allAvailable * totalQuests

3. 填充剩余任务
```

### 6.4 任务类型分配

```csharp
private static void DistributeQuestTypes(QuestGenerationInfo questInfo)
{
    foreach(var dungeonInfo in questInfo.DungeonQuests)
    {
        foreach(var quest in dungeonInfo.Quests)
        {
            if (quest.IsPlotQuest)
                continue;
                
            // 根据地牢熟练度等级选择任务类型集合
            GeneratedQuestType questType = RandomSolver.ChooseByRandom(dungeonInfo.GeneratedTypes);
            quest.Type = questType.Type;
            quest.Length = questType.Length;
        }
    }
}
```

### 6.5 任务目标分配

```
任务目标根据 QuestType 和地牢确定：
1. 获取 QuestType 的所有目标列表
2. 筛选适用于当前地牢的目标
3. 随机选择目标
```

### 6.6 Quest 任务结构

```csharp
public class Quest : IBinarySaveData
{
    public bool IsPlotQuest { get; set; }           // 是否剧情任务
    public string Type { get; set; }                // 任务类型
    public string Dungeon { get; set; }             // 地牢ID
    public int Difficulty { get; set; }            // 难度等级
    public int Length { get; set; }                 // 任务长度
    public QuestGoal Goal { get; set; }            // 任务目标
    public CompletionReward Reward { get; set; }   // 奖励
    
    // 地牢设置
    public bool IsProgression { get; set; }        // 是否推进进度
    public bool HasStatueContents { get; set; }    // 是否有雕像内容
    public bool CanRetreat { get; set; }            // 是否可撤退
    public bool IsSurpriseEnabled { get; set; }     // 是否启用突袭
    public bool IsScoutingEnabled { get; set; }    // 是否启用侦察
    
    // 失败惩罚
    public int RosterBuffOnFailureMinimumPartyResolveLevel { get; set; }
    public List<Buff> RosterBuffsOnFailure { get; set; }
}
```

### 6.7 任务目标类型

```csharp
// 击杀怪物目标
public class QuestKillMonsterData : IQuestData
{
    public List<string> MonsterNameIds { get; set; }  // 怪物ID列表
    public int Amount { get; set; }                   // 击杀数量
    
    public bool IsQuestCompleted()
    {
        foreach (var monsterId in MonsterNameIds)
            if (RaidSceneManager.Raid.KilledMonsters.Contains(monsterId))
                return true;
        return false;
    }
}

// 收集物品目标
public class QuestGatherData : IQuestData
{
    public string CurioName { get; set; }      // 调查的秘宝名称
    public ItemDefinition Item { get; set; }   // 需要收集的物品
    
    public bool IsQuestCompleted()
    {
        return RaidSceneManager.Inventory.ContainsEnoughItems(Item);
    }
}
```

### 6.8 任务奖励结构

```csharp
public class CompletionReward
{
    public int ResolveXP { get; set; }                          // 决心经验
    public List<ItemDefinition> ItemDefinitions { get; set; }  // 物品奖励
}

// ItemDefinition 结构
public class ItemDefinition
{
    public string Id { get; set; }     // 物品ID
    public string Type { get; set; }  // 物品类型 ("heirloom", "trinket", etc.)
    public int Amount { get; set; }    // 数量
}
```

**奖励分配逻辑**：
```
根据地牢、难度、长度计算奖励：
1. ResolveXP → 从 ResolveXpReward[Difficulty][Length] 获取
2. Heirloom One → 从 HeirloomTypes[Dungeon][0] 和 HeirloomAmounts 获取
3. Heirloom Two → 从 HeirloomTypes[Dungeon][1] 和 HeirloomAmounts 获取
4. 普通物品 → 从 ItemTable[Difficulty][Length] 获取
5. 饰品 → 根据 TrinketChances 概率判定
```

---

## 7. Heirloom 传家宝系统

### 7.1 传家宝类型

| 类型 | 英文名 | 主要用途 |
|------|--------|----------|
| 半身像 | bust | 疗养院升级 |
| 契约 | deed | 城镇设施升级 |
| 肖像 | portrait | 特定升级 |
| 纹章 | crest | 特定升级 |

### 7.2 传家宝获取

```csharp
// 通过任务奖励获取
reward.ItemDefinitions.Add(new ItemDefinition()
{
    Type = "heirloom",
    Id = heirloomType,  // "bust", "deed", "portrait", "crest"
    Amount = amount
});

// 庄园添加方法
public void AddHeirlooms(int crest, int deed, int portrait, int bust)
{
    Currencies["crest"] = Mathf.Clamp(Currencies["crest"] + crest, 0, int.MaxValue);
    Currencies["deed"] = Mathf.Clamp(Currencies["deed"] + deed, 0, int.MaxValue);
    Currencies["portrait"] = Mathf.Clamp(Currencies["portrait"] + portrait, 0, int.MaxValue);
    Currencies["bust"] = Mathf.Clamp(Currencies["bust"] + bust, 0, int.MaxValue);
}
```

### 7.3 传家宝交易配置

```csharp
public class HeirloomExchange
{
    public string FromType { get; set; }   // 源类型
    public int FromAmount { get; set; }     // 源数量
    public string ToType { get; set; }      // 目标类型
    public int ToAmount { get; set; }       // 目标数量
}
```

---

## 8. RealmInventory 背包/饰品库存

### 8.1 数据结构

```csharp
public class RealmInventory
{
    public int MaxCapacity { get; set; }
    public List<Trinket> Trinkets { get; set; }  // 存储的饰品列表
}
```

### 8.2 饰品添加机制

```csharp
public void AddTrinket(Trinket trinket)
{
    if (TrinketAddAction != null)
        TrinketAddAction(trinket);  // 触发外部回调
    else
        Trinkets.Add(trinket);      // 直接添加
}
```

---

## 9. Graveyard 墓地系统

### 9.1 死亡记录结构

```csharp
public class DeathRecord : IBinarySaveData
{
    public string HeroName { get; set; }       // 英雄名称
    public int HeroClassIndex { get; set; }     // 英雄职业索引
    public string KillerName { get; set; }     // 击杀者名称
    public int ResolveLevel { get; set; }      // 死亡时决心等级
    public DeathFactor Factor { get; set; }    // 死亡因素
}
```

### 9.2 死亡因素枚举

```csharp
public enum DeathFactor
{
    Hunger,           // 饥饿
    Trap,            // 陷阱
    Obstacle,        // 障碍物
    AttackMonster,    // 怪物攻击
    BleedMonster,     // 怪物流血
    PoisonMonster,    // 怪物中毒
    AttackFriend,     // 队友攻击
    BleedFriend,      // 队友流血
    PoisonFriend,    // 队友中毒
    PoisonUnknown,   // 未知中毒
    BleedUnknown,    // 未知流血
    Unknown,         // 未知
    CaptorMonster,   // 被捕获
    HeartAttack      // 心脏骤停（压力过高）
}
```

---

## 10. 关键类图

```
Campaign
├── Estate
│   ├── Abbey (ActivityBuilding)
│   │   └── TownActivity[] (冥想/祈祷/鞭笞)
│   ├── Tavern (ActivityBuilding)
│   │   └── TownActivity[] (酒吧/赌博/妓院)
│   ├── Sanitarium
│   │   ├── QuirkTreatmentActivity
│   │   └── DiseaseTreatmentActivity
│   ├── Blacksmith
│   ├── Guild
│   ├── NomadWagon
│   ├── StageCoach
│   ├── CampingTrainer
│   ├── Graveyard
│   │   └── DeathRecord[]
│   └── Statue
├── RealmInventory
│   └── Trinket[]
├── List<Hero>
├── List<Quest>
├── List<WeekActivityLog>
├── TownEvent (TriggeredEvent)
├── TownEvent (GuaranteedEvent)
├── EventModifiers
└── Dictionary<string, DungeonProgress>
```

---

## 11. 每周游戏循环

```
┌─────────────────────────────────────────────────────────────────┐
│                        周末结算 (Week End)                      │
├─────────────────────────────────────────────────────────────────┤
│ 1. ExecuteProgress()                                            │
│    ├─ 应用事件效果 (IdleBuff, IdleResolve, InActivityBuff)      │
│    ├─ 重置事件状态                                              │
│    ├─ 庄园进度执行                                             │
│    │   ├─ NomadWagon.RestockTrinkets()                         │
│    │   ├─ StageCoach.RestockHeroes()                           │
│    │   ├─ Abbey.ProvideActivity() (压力恢复结算)                │
│    │   ├─ Tavern.ProvideActivity() (压力恢复结算)              │
│    │   └─ Sanitarium.ProvideActivity() (疾病/怪癖治疗结算)     │
│    ├─ 随机事件判定                                             │
│    └─ GenerateQuests()                                         │
│                                                                  │
│ 2. AdvanceNextWeek()                                           │
│    ├─ CurrentWeek++                                            │
│    ├─ 任务完成检查 → 触发保证事件                              │
│    ├─ 传令官部署/移除                                          │
│    └─ 清除来源为Estate的BUFF                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        新周开始 (Week Start)                    │
├─────────────────────────────────────────────────────────────────┤
│ - 显示新任务                                                    │
│ - 应用保证事件效果                                              │
│ - 玩家进行英雄管理：                                             │
│   ├─ 招募新英雄 (StageCoach)                                   │
│   ├─ 装备升级 (Blacksmith)                                     │
│   ├─ 技能升级 (Guild)                                          │
│   ├─ 压力恢复 (Abbey/Tavern)                                   │
│   ├─ 疾病/怪癖治疗 (Sanitarium)                               │
│   └─ 饰品购买 (NomadWagon)                                     │
│                                                                  │
│ - 选择任务并派遣队伍                                            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        任务执行 (Raid)                          │
├─────────────────────────────────────────────────────────────────┤
│ - 进入地牢                                                      │
│ - 完成目标                                                      │
│ - 撤离或全员死亡                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                              (返回周末结算)
```

---

## 12. 文件路径参考

| 源文件 | 功能 |
|--------|------|
| `Assets/Scripts/Campaign/Campaign.cs` | 战役主控制器 |
| `Assets/Scripts/Campaign/Estate.cs` | 庄园系统 |
| `Assets/Scripts/Campaign/Town/StageCoach.cs` | 驿站马车 |
| `Assets/Scripts/Campaign/Town/Guild.cs` | 行会 |
| `Assets/Scripts/Campaign/Town/Blacksmith.cs` | 铁匠 |
| `Assets/Scripts/Campaign/Town/Abbey.cs` | 修道院 |
| `Assets/Scripts/Campaign/Town/Tavern.cs` | 酒馆 |
| `Assets/Scripts/Campaign/Town/Sanitarium.cs` | 疗养院 |
| `Assets/Scripts/Campaign/Town/NomadWagon.cs` | 流浪商车 |
| `Assets/Scripts/Campaign/Town/TownActivity.cs` | 活动系统 |
| `Assets/Scripts/Campaign/Town/ActivityBuilding.cs` | 活动建筑基类 |
| `Assets/Scripts/Campaign/Town/QuirkTreatmentActivity.cs` | 怪癖治疗 |
| `Assets/Scripts/Campaign/Town/DiseaseTreatmentActivity.cs` | 疾病治疗 |
| `Assets/Scripts/Campaign/Town/Graveyard.cs` | 墓地 |
| `Assets/Scripts/Campaign/Town/DeathRecord.cs` | 死亡记录 |
| `Assets/Scripts/Campaign/TownEvent.cs` | 城镇事件 |
| `Assets/Scripts/Campaign/EventModifiers.cs` | 事件修饰符 |
| `Assets/Scripts/Campaign/Upgrade.cs` | 升级系统 |
| `Assets/Scripts/Campaign/UpgradeTree.cs` | 升级树 |
| `Assets/Scripts/Generation/QuestGenerator.cs` | 任务生成器 |
| `Assets/Scripts/Campaign/Quests/Quest.cs` | 任务定义 |
| `Assets/Scripts/Campaign/RealmInventory.cs` | 背包系统 |
| `Assets/Scripts/Campaign/HeirloomExchange.cs` | 传家宝交易 |
