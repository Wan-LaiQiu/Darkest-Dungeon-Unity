---
name: project_doc_directory
description: 项目文档统一存放在 Assets\Doc 目录，包含架构、角色、战斗、战役等模块文档
type: project
---

项目文档统一存放在 `Assets\Doc` 目录

**为什么：**
项目采用 Unity 开发，文档作为资源文件与代码并列管理，便于版本控制和打包发布。

**包含文档：**
- `Architecture.md` - 系统架构文档
- `Campaign.md` - 战役系统文档
- `Character.md` - 角色系统文档
- `Combat.md` / `CombatMechanics.md` - 战斗及战斗机制文档
- `Database.md` - 数据库文档
- `DungeonGeneration.md` - 地牢生成文档
- `Index.md` - 文档索引
- `ProjectOverview.md` - 项目概览
- `Raid.md` - 突袭系统文档
- `SkillSystem.md` - 技能系统文档
- `UI.md` - UI 系统文档

**如何应用：**
当用户要求"生成文档"、"更新文档"、"分析代码并写文档"时，文档输出路径应指向 `Assets\Doc\` 目录。
