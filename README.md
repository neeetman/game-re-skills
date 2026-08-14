# game-re-skills

游戏逆向世界模型数据项目（game-re）的 AI agent skills（内部使用）。

## 安装

```bash
npx skills add <org>/game-re-skills -y -g
```

支持 Claude Code / Cursor / Codex 等（[skills CLI](https://github.com/vercel-labs/skills)）。
更新：重跑同一命令。

## 包含的 skills

| skill | 用途 |
|---|---|
| `use-game-re-dataset` | 算法侧消费 OSS 数据根（schema v8）：文档树入口、数据访问、快照钉住流程、格式陷阱 |

配套建议一并安装官方飞书技能（读项目文档用）：

```bash
npx skills add larksuite/cli -y -g
```
