# Godot 可编辑 RPG 地图（溪流神社）设计文档

## 目标

在当前仓库中生成一个**可复制进任意 Godot 4 项目**的独立场景包，包含：

- **分离 props**（Sprite2D）
- **遭遇草丛区域**（Area2D）
- **碰撞阻挡物**（StaticBody2D + CollisionShape2D）
- **出口区域**（Area2D，支持携带出口 id）
- **debug player 场景**（可移动、可触发区域回调）

本次输出以“可编辑、可验证”为第一优先级，不依赖从像素图推导碰撞。

## 范围与非目标

### 范围

- 地图主题：**溪流神社（river shrine）**
- 地图规模：**32x24 tiles**（用于组织布局；实际底图为像素图）
- 输出：独立场景包 + 生成素材 + 元数据 JSON + QA 预览图

### 非目标

- 不创建完整 Godot 项目（无 `project.godot`）
- 不实现战斗、随机遭遇逻辑、存档系统
- 不生成角色/NPC/怪物/技能动画资产（属于 `$generate2dsprite` 的角色资产范畴）

## 交付物（文件结构）

输出根目录（建议，可调整）：`output/river-shrine-godot/`

- `output/river-shrine-godot/assets/map/river-shrine-base.png`
- `output/river-shrine-godot/assets/map/river-shrine-base.prompt.txt`
- `output/river-shrine-godot/assets/map/river-shrine-preview.png`（QA 预览，非运行时依赖）
- `output/river-shrine-godot/assets/props/<prop_id>/prop.png`
- `output/river-shrine-godot/godot/river_shrine.tscn`
- `output/river-shrine-godot/godot/debug_player.tscn`
- `output/river-shrine-godot/godot/debug_player.gd`
- `output/river-shrine-godot/data/river-shrine-objects.json`

## 地图运行时对象模型（Godot 节点约定）

`river_shrine.tscn`（根节点：`Node2D`）

- `Background`（Sprite2D）
  - 显示 `river-shrine-base.png`
- `Props`（Node2D）
  - 多个 `Sprite2D` 子节点，按 y-sort 需求可改为 `YSort`（先保持简单）
- `Blockers`（Node2D）
  - 多个 `StaticBody2D` 子节点
    - 每个包含 `CollisionShape2D`（RectangleShape2D / CapsuleShape2D / ConvexPolygonShape2D）
- `EncounterGrass`（Node2D）
  - 多个 `Area2D` 子节点
    - 每个包含 `CollisionShape2D`（多用 RectangleShape2D 或 ConvexPolygonShape2D）
    - 每个携带 `zone_id`
- `Exits`（Node2D）
  - 多个 `Area2D` 子节点
    - 每个包含 `CollisionShape2D`
    - 每个携带 `exit_id`、可选 `to`（字符串，预留连接信息）
- `DebugSpawn`（Marker2D）
  - debug player 初始生成点（或作为手动放置参考）

## Debug Player 约定

`debug_player.tscn`（根节点：`CharacterBody2D`）

- 8 方向或 4 方向移动（先做 4 方向），基于输入映射：
  - `ui_left`, `ui_right`, `ui_up`, `ui_down`
- 脚本 `debug_player.gd`：
  - 处理移动（speed 可配置）
  - 连接/响应 Area2D 的 `body_entered` / `body_exited`：
    - 进入草丛：打印 `zone_id`
    - 进入出口：打印 `exit_id`

## 元数据（JSON）设计

`data/river-shrine-objects.json`（用于回放/复用摆放，避免完全依赖场景手工编辑）

顶层结构：

- `meta`
  - `map_id`: `"river_shrine"`
  - `tile_size`: `{ "w": 32, "h": 32 }`（约定值；用于布局参考）
  - `tile_dims`: `{ "w": 32, "h": 24 }`
  - `base_image`: `"assets/map/river-shrine-base.png"`
- `props`: 数组
  - `id`: string（如 `"shrine_gate"`, `"stone_lantern_a"`）
  - `image`: string（如 `"assets/props/shrine_gate/prop.png"`）
  - `pos`: `{ "x": number, "y": number }`
  - `z_hint`: `"ground" | "above"`（可选，预留渲染层）
- `blockers`: 数组
  - `id`: string
  - `shape`: `{ "type": "rect" | "poly", ... }`
  - `pos`: `{ "x": number, "y": number }`
- `encounter_grass_zones`: 数组
  - `zone_id`: string
  - `shape`: 同上
  - `pos`: 同上
- `exits`: 数组
  - `exit_id`: string（如 `"north"`, `"west"`）
  - `shape`: 同上
  - `pos`: 同上
  - `to`: string（可选）
- `spawns`
  - `debug_player`: `{ "x": number, "y": number }`

本次实现时：**场景文件为可运行编辑载体**，JSON 作为“可再生/可迁移”描述；两者应保持一致。

## 视觉资产生成规则

- `river-shrine-base.png` 必须是 **foundation-only**：
  - 允许：地面材质、路径、河流/水体、台阶/地形边界、非互动地表细节
  - 禁止烘焙：树、石灯、神社建筑、箱子、门、可碰撞/可交互物体
- props 单独生成透明背景 PNG：
  - 神社小建筑/鸟居/石灯/岩石/树木等
- 每个生成的可见资产必须保存对应 `*.prompt.txt`

## 验证标准

- `river_shrine.tscn` 在任意 Godot 4 项目中打开后：
  - 底图显示正常
  - 具备可编辑的 props、草丛区域、阻挡、出口区域
- `debug_player.tscn` 实例化后：
  - 能移动并被阻挡
  - 进入草丛/出口能在输出打印对应 id
- JSON 可解析、引用文件存在、路径相对 `output/river-shrine-godot/` 成立
- QA 预览图尺寸与底图一致（仅用于人工检查）

