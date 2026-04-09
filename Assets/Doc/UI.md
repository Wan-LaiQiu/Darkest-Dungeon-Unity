# UI 与交互系统 (UI System)

本项目采用了基于面板 (Panel) 和 窗口 (Window) 的层级化架构，通过 `DarkestDungeonManager` 提供的材质资源实现统一的视觉风格。

---

## 1. UI 架构 (Panel & Window 系统)

### 1.1 核心组件分类
- **Panels (面板)**: 常驻或半常驻的 UI 块。
    - `RaidPanel`: 地牢探索时的底部主控制栏。
    - `HeroPanel`: 英雄列表与简易状态。
    - `MapPanel`: 右下角的地牢缩略图。
- **Windows (窗口)**: 模态或非模态的弹出界面。
    - `CharacterWindow`: 英雄详细数值、怪癖、技能管理。
    - `ProvisionWindow`: 出征前的补给购买界面。
    - `QuestCompletionWindow`: 任务结束后的结算总结。

### 1.2 材质驱动的视觉逻辑
UI 的高亮和置灰效果不是通过简单的颜色叠加，而是通过 Shader 材质切换实现的。
```csharp
// DarkestDungeonManager.cs 中的全局材质引用
public static Material HighlightMaterial => Instanse.sharedHighlight;
public static Material GrayMaterial => Instanse.sharedGrayscale;
public static Material DeactivatedMaterial => Instanse.sharedDeactivated;

// 在 UI 脚本中使用示例
public void SetHighlight(bool highlight)
{
    targetImage.material = highlight ? DarkestDungeonManager.HighlightMaterial : null;
}
```

---

## 2. 核心交互机制

### 2.1 拖拽管理 (DragManager)
饰品的装备、英雄位置的调整均通过 `DragManager` 统一处理。
- **IDragHandler**: 实现 Unity 的拖拽接口。
- **Slot 校验**: 拖拽结束时，目标 Slot 会校验对象是否合法（如：饰品是否只能装在饰品槽）。

### 2.2 战斗飘字与弹出信息
在战斗中，数值变动会触发 `PopupMessage`。
```csharp
// 典型的战斗信息弹出调用
public void ShowCombatText(string text, Color color, Vector3 worldPosition)
{
    var popup = PopupPool.GetNext();
    popup.SetText(text, color);
    popup.transform.position = Camera.main.WorldToScreenPoint(worldPosition);
}
```

---

## 3. 实时状态更新 (Data Binding)

UI 与数据层通过事件或每帧轮询的方式保持同步：
- **生命/压力条**: 监听 `PairedAttribute` 的变化，动态调整 Slider 的 `value`。
- **状态图标 (Status Icons)**: 遍历英雄的 `Buffs` 和 `StatusEffects` 列表，动态生成对应的小图标。

---

## 4. UI 性能优化
- **对象池 (Object Pooling)**: 用于大量的战斗飘字、伤害数字和地图小方块。
- **精灵图集 (Sprite Atlas)**: 所有的 UI 元素（图标、框体）都打包在图集中以减少 DrawCall。

---
[返回主页](Index.md)
