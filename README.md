# game-re-skills

游戏逆向世界模型数据项目（game-re）的 AI agent skills。

访问参数（bucket、endpoint、凭证）不在本仓库——skill 会引导 agent 先读取飞书文档
（需项目权限）获取参数后再访问数据。

## 安装

```bash
npx skills add neeetman/game-re-skills -y -g
```

支持 Claude Code / Cursor / Codex 等（[skills CLI](https://github.com/vercel-labs/skills)）。
更新：重跑同一命令。

## 包含的 skills

| skill | 用途 |
|---|---|
| `use-game-re-dataset` | 算法侧消费世界模型数据集（schema v8）：文档树入口、数据访问流程、快照钉住、格式陷阱 |

配套建议一并安装官方飞书技能（读项目文档用）：

```bash
npx skills add larksuite/cli -y -g
```
