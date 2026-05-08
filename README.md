# generate-sprites

在 **Cursor** 中测试 [agent-sprite-forge](https://github.com/0x0funky/agent-sprite-forge) 的工作环境。

原仓库面向 Codex CLI 设计，本仓库把 skills 接到 Cursor 的项目级技能目录 `.cursor/skills/`，并保留原仓库的 Python 后处理脚本与示例素材，整套「Agent 出图 + Python 后处理」流程在 Cursor 里可直接触发。

## 目录结构

```text
generate-sprites/
├── .cursor/
│   └── skills/
│       ├── generate2dsprite/      # 已从原仓库复制，Cursor 加载用
│       └── generate2dmap/
├── agent-sprite-forge/            # 原仓库克隆，保留 README/scripts/src
│   ├── skills/
│   ├── src/                       # 原仓库示例图与 GIF
│   ├── requirements.txt
│   └── README.md
└── assets/                        # 建议把生成结果放这里（自行创建）
```

## 已完成的安装步骤

1. `git clone https://github.com/0x0funky/agent-sprite-forge.git agent-sprite-forge`
2. `python -m pip install -r agent-sprite-forge/requirements.txt`（Pillow、numpy）
3. 把 `agent-sprite-forge/skills/{generate2dsprite,generate2dmap}` 复制到 `.cursor/skills/`
4. 冒烟校验：两个 skill 的 Python 脚本都能正常 import

## 在 Cursor 中如何触发

1. **重启 Cursor**（或新开一个 Agent 会话），让 `.cursor/skills/` 重新加载。
2. 在 Agent 里用自然语言下达请求即可，触发词与原 README 一致：
   - `Use $generate2dsprite to create a 3x3 idle for an ultimate earth titan.`
   - `Use $generate2dsprite to create a side-view lightning knight attack animation.`
   - `Use $generate2dmap to create a top-down RPG forest shrine map ...`

更多示例见 `agent-sprite-forge/README.md` 的 *Suggested Prompts* 段。

## 关键差异：`image_gen` 在 Cursor 中的对应

原 SKILL.md 全程要求调用 Codex 内置工具 **`image_gen`** 出图。Cursor 没有同名工具，但当前会话/模型若具备图像生成能力，Agent 会用自身的图像生成出 raw 图。**SKILL.md 没有改动**，把 `image_gen` 当作"调用图像生成工具"的语义即可。

如果发现 Cursor 实际不出图，常见原因与对策：

| 现象 | 可能原因 | 处理 |
|---|---|---|
| Agent 说自己不能生成图 | 当前模型/会话没绑定图像生成 | 换一个具备图像生成能力的模型，或接入 MCP 图像服务（OpenAI / Gemini / 本地 SD 等）后再改 SKILL.md 把 `image_gen` 替换成对应工具名 |
| 出图风格偏离要求 | 提示词中 `#FF00FF` 纯色背景规则未被严格执行 | 在请求里再次强调；原 prompt-rules.md 已在 references 中 |
| 后处理报错 | numpy/Pillow 版本不匹配 | 见下方依赖说明 |

## 后处理脚本

两个 skill 各自带 Python 脚本，原始路径仍在 `agent-sprite-forge/skills/.../scripts/`，复制后的副本在 `.cursor/skills/.../scripts/`。Agent 触发后会自己调用这些脚本，无需手动跑。如需手动调试：

```bash
python agent-sprite-forge/skills/generate2dsprite/scripts/generate2dsprite.py --help
python agent-sprite-forge/skills/generate2dmap/scripts/extract_prop_pack.py --help
python agent-sprite-forge/skills/generate2dmap/scripts/compose_layered_preview.py --help
```

## 依赖说明 / 已知影响

- `requirements.txt` 要求 `numpy>=1.26`，**安装时把全局环境的 numpy 升到了 2.2.6**。如果你别的项目要求 `numpy<2`（matplotlib、pandas、scikit-learn、scipy 等会报告版本不兼容），建议为本仓库使用虚拟环境隔离：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r agent-sprite-forge\requirements.txt
```

激活该 venv 后再启动 Cursor，让 Agent 调脚本时使用本环境。

## 参考

- 原仓库：<https://github.com/0x0funky/agent-sprite-forge>
- 原 README（含示例 prompts、生成示例）：`agent-sprite-forge/README.md`
- skill 行为规范：`.cursor/skills/generate2dsprite/SKILL.md`、`.cursor/skills/generate2dmap/SKILL.md`
- prompt 规则：`.cursor/skills/generate2dsprite/references/prompt-rules.md`
