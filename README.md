# dsh-whale-soul

大肥鱼的**人设 + 机制层**。装上它，你的 DSH agent 就有了「大肥鱼」这个人，以及它的一整套干活方式——但没有它的记忆。

## 这是什么

`dsh-whale-core` 是引擎（记忆注入、多 agent 派活、心跳调度、兴趣调度），**这个仓库是引擎的「人格」**：

| 文件 | 作用 |
|---|---|
| `cordis.patch.yml` | soul-md 人设卡：大鱼 + 6 条子 agent（学徒/考官/账房/掌柜/人味/记性/管家）+ sessions 映射 |
| `记忆系统/编队.md` | agent 三级协议（主 agent / 临时父 agent / 干活子 agent）+ 委派四件套 |
| `multi-agent/链条.json` | 派活时的链条定义（验收/学习新内容/查个资料/改插件代码/定学什么） |
| `兴趣系统/种子.txt` | 兴趣起点：19 个可挖方向名，新鱼上路的「方向表」 |
| `记忆系统/设计.md`、`兴趣系统/设计.md`、`multi-agent/子agent构筑方法.md` | 设计文档与学习方法论 |

## 它**不**带什么

- ❌ 底色.md —— 「我是大肥鱼 / 他养我 / 正在做记忆系统 / 最近纠结什么」，这些是**个人状态**，不是机制
- ❌ 记忆.db / 图.db / 状态.json / 任务流.jsonl / 好奇心.md / 口味.md —— **记忆和知识储备**，新机器是空的
- ❌ 任何凭据（xhs 登录态 / 微信 token / API key）

装完它，你得到的是一个「**会本事的空鱼**」：脾气、说话方式、编队、学习链路都在，但记忆清零，重新长大。

## 安装

这个仓库是**纯配置包**，没有 JS 代码。它通过 `cordis.patch.yml` 给 `dsh-soul-md` 插件注入人设卡。

### 1. 作为 bundle 安装（人设卡生效）

在你的 profile 的 `package.json` 里加：

```json
{
  "dependencies": {
    "dsh-soul-md": "^0.10.0",
    "dsh-whale-soul": "https://github.com/YongyiLi0603/dsh-whale-soul/tarball/main"
  },
  "dsh": {
    "profile": {
      "bundles": ["dsh-soul-md", "dsh-whale-soul"]
    }
  }
}
```

然后 `pnpm install`。这样 `cordis.patch.yml` 里的人设卡就生效了。

### 2. 机制文件复制到 DSH_WHALE_HOME（编队/链条/种子）

bundle 只负责人设卡。机制文件要放到 `DSH_WHALE_HOME` 下（默认 `/home/ubuntu/.dsh`），core 插件会去那儿读：

```bash
DSH_WHALE_HOME=~/.dsh
cp 记忆系统/编队.md "$DSH_WHALE_HOME/记忆系统/编队.md"
cp 记忆系统/设计.md "$DSH_WHALE_HOME/记忆系统/设计.md"
cp 兴趣系统/种子.txt "$DSH_WHALE_HOME/兴趣系统/种子.txt"
cp 兴趣系统/设计.md "$DSH_WHALE_HOME/兴趣系统/设计.md"
cp multi-agent/链条.json "$DSH_WHALE_HOME/multi-agent/链条.json"
cp multi-agent/子agent构筑方法.md "$DSH_WHALE_HOME/multi-agent/子agent构筑方法.md"
```

（完整流程见 `dsh-whale-core` 的一键安装脚本，它会自动做这些。）

## 已知限制

- 人设卡里的工具路径写死为 `/home/ubuntu/.dsh/...`（默认安装路径）。如果你装到别处，需要同时改 `DSH_WHALE_HOME` 环境变量 + 人设卡里的路径。
- 种子可视化依赖 `dsh-whale-core` 的 `/whale-seeds` 路由 + `dsh-whale-tools` 的 `种子看板.html`，三个仓库装齐才完整。
