# DSH Profile 备份（插件移植前后留存）

备份时间：2026-09-30
用途：把 Web 版的 8 个社区插件移植到 DSH 桌面版（0.2.0-rc.2）之前与之后的配置留存，便于回滚。

## 两个 profile 的关系

- `DSH_HOME` = `C:\Users\Dell\.dsh`，Web 版与桌面版**共用**同一个 HOME，但用**各自独立的 profile 目录**。
- `profiles\web\`：Web 版 dsh 0.1.7-rc.2（`npx @deepseek-ai/dsh web` / `start-dsh.bat`）。
- `profiles\desktop\`：桌面版 dsh 0.2.0-rc.2（`D:\DeepSeekHarness desktop`）。
- 两边的 `cordis.yml` 都是空数组 `[]` 并注明「Edit cordis.patch.yml, not this file」；实际配置在 `cordis.patch.yml`。
- `package.json` 的 `dsh.profile.bundles` 是**加载层序**，`dependencies` 是安装清单。**两者都要有，插件才会加载。**

## 移植后的插件清单（桌面版，全部已进 bundles）

| 插件 | 桌面版版本 | Web 版版本 | 说明 |
|---|---|---|---|
| dsh-whale-widget | 0.3.16 | 0.3.16 | 余额小鲸鱼挂件，无 peerDeps，首个试水 |
| dsh-context | 0.58.0 | 0.58.0 | 依赖用 `>=` 写法，恰好通过兼容闸门 |
| dsh-ai4scholar | 0.3.7 | 0.3.7 | 同上 |
| billion-context | 0.1.173 | 0.1.164 | 无 peerDeps；桌面版被 pnpm 解析到 0.1.173 |
| dsh-mindmap | 0.15.0 | 0.14.1 | 全系不支持 0.2.x，用 `allow-version` 显式豁免强装 |
| dsh-better-sidebar | 0.24.1 | 0.22.1 | 仅 0.24.1 把 15 个 peer 改成 `^0.2.0-rc.1`，专为 0.2.0 发布 |
| dsh-vision-router | 2.2.8 | 2.2.5 | 2.2.6+ 才在 peer 里追加 `>=0.2.0-rc.1 <0.3.0-0` |
| dshmarket | 1.66.5 | 1.66.3 | 1.66.4+ 才在 peer 里追加 `^0.2.0-rc.1` |

**桌面版与 Web 版的插件版本必然分叉**：better-sidebar 0.24.1 只认 `^0.2.0-rc.1`，在 Web 版 0.1.7 上反而装不了。这是正常的，不要试图统一版本。

## 关键坑（踩过的，务必记住）

### 1. dsh 的兼容闸门只看 peerDependencies，不看 dsh.compatibility
`dsh plugin add` 会检查 `peerDependencies` 能否被运行时版本满足：
- `>=0.1.x` 形式**放行**（`>=` 包含 0.2.0）
- `^0.1.x` 形式**拒绝**（`^0.1.7` = `>=0.1.7 <0.2.0`）

被拒时的报错会直接给出豁免命令：
```
dsh plugin --profile desktop allow-version <pkg>@<ver> --dsh-version 0.2.0-rc.2 --accept-risk
```
豁免结果记录在 `profiles\desktop\compatibility.json`。

注意 `dsh-context@0.58.0` 自己的 `dsh.compatibility.dshReleases` 只写了 `0.1.5-rc.1` / `0.1.7-rc.2`，**不含 0.2.0**，却因 `>=` 写法被放行 —— 属于侥幸通过，升级时需重测。

### 2. pnpm 走 registry.npmjs.org 会导致「假死」
最耗时的坑。pnpm 不读 `~/.npmrc` 的国内镜像，会去 `registry.npmjs.org` 拉 `@napi-rs/canvas` / `@img/sharp-*` 的**全体平台变体**（含 linux-arm64-musl 等在 Windows 上根本用不到的），每个下载失败后退避重试 10 秒 → 1 分钟，十几个变体叠加就是几十分钟看起来完全卡死（`pnpm.log` 恒为 0 B，`package.json.lock` 留成僵尸锁）。

**解法**：在 profile 目录写 `.npmrc`，并在调用前设环境变量：
```powershell
Set-Content -Path "$env:USERPROFILE\.dsh\profiles\desktop\.npmrc" -Value "registry=https://registry.npmmirror.com/" -Encoding ascii
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
用 node 改写，Node 的 `writeFileSync(..., 'utf8')` 不带 BOM；或在 PowerShell 里用 `[System.IO.File]::WriteAllText($p, $s, (New-Object System.Text.UTF8Encoding($false)))`。

### 4. 沙箱限制
- 所有 `dsh plugin` 操作、git commit 都必须提权（沙箱无法写 `C:\Users\Dell\.dsh`，也无法让 git 清理自己创建的 `index.lock`）。
- 桌面 profile 被 Electron 应用独占：`dsh --profile desktop --dump-config` 会报 `profile "desktop" is managed exclusively by the Electron application`，只能靠重启应用来验证配置是否生效。
- 每次 `dsh` CLI 调用都会输出 crashpad 噪声 `TransactNamedPipe: 有更多数据可用 (0xEA)`，**无害**。

### 5. bundle 已存在时不会再挂载
若某个包已经在 `dependencies` 里、但不在 `bundles` 里，直接 `dsh plugin add` 会因为「已存在」而**跳过挂载步骤**。必须先把它从 `dependencies` 摘掉（用 node 无 BOM 改写），再重新 `add`。

## 回滚方法

### 只回滚桌面版插件（推荐）
```powershell
# 1. 关闭桌面应用
Get-Process -Name 'DeepSeek Harness' | Stop-Process -Force
# 2. 把本目录 desktop\ 下的文件复制回 profile
Copy-Item .\desktop\* "$env:USERPROFILE\.dsh\profiles\desktop\" -Force
# 3. 删掉后来装上的 node_modules 与锁文件
Remove-Item "$env:USERPROFILE\.dsh\profiles\desktop\node_modules" -Recurse -Force
Remove-Item "$env:USERPROFILE\.dsh\profiles\desktop\pnpm-lock.yaml","$env:USERPROFILE\.dsh\profiles\desktop\package.json.lock" -Force
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

### Web 版不受影响
本次移植**从未修改** `profiles\web\`，那里的 8 个插件（0.1.7 版本线）保持原样。

## 桌面版运行时信息（供参考）

- 安装目录 `D:\DeepSeekHarness desktop`，版本 0.2.0-rc.2，Electron 44，内置 Node 24.21.0 / pnpm 11.7.0 / Python 3.12.14。
- 官方更新源 `https://download.deepseek.com/dsh-desk/feeds/win-x64/`（channel: nightly），签名 `CN=Hangzhou DeepSeek Artificial Intelligence Co., Ltd.`。
- CLI 入口：`D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd`
  ```powershell
  # 常用
  & 'D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop add <pkg>@<ver>
  & 'D:\DeepSeekHarness desktop\resources\runtime\cli\bin\dsh.cmd' plugin --profile desktop remove <pkg>
  ```