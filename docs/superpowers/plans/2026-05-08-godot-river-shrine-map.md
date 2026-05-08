# Godot River Shrine Map Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 生成一个可复制进任意 Godot 4 项目的独立 RPG 地图场景包（溪流神社），包含分离 props、遭遇草丛 Area2D、碰撞 StaticBody2D、出口 Area2D、以及 debug player 场景，并同时产出与场景一致的 JSON 元数据与 QA 预览图。

**Architecture:** 使用“foundation-only 底图 + 独立透明 props + Godot 节点分组”的 layered scene 模式。地图的碰撞/区域由 Godot Shape 明确表达，不从像素推导。JSON 作为可再生描述，与 `.tscn` 保持一致。

**Tech Stack:** 生成图像（Cursor 图像生成能力），Python（Pillow/numpy）用于预览合成与校验，Godot 4 `.tscn` 文本场景与 GDScript。

---

## 文件结构（锁定）

**Create（新建）：**
- `docs/superpowers/plans/2026-05-08-godot-river-shrine-map.md`（本文件）
- `output/river-shrine-godot/assets/map/river-shrine-base.png`
- `output/river-shrine-godot/assets/map/river-shrine-base.prompt.txt`
- `output/river-shrine-godot/assets/map/river-shrine-preview.png`
- `output/river-shrine-godot/assets/props/<prop_id>/prop.png`
- `output/river-shrine-godot/assets/props/<prop_id>/prop.prompt.txt`
- `output/river-shrine-godot/godot/river_shrine.tscn`
- `output/river-shrine-godot/godot/debug_player.tscn`
- `output/river-shrine-godot/godot/debug_player.gd`
- `output/river-shrine-godot/data/river-shrine-objects.json`
- `output/river-shrine-godot/tools/compose_preview.py`
- `output/river-shrine-godot/tools/validate_bundle.py`

**Modify（可能修改）：**
- `.gitignore`（若需要忽略 `output/river-shrine-godot/`；当前已忽略整个 `output/`，通常无需改）

## Godot 节点约定（必须满足 spec）

`godot/river_shrine.tscn` 根节点 `Node2D`，包含子节点：
- `Background`（Sprite2D）
- `Props`（Node2D）
- `Blockers`（Node2D，子为 StaticBody2D + CollisionShape2D）
- `EncounterGrass`（Node2D，子为 Area2D + CollisionShape2D，携带 `zone_id`）
- `Exits`（Node2D，子为 Area2D + CollisionShape2D，携带 `exit_id`）
- `DebugSpawn`（Marker2D）

`godot/debug_player.tscn` 根节点 `CharacterBody2D`，脚本 `debug_player.gd`：
- 4 方向移动，使用 `ui_left/right/up/down`
- 在进入草丛/出口时打印对应 id

---

### Task 1: 创建输出目录骨架 + 基础校验脚本

**Files:**
- Create: `output/river-shrine-godot/tools/validate_bundle.py`
- Create: `output/river-shrine-godot/tools/compose_preview.py`

- [ ] **Step 1: 创建目录**

Run:

```bash
mkdir -p "output/river-shrine-godot/assets/map" \
         "output/river-shrine-godot/assets/props" \
         "output/river-shrine-godot/data" \
         "output/river-shrine-godot/godot" \
         "output/river-shrine-godot/tools"
```

Expected: 以上目录存在。

- [ ] **Step 2: 写 `validate_bundle.py`（可独立运行）**

Content:

```python
from __future__ import annotations

import json
import os
from pathlib import Path


ROOT = Path(__file__).resolve().parents[1]


def _require_file(rel: str) -> Path:
    p = (ROOT / rel).resolve()
    if not p.exists() or not p.is_file():
        raise SystemExit(f"Missing file: {p}")
    return p


def _require_dir(rel: str) -> Path:
    p = (ROOT / rel).resolve()
    if not p.exists() or not p.is_dir():
        raise SystemExit(f"Missing dir: {p}")
    return p


def main() -> None:
    _require_dir("assets/map")
    _require_dir("assets/props")
    _require_dir("data")
    _require_dir("godot")

    _require_file("assets/map/river-shrine-base.png")
    _require_file("assets/map/river-shrine-base.prompt.txt")
    _require_file("godot/river_shrine.tscn")
    _require_file("godot/debug_player.tscn")
    _require_file("godot/debug_player.gd")
    data_path = _require_file("data/river-shrine-objects.json")

    with data_path.open("r", encoding="utf-8") as f:
        data = json.load(f)

    base_image = data["meta"]["base_image"]
    _require_file(base_image)

    for prop in data.get("props", []):
        _require_file(prop["image"])

    print("OK: bundle validates")


if __name__ == "__main__":
    main()
```

- [ ] **Step 3: 写 `compose_preview.py`（仅用于 QA）**

Content:

```python
from __future__ import annotations

import json
from pathlib import Path

from PIL import Image


ROOT = Path(__file__).resolve().parents[1]


def main() -> None:
    data_path = ROOT / "data/river-shrine-objects.json"
    with data_path.open("r", encoding="utf-8") as f:
        data = json.load(f)

    base_rel = data["meta"]["base_image"]
    base = Image.open(ROOT / base_rel).convert("RGBA")

    for prop in data.get("props", []):
        img = Image.open(ROOT / prop["image"]).convert("RGBA")
        x = int(prop["pos"]["x"])
        y = int(prop["pos"]["y"])
        base.alpha_composite(img, dest=(x, y))

    out_path = ROOT / "assets/map/river-shrine-preview.png"
    out_path.parent.mkdir(parents=True, exist_ok=True)
    base.save(out_path)
    print(f"Wrote {out_path}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: 运行脚本（预期先失败，因为资源未生成）**

Run:

```bash
python "output/river-shrine-godot/tools/validate_bundle.py"
```

Expected: FAIL，提示缺失 `river-shrine-base.png` 等文件。

---

### Task 2: 生成 foundation-only 底图（溪流神社）

**Files:**
- Create: `output/river-shrine-godot/assets/map/river-shrine-base.png`
- Create: `output/river-shrine-godot/assets/map/river-shrine-base.prompt.txt`

- [ ] **Step 1: 生成底图（foundation-only）**

生成要求（用于图像生成提示词）：
- 俯视/3-4 俯视 RPG 地图风格，**干净清晰、非像素风**（clean HD）
- 画面包含：地面、道路/小径、溪流/浅水、地形边界（岸线/石阶）
- **不得**出现可作为独立物体的：树、石灯、神社建筑、鸟居、箱子、门、路牌等
- 构图包含可用于出口的边缘留白（至少 3 个方向可留通道）
- 输出 PNG

保存同名提示词到：
- `output/river-shrine-godot/assets/map/river-shrine-base.prompt.txt`

- [ ] **Step 2: 人工快速验收底图**

检查：
- 是否无明显可拆分 props
- 是否有足够空间放置 props 与区域

---

### Task 3: 产出 prop 清单并生成透明 props

**Files:**
- Create: `output/river-shrine-godot/assets/props/shrine/prop.png` + `prop.prompt.txt`
- Create: `output/river-shrine-godot/assets/props/torii/prop.png` + `prop.prompt.txt`
- Create: `output/river-shrine-godot/assets/props/stone_lantern/prop.png` + `prop.prompt.txt`
- Create: `output/river-shrine-godot/assets/props/tree_a/prop.png` + `prop.prompt.txt`
- Create: `output/river-shrine-godot/assets/props/rock_a/prop.png` + `prop.prompt.txt`

- [ ] **Step 1: 定义 5 个 props（先满足最小可玩）**

最小集合：
- `shrine`：小型神社建筑或祭坛
- `torii`：鸟居/入口门
- `stone_lantern`：石灯（可重复摆放）
- `tree_a`：树（用于边界阻挡视觉）
- `rock_a`：岩石（用于边界阻挡视觉）

- [ ] **Step 2: 逐个生成透明背景 PNG**

每个 prop 的提示词约束：
- 与底图同风格、同视角、同光照
- **透明背景**（alpha）
- 不要包含文字/UI
- 每个 prop 保存对应 `prop.prompt.txt`

---

### Task 4: 写 JSON 元数据（初版）并生成 QA 预览图

**Files:**
- Create: `output/river-shrine-godot/data/river-shrine-objects.json`
- Create: `output/river-shrine-godot/assets/map/river-shrine-preview.png`

- [ ] **Step 1: 写 `river-shrine-objects.json`（可运行脚本）**

Content（坐标为像素；先用占位坐标，后续可在 Godot 微调后回写）：

```json
{
  "meta": {
    "map_id": "river_shrine",
    "tile_size": { "w": 32, "h": 32 },
    "tile_dims": { "w": 32, "h": 24 },
    "base_image": "assets/map/river-shrine-base.png"
  },
  "props": [
    {
      "id": "torii",
      "image": "assets/props/torii/prop.png",
      "pos": { "x": 480, "y": 600 },
      "z_hint": "above"
    },
    {
      "id": "shrine",
      "image": "assets/props/shrine/prop.png",
      "pos": { "x": 520, "y": 160 },
      "z_hint": "above"
    },
    {
      "id": "stone_lantern",
      "image": "assets/props/stone_lantern/prop.png",
      "pos": { "x": 460, "y": 260 },
      "z_hint": "above"
    },
    {
      "id": "tree_a",
      "image": "assets/props/tree_a/prop.png",
      "pos": { "x": 120, "y": 90 },
      "z_hint": "above"
    },
    {
      "id": "rock_a",
      "image": "assets/props/rock_a/prop.png",
      "pos": { "x": 860, "y": 520 },
      "z_hint": "above"
    }
  ],
  "blockers": [],
  "encounter_grass_zones": [],
  "exits": [],
  "spawns": {
    "debug_player": { "x": 480, "y": 560 }
  }
}
```

- [ ] **Step 2: 合成 QA 预览**

Run:

```bash
python "output/river-shrine-godot/tools/compose_preview.py"
```

Expected: 生成 `assets/map/river-shrine-preview.png`

---

### Task 5: 生成 Godot 场景 `river_shrine.tscn`

**Files:**
- Create: `output/river-shrine-godot/godot/river_shrine.tscn`

- [ ] **Step 1: 写 `.tscn` 骨架（纯文本）**

要求：
- 节点树必须包含：`Background/Props/Blockers/EncounterGrass/Exits/DebugSpawn`
- `Background` 指向底图纹理（路径使用相对输出包的 res:// 语义，便于复制）

提示：因为这是“可复制进任意 Godot 项目”的包，资源在目标项目里放置后路径会以导入位置为准；此处先约定将整个 `output/river-shrine-godot/` 复制到 Godot 项目内的某个目录（如 `res://river-shrine-godot/`）。

- [ ] **Step 2: 根据 JSON 在 `Props` 下实例化 Sprite2D 并设置位置**

先实现最小可用：手工把 5 个 prop 放进去，位置与 JSON 一致。

---

### Task 6: 添加草丛 Area2D、出口 Area2D、阻挡 StaticBody2D

**Files:**
- Modify: `output/river-shrine-godot/godot/river_shrine.tscn`
- Modify: `output/river-shrine-godot/data/river-shrine-objects.json`

- [ ] **Step 1: 在 `EncounterGrass` 下添加 2 个 Area2D**

约定：
- `zone_id`: `"grass_a"`, `"grass_b"`
- shape：RectangleShape2D 起步（后续可替换 poly）

- [ ] **Step 2: 在 `Exits` 下添加 3 个 Area2D**

约定：
- `exit_id`: `"north"`, `"west"`, `"south"`（与底图留白方向一致）

- [ ] **Step 3: 在 `Blockers` 下添加若干 StaticBody2D**

至少覆盖：
- 溪流边缘不可通行区（或假设水体不可走的区域）
- 神社台阶/边界（避免走出地图）

- [ ] **Step 4: 回写 JSON**

把 blockers/grass/exits 的 shape 与 pos 写入 JSON，使 `validate_bundle.py` 可扩展校验（本次至少保证字段存在、可解析、id 不重复）。

---

### Task 7: Debug Player 场景与脚本

**Files:**
- Create: `output/river-shrine-godot/godot/debug_player.tscn`
- Create: `output/river-shrine-godot/godot/debug_player.gd`

- [ ] **Step 1: 写 `debug_player.gd`**

Content（最小可运行）：

```gdscript
extends CharacterBody2D

@export var speed: float = 220.0

func _physics_process(delta: float) -> void:
	var dir := Vector2(
		Input.get_action_strength("ui_right") - Input.get_action_strength("ui_left"),
		Input.get_action_strength("ui_down") - Input.get_action_strength("ui_up")
	)
	if dir.length() > 1.0:
		dir = dir.normalized()

	velocity = dir * speed
	move_and_slide()

func on_enter_grass(zone_id: String) -> void:
	print("enter grass: %s" % zone_id)

func on_enter_exit(exit_id: String) -> void:
	print("enter exit: %s" % exit_id)
```

- [ ] **Step 2: 写 `debug_player.tscn`**

要求：
- 根节点 `CharacterBody2D`，挂载脚本 `debug_player.gd`
- 添加 `CollisionShape2D`（CapsuleShape2D 或 RectangleShape2D）
- 可选添加一个简单 `Sprite2D`（占位可见）

---

### Task 8: 校验与运行指引（文档化）

**Files:**
- Modify: `output/river-shrine-godot/tools/validate_bundle.py`（增强校验规则）

- [ ] **Step 1: validate_bundle.py 增强（字段存在性 & id 唯一性）**

新增校验：
- exits/grass/blockers 中 id 不重复
- `exit_id` / `zone_id` 存在

- [ ] **Step 2: 运行 bundle 校验**

Run:

```bash
python "output/river-shrine-godot/tools/validate_bundle.py"
```

Expected: `OK: bundle validates`

- [ ] **Step 3: 复制进 Godot 项目后的手动验证步骤（写到 plan 或 README）**

在任意 Godot 4 项目中：
- 把 `output/river-shrine-godot/` 复制到项目内，例如 `res://river-shrine-godot/`
- 打开 `res://river-shrine-godot/godot/river_shrine.tscn`
- 实例化 `debug_player.tscn` 到 `DebugSpawn` 位置
- 运行：移动测试碰撞与区域打印

---

## Plan Self-Review

### Spec coverage
- props 分离：Task 3/5
- 草丛 Area2D：Task 6
- 阻挡 StaticBody2D：Task 6
- 出口 Area2D：Task 6
- debug player：Task 7
- JSON 一致性：Task 4/6/8
- QA 预览：Task 4（compose_preview）

### Placeholder scan
- 仅坐标与 shape 允许后续在 Godot 微调，但字段与文件路径无 TBD/TODO。

### Type consistency
- `exit_id`、`zone_id`、`meta.base_image`、`props[].image` 在各处一致。

