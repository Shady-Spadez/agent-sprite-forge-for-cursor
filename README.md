# generate-sprites

在 **Cursor** 中测试 [agent-sprite-forge](https://github.com/0x0funky/agent-sprite-forge) 的工作环境。

原仓库面向 Codex CLI 设计，本仓库把 skills 接到 Cursor 的项目级技能目录 `.cursor/skills/`，并保留原仓库的 Python 后处理脚本与示例素材，整套「Agent 出图 + Python 后处理」流程在 Cursor 里可直接触发。

## 安装步骤

1. 克隆本项目：

```bash
git clone <this-repo-url>
cd generate-sprites
```

2. 下载 / 更新子模块（`agent-sprite-forge`）：

```bash
git submodule update --init --recursive
```

3. 安装 Python 依赖（建议使用虚拟环境隔离）：

```bash
python -m venv .venv
python -m pip install -r agent-sprite-forge/requirements.txt
```

激活虚拟环境：

```bash
# PowerShell
.\.venv\Scripts\Activate.ps1

# Git Bash
source .venv/Scripts/activate
```

4. 在 Cursor 中打开本项目目录（包含 `.cursor/skills/`），必要时重启 Cursor 让技能被重新加载。

## Cursor 中的使用方法

1. **重启 Cursor**（或新开一个 Agent 会话）以重新加载 `.cursor/skills/`。
2. 在 Agent 里用自然语言触发：
   - `Use $generate2dsprite to create a 3x3 idle for an ultimate earth titan.`
   - `Use $generate2dsprite to create a side-view lightning knight attack animation.`
   - `Use $generate2dmap to create a top-down RPG forest shrine map ...`

## 关键差异

- `image_gen`：原 SKILL.md 里写的是 Codex CLI 的内置工具名 **`image_gen`**。Cursor 没有同名工具；如果当前模型/会话具备图像生成能力，Agent 会用自身能力完成出图。**SKILL.md 保持不改**，把 `image_gen` 当作“调用图像生成工具”的语义即可。
- 不出图时：通常是当前会话未绑定图像生成能力；切换到支持图像生成的模型/配置后再试。

## 后处理脚本

两个 skill 各自带 Python 脚本，路径在 `agent-sprite-forge/skills/.../scripts/`。Agent 触发后会自动调用这些脚本；需要手动调试时可用：

```bash
python agent-sprite-forge/skills/generate2dsprite/scripts/generate2dsprite.py --help
python agent-sprite-forge/skills/generate2dmap/scripts/extract_prop_pack.py --help
python agent-sprite-forge/skills/generate2dmap/scripts/compose_layered_preview.py --help
```
