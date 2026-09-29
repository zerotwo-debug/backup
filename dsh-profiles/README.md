# DSH profile 配置备份

备份时间：2026-09-29
备份原因：准备把 Web 版的插件迁移到桌面版（`desktop` profile），迁移前留存可回滚的原始配置。

## 两个 profile 的区别

| profile | 版本 | 载体 | 路径 | 状态 |
|---|---|---|---|---|
| `web` | dsh **0.1.7-rc.2** | `D:\DeepSeekHarness`，浏览器访问 `127.0.0.1:3080` | `C:\Users\Dell\.dsh\profiles\web` | 已装 8 个插件 |
| `desktop` | dsh **0.2.0-rc.2** | `D:\DeepSeekHarness desktop`，Electron 应用 | `C:\Users\Dell\.dsh\profiles\desktop` | 只有内置 `dsh-base` + `dsh-web-app` |

两个版本**共用同一个 `DSH_HOME`**（`C:\Users\Dell\.dsh`），只是 profile 目录不同。因此移植插件 = 把依赖写进 `desktop/package.json` 的 `dsh.profile.bundles` 再装一次。

## web profile 已装的 8 个插件

| 插件 | 版本 | 来源仓库 | 声明的 Node |
|---|---|---|---|
| `billion-context` | 0.1.164 | ranxianglei/billion-context | >=20 |
| `dsh-ai4scholar` | ^0.3.7 | literaf/dsh-ai4scholar | >=22.19.0 |
| `dsh-better-sidebar` | ^0.22.1 | omdsh-dev/DSH-better-sidebar | >=20 |
| `dsh-context` | ^0.58.0 | bowenliang123/dsh-context | ^22.19 ‖ >=24 |
| `dsh-mindmap` | ^0.14.1 | gunhanfei-ai/dsh-mindmap | >=20.11 |
| `dsh-vision-router` | 2.2.5 | ysr666/dsh-vision-router | ^22.19 ‖ >=24 |
| `dsh-whale-widget` | ^0.3.16 | MeteorNOX/DeepSeek-Balance-Whale-Widget | — |
| `dshmarket` | ^1.66.3 | dsh-market/dsh-market | — |

## 已知风险：0.1.7 → 0.2.0 版本鸿沟

插件几乎全部按 dsh `0.1.x` 编写，而桌面版是 `0.2.0-rc.2`：

- `dsh-better-sidebar` 的 peerDeps 是 `@deepseek-ai/dsh-agent: ^0.1.7-rc.1`、`dsh-llm: ^0.1.7-rc.1`、`dsh-session` / `dsh-settings` / `dsh-tools` / `dsh-client-*: ^0.1.7-rc.1`
- `dsh-mindmap` 要求 `dsh-tools ^0.1.0 ~ ^0.1.5`、`dsh-better-sidebar >=0.10.0`
- `dsh-vision-router` 的 peerDeps 只列 `0.1.x` 系列
- `dsh-context` 自带兼容白名单 `compatibility.dshReleases = {"0.1.5-rc.1": "compatible", "0.1.7-rc.2": "compatible"}`，**不含 0.2.0**
- semver 中 `^0.1.7` 等价于 `>=0.1.7 <0.2.0`，**不匹配 `0.2.0`**

插件市场的兼容性缓存 `profiles\web\.dsh-market\discovery-compatibility-v1.json` 中也**没有出现 `0.2.0-rc.2`**。

结论：能否运行无法靠静态分析断定，必须实跑验证。因此采取**小步试水**策略，先移植风险最低的 `dsh-whale-widget`（无 peerDeps、无 engines 约束的纯界面挂件）。

## 回滚方法

把本目录下对应 profile 的文件复制回 `C:\Users\Dell\.dsh\profiles\<profile>\`，覆盖后用桌面版自带 pnpm 重装即可：

```
D:\DeepSeekHarness desktop\resources\runtime\pnpm\bin\pnpm.cjs install
```

桌面版自带运行时：Node 24.21.0 + pnpm 11.7.0 + Python 3.12.14。
