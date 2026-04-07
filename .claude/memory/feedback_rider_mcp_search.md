---
name: rider_mcp_search_symbol
description: Rider MCP search_symbol 工具使用规范 - 不带 class 前缀
type: feedback
---

## 规则：Rider MCP `search_symbol` 搜索时不带修饰符前缀

**问题：** 使用 `search_symbol` 时带 `class` 前缀（如 `"class BattleSolver"`）会返回空结果。

**原因：** 该工具不支持前缀匹配，只识别纯标识符名称。

**正确用法：**
- ✅ `"BattleSolver"` → 找到结果
- ❌ `"class BattleSolver"` → 空结果
- ✅ `"Skill"` → 找到 Skill 类
- ❌ `"class Skill"` → 空结果

**Why:** Rider MCP 的语义搜索基于标识符名称，"class" 是语言语法，不是符号名称的一部分。

**How to apply:** 今后在使用 `mcp__rider-mcp__search_symbol` 时，始终传入纯类名/接口名，不带 `class`、`interface`、`struct` 等修饰符。
