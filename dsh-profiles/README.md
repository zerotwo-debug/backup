# DSH Profile 备份

- 首次备份：2026-09-30（移植桌面版插件前后留存）
- **本次刷新：2026-09-30 —— `DSH_HOME` 已从 `C:\Users\Dell\.dsh` 迁移到 `D:\dsh-home`**

用途：记录 Web 版与桌面版两个 profile 的完整配置（插件清单、加载层序、兼容性豁免、镜像设置），供回滚与换机复原。

---

## 一、两个 profile 的关系

- `DSH_HOME` = **`D:\dsh-home`**（2026-09-30 从 `C:\Users\Dell\.dsh` 迁移而来，**旧目录已删除**）
- Web 版与桌面版**共用同一个 HOME**，但用**各自独立的 profile 目录**：
  - `profiles\web\` —— Web 版 dsh **0.1.7-rc.2**（`start-dsh.bat` / `node_modules\.bin\dsh web`）
  - `profiles\desktop\` —— 桌面版 dsh **0.2.0-rc.2**（`D:\DeepSeekHarness desktop\DeepSeek Harness.exe`）
- 两边的 `cordis.yml` 都是空数组 `[]` 并注明「Edit cordis.patch.yml, not this file」；**实际配置在 `cordis.patch.yml`**。
- `package.json` 里 `dsh.profile.bundles` 是**加载层序**，`dependencies` 是**安装清单**。**两者都要有，插件才会加载**（只进 `dependencies` 不进 `bundles` = 装了不生效）。

---

## 二、当前插件清单（2026-09-30 快照）

| 插件 | 桌面版 (0.2.0-rc.2) | Web 版 (0.1.7-rc.2) | 说明 |
|---|---|---|---|
| dsh-whale-widget | 0.3.17 | ^0.3.16 | 余额小鲸鱼挂件；无 peerDeps、无 engines，最干净 |
| dsh-context | 0.60.0 | ^0.58.0 | 依赖用 `>=` 写法，恰好通过兼容闸门 |
| dsh-ai4scholar | 0.3.7 | ^0.3.7 | 同上；需在「设置 → 插件 → AI4Scholar」配 API key |
| billion-context | 0.1.173 | 0.1.173 | 无 peerDeps（压缩/上下文管理，即 bili proxy） |
| dsh-mindmap | 0.15.0 | ^0.14.1 | 全系不支持 0.2.x，用 `allow-version` 显式豁免强装 |
| dsh-better-sidebar | 0.24.1 | ^0.22.1 | 仅 0.24.1 把 15 个 peer 改成 `^0.2.0-rc.1`，专为 0.2.0 发布 |
| dsh-vision-router | 2.2.8 | 2.2.5 | 2.2.6+ 才在 peer 里追加 `>=0.2.0-rc.1 <0.3.0-0` |
| dshmarket | 1.66.5 | ^1.66.3 | 1.66.4+ 才在 peer 里追加 `^0.2.0-rc.1`（插件市场） |

**桌面版与 Web 版的插件版本必然分叉**：`dsh-better-sidebar@0.24.1` 只认 `^0.2.0-rc.1`，在 Web 版 0.1.7 上反而装不了。这是正常的，**不要试图统一版本**。

### bundles 加载层序

桌面版（10 项）：
```
@deepseek-ai/dsh-base, @deepseek-ai/dsh-web-app,
dsh-whale-widget, dsh-context, dsh-ai4scholar, billion-context,
dsh-mindmap, dsh-better-sidebar, dshmarket, dsh-vision-router
```

Web 版（11 项）：
```
@deepseek-ai/dsh-base, @deepseek-ai/dsh-web-app,
dshmarket, dsh-mindmap, dsh-context, dsh-ai4scholar, dsh-vision-router,
dsh-better-sidebar, billion-context, @deepseek-ai/dsh-experimental-agent-team-profile,
dsh-whale-widget
```
Web 版另有 `patchReload: "live"`。

### 与上一版快照（提交 `8a98a31`）的差异

| 项 | 变化 |
|---|---|
| `DSH_HOME` | `C:\Users\Dell\.dsh` → **`D:\dsh-home`**（旧目录已删） |
| dsh-context | 0.58.0 → **0.60.0**（插件市场自动升级） |
| dsh-whale-widget | 0.3.16 → **0.3.17**（插件市场自动升级） |
| billion-context（Web） | 0.1.164 → **0.1.173** |
| `.npmrc` | 新增 `store-dir=D:\.pnpm-store\v11` |
| `profiles\web\.npmrc` | **新增**（此前不存在） |
| `cordis.patch.yml`（桌面） | 多了 `vision-router` / `better-sidebar` / `agent-default-model` 三个条目 |
| `pnpm-workspace.yaml`（桌面） | `minimumReleaseAgeExclude` 从 4 项变 5 项，全部重写为精确版本 |

---

## 三、关键坑（都实际踩过，务必记住）

### 1. dsh 的兼容闸门只看 peerDependencies，不看 dsh.compatibility
`dsh plugin add` 检查 `peerDependencies` 能否被运行时版本满足：
- `>=0.1.x` 形式**放行**（`>=` 包含 0.2.0）
- `^0.1.x` 形式**拒绝**（`^0.1.7` = `>=0.1.7 <0.2.0`）

被拒时报错会直接给出豁免命令：
```
dsh plugin --profile desktop allow-version <pkg>@<ver> --dsh-version 0.2.0-rc.2 --accept-risk
```
豁免结果记录在 `profiles\desktop\compatibility.json`（本仓库已备份）。

**残留风险**：`dsh-context@0.60.0` 的 `dsh.compatibility.dshReleases` 只写了 `0.1.5-rc.1` / `0.1.7-rc.2`，**不含 0.2.0**，只因 `>=` 写法被放行 —— 属侥幸通过，升级时需重测。（上游 `bowenliang123/dsh-context` 已有提交 `74117d8` 补上 `0.2.0-rc.2`，但尚未发 npm 版本。）

### 2. pnpm 走 registry.npmjs.org 会导致「假死」
最耗时的坑。pnpm 不读 `~/.npmrc` 的国内镜像，会去 `registry.npmjs.org` 拉 `@napi-rs/canvas` / `@img/sharp-*` 的**全体平台变体**（含 Windows 上根本用不到的 linux-arm64-musl），每个下载失败后退避重试 10 秒 → 1 分钟，十几个变体叠加就是几十分钟的完全卡死（`pnpm.log` 恒为 0 B，`package.json.lock` 留成僵尸锁）。

**解法**：在 profile 目录写 `.npmrc`，并在调用前设环境变量：
```powershell
Set-Content -Path "$env:DSH_HOME\profiles\desktop\.npmrc" -Value "registry=https://registry.npmmirror.com/" -Encoding ascii
$env:npm_config_registry = 'https://registry.npmmirror.com/'
$env:DSH_DESKTOP_NPM_REGISTRY = 'https://registry.npmmirror.com/'
```
加上之后单次安装从「无限挂起」变成 1～5 秒。

### 3. 绝对不要用 PowerShell 的 Set-Content -Encoding UTF8 改写 package.json
本机默认 shell 是 **Windows PowerShell 5.1**（不是 pwsh 7），它的 `-Encoding UTF8` 会写入 **UTF-8 BOM**，而 dsh 的 `readProfileManifest` 用 `JSON.parse` 直接读文本，遇到 BOM 立刻崩：
```
SyntaxError: Unexpected token '锘?, "锘縶
```
（`锘縶` 就是 `EF BB BF` + `{` 的 GBK 乱码。）

用 node 改写（`fs.writeFileSync(p, s, 'utf8')` 不带 BOM），或在 PowerShell 里用：
```powershell
[System.IO.File]::WriteAllText($p, $s, (New-Object System.Text.UTF8Encoding($false)))
```
**同理，`.ps1` 脚本反而必须带 BOM**，否则 PS 5.1 按 ANSI 读中文会解析失败。

### 4. 沙箱 / 权限限制
- 所有 `dsh plugin` 操作必须提权（沙箱无法写 `$DSH_HOME`；桌面版 CLI 还会同步读写 `$DSH_HOME\profiles\desktop\package.json.lock`）。
- git commit 也需提权：沙箱内 git 清理不掉自己创建的 `.git\index.lock`。
- 桌面 profile 被 Electron 应用独占：`dsh --profile desktop --dump-config` 会报 `error: profile "desktop" is managed exclusively by the Electron application`，**验证配置只能靠重启应用**。
- 每次 `dsh` CLI 调用都会输出 crashpad 噪声 `CreateFile: 拒绝访问 (0x5)` / `TransactNamedPipe: 有更多数据可用 (0xEA)`，**无害**。

### 5. bundle 已存在时不会再挂载
若某个包已经在 `dependencies` 里、但不在 `bundles` 里，直接 `dsh plugin add` 会因为「已存在」而**跳过挂载步骤**（pnpm 报 `Packages: -N`，没有 `dependencies: + ...` 行）。必须先把它从 `dependencies` 摘掉（用 node 无 BOM 改写），再重新 `add`，这样才会同时进 `dependencies` 与 `bundles`。

---

## 四、DSH_HOME 搬迁记录（2026-09-30）

从 `C:\Users\Dell\.dsh` 迁到 `D:\dsh-home`，目的是减轻 C 盘压力。

### 机制
- 主目录解析规则（`@deepseek-ai/dsh-home-paths`）：**显式配置 > `$DSH_HOME` 环境变量 > `~/.dsh`**；空或纯空白视为未设置。桌面版的 Electron 主进程在无参调用 `resolveDshHome()`，即读 `process.env.DSH_HOME`。
- **pnpm store 是「按盘」走的**：C: 上用 `%LOCALAPPDATA%\pnpm\store\v11`，换到别的盘时 pnpm 会在该盘重建 store。**硬链接不能跨盘**，所以主目录与 store 必须同盘，否则 pnpm 退化成整份复制。
- 两个 profile 的 `.npmrc` 现均写有 `store-dir=D:\.pnpm-store\v11`。

### 易被忽略的坑：482 个 junction
`profiles\node_modules\`（注意是 `profiles` 的直接子目录，**不是** `desktop\` / `web\` 各自的 `node_modules`）下有 **482 个 junction**，全部指向 `D:\DeepSeekHarness\node_modules\...`，用于让两个 profile 共享 Web 版安装的 `@deepseek-ai/dsh-*` 内核包。

搬迁时：
- **robocopy 必须加 `/XJ`**，否则会跟着 junction 把目标内容重复复制一遍（实测会多复制出整份 `.dsh`，1 GB 垃圾）。
- `/XJ` 之后 robocopy 仍会为「只装着 junction 的目录」建出**空壳目录**，所以重建链接时不能只 `Test-Path` 就跳过 —— 必须先判断是不是重解析点，是空壳目录就先删掉再建链接，否则链接全变成空目录、插件加载不了。
- 建链接用 `New-Item -ItemType Junction`（**不需要管理员**）；目标不存在的「悬空链接」无法用 junction 重建（`mklink /J` 会报 `The system cannot find the file specified.`），只能用需要管理员的目录符号链接。当时的 482 个里有 **74 个本来就是悬空的**，未重建不改变原有行为。

---

## 五、回滚方法

### 只回滚桌面版插件（推荐）
```powershell
# 1. 关闭桌面应用
Get-Process -Name 'DeepSeek Harness' | Stop-Process -Force
# 2. 把本目录 desktop\ 下的文件复制回 profile
Copy-Item .\desktop\* "$env:DSH_HOME\profiles\desktop\" -Force
# 3. 删掉后来装上的 node_modules 与锁文件
Remove-Item "$env:DSH_HOME\profiles\desktop\node_modules" -Recurse -Force
Remove-Item "$env:DSH_HOME\profiles\desktop\pnpm-lock.yaml","$env:DSH_HOME\profiles\desktop\package.json.lock" -Force
# 4. 重启桌面应用
Start-Process 'D:\DeepSeekHarness desktop\DeepSeek Harness.exe'
```
（第 3 步后首次启动会重新安装 `dependencies` 里的包，记得同时设好上面的镜像环境变量。）

### 彻底回到移植前（只剩空 profile）
删除 `profiles\desktop\package.json` 里的全部 `dependencies`，并把 `dsh.profile.bundles` 恢复为：
```json
["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app"]
```
然后删除 `node_modules\`、`pnpm-lock.yaml`、`compatibility.json`、`.npmrc`。

### 回到 C 盘
删掉用户级环境变量 `DSH_HOME` 即可（若已删除旧目录，需先把 `D:\dsh-home` 整个复制回 `C:\Users\Dell\.dsh`）：
```powershell
[Environment]::SetEnvironmentVariable('DSH_HOME', $null, 'User')
```

---

## 六、桌面版运行时信息（供参考）

- 安装目录 `D:\DeepSeekHarness desktop`，版本 0.2.0-rc.2，Electron 44，内置 Node 24.21.0 / pnpm 11.7.0 / Python 3.12.14。
- 官方更新源 `https://download.deepseek.com/dsh-desk/feeds/win-x64/`（channel: nightly），签名 `CN=Hangzhou DeepSeek Artificial Intelligence Co., Ltd.`。
- CLI 入口：`D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd`
  ```powershell
  # 常用
  & 'D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop add <pkg>@<ver>
  & 'D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop remove <pkg>
  # 兼容性豁免
  & 'D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop allow-version <pkg>@<ver> --dsh-version 0.2.0-rc.2 --accept-risk
  ```
- 本仓库里的 `pnpm-lock.yaml` **未备份**（98 KB，且随安装频繁变动），需要时按 `package.json` 重装即可。
