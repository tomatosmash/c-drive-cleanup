---
name: c-drive-cleanup
description: 安全清理 Windows C 盘空间。当用户说「C 盘满了 / 清理 C 盘 / 释放磁盘空间 / 磁盘空间不足」时使用。先只读探查定位占用大头,优先清用户级零风险缓存,大目录(WSL / VM 沙箱)迁 D 盘不删,系统级清理写成脚本让用户管理员运行,每个不可逆操作先确认。
---

# C 盘清理(Windows)

严格按「先探查再动手、优先零风险项、每个不可逆操作先确认」执行。

## 一、先探查(只读,不动任何文件)

1. 查 C 盘总量/已用/剩余/使用率:
   ```powershell
   powershell.exe -NoProfile -Command 'Get-PSDrive C | Select-Object @{N="TotalGB";E={[math]::Round(($_.Used+$_.Free)/1GB,1)}},@{N="FreeGB";E={[math]::Round($_.Free/1GB,1)}},@{N="UsedPct";E={[math]::Round($_.Used/($_.Used+$_.Free)*100,1)}} | Format-List'
   ```
2. 扫描 C 盘顶层目录占用,定位大头(通常 Users、Windows、Program Files、ProgramData):
   ```powershell
   powershell.exe -NoProfile -Command 'Get-ChildItem C:\ -Directory -Force -ErrorAction SilentlyContinue | ForEach-Object { $s=(Get-ChildItem $_.FullName -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object Length -Sum -ErrorAction SilentlyContinue).Sum; [PSCustomObject]@{Name=$_.Name;SizeGB=[math]::Round($s/1GB,2)} } | Sort-Object SizeGB -Descending | Format-Table -AutoSize'
   ```
3. 深入扫描 `AppData\Local`、`AppData\Roaming`、`ProgramData` 的子目录,按大小排序。
4. 检查系统级可清理项:回收站、`C:\Windows\SoftwareDistribution\Download`、`C:\$WinREAgent`、NVIDIA 驱动缓存、`C:\ProgramData\Package Cache`、`%TEMP%`。

## 二、清理分类与优先级

### 1. 用户级缓存 —— 直接 rm,零风险(最优先)

- `%TEMP%`:删内容、保留目录,占用中的文件跳过。
- 开发缓存:`go\pkg\mod`、`.gradle\caches`、`.m2\repository`、pip cache、npm cache。
- 纯系统缓存:**NVIDIA `DXCache`**(着色器缓存,约 4G)、Steam `htmlcache`、浏览器 Cache——名字明确含 cache/临时性质的目录,单条 `rm -rf`(即使带多个路径)通常能通过权限分类器。
- 更新器残留:目录名常含 `updater`/`update`。
- `Downloads` 里的软件安装包(Setup.exe,如 OllamaSetup):已装好的话可删,但先列出让用户确认,个人文件(如 .rar)绝不删。

⚠️ 删 `%TEMP%` 会顺带删掉 Claude Code 自己的临时输出文件,导致该命令的结果读不到(报 output file ENOENT)。属正常副作用,删完单独重查 `Get-PSDrive C` 即可,不要当失败重试。

### 2. 应用数据类缓存 —— 软件会重建,但 auto 模式会拦

WPS `kingsoft\wps\addons`、飞书 `LarkShell\iron\users` / `sdk_storage` / `aha\users`、WeGame `qbcore*`、微信 `xwechat\xplugin`、直播姬 `bililive\User Data\Cache` 等,直接 `rm -rf` 会被 auto 权限分类器判为「不可逆破坏」拦下(即使一次只删一个目录、或改用 `目录/*` 删内容也一样)。

**正确做法**:把这些目录写成一个 `.ps1` 脚本让用户自己跑(普通终端即可,不需要管理员),不要在会话里反复尝试删除或切权限(切权限、改 settings.json 都会被判「自我修改」拦)。脚本用 `Remove-Item` 删内容、`-ErrorAction SilentlyContinue` 跳过占用。

### 3. 大目录 —— 能迁不删

- **WSL 虚拟磁盘**(`AppData\Local\wsl\{guid}\ext4.vhdx`,可达数 G):用 `wsl --manage <发行版名> --move D:\wsl` 迁到 D 盘,不丢数据。先 `wsl --list --verbose` 确认发行版名与状态(Stopped 最佳)。
- Claude Desktop VM 沙箱 `Claude-3p\vm_bundles`(约 11G):**先确认存在再动**,本机可能根本没有。存在且 VM 闲置时用 `mklink /J` 迁 D 盘,不要直接删。
- 其他占空间的大应用数据:优先迁 D 盘,而非删除。

### 4. 系统级缓存 —— 需管理员,写成脚本让用户手动跑

`ProgramData\NVIDIA`、`ProgramData\Package Cache`、`$WinREAgent`、`SoftwareDistribution\Download` 等普通用户删不动,写成 `.ps1` 让用户 `Win+X → 终端(管理员)` 运行。

脚本要点(避免编码坑):**脚本内提示一律用英文**,否则 PowerShell 5.1 按 GBK 读 UTF-8 无 BOM 的 .ps1 会中文乱码。核心命令:
- `Dism.exe /Online /Cleanup-Image /StartComponentCleanup`(收缩 WinSxS,唯一合法手段)
- 停 `wuauserv`/`bits` → 删 `SoftwareDistribution\Download\*` → 重启服务
- `Clear-RecycleBin -Force`
- `Package Cache` 默认注释掉不删(删了 MSI 无法修复/卸载)

## 三、安全边界(绝不直接删)

- `Windows\WinSxS`(约 23G):只能用 DISM StartComponentCleanup。
- `Windows\Installer`(约 6G):不能删,删了已装程序无法修复/卸载。
- `Program Files` / `Program Files (x86)`:程序本身,要动只能迁移大程序到 D 盘(风险高,需用户指定具体程序)。
- 陌生目录、游戏存档(如 `.minecraft`)、`Downloads` 个人文件、`AppData` 下不明用途的应用数据:删前必须向用户确认。

## 四、权限与执行要点(本机验证)

- 本会话可能不是管理员,`ProgramData`/`Windows` 下删除会 Permission denied。
- **不要反复尝试自动提权**(`Start-Process -Verb RunAs` 会被安全策略拦)。
- 单条 `rm -rf`(多个缓存路径)通常能通过分类器;批量 PowerShell 删除脚本容易被拒。
- **auto 权限模式的三道锁**:删应用数据(判不可逆破坏)、切权限模式、改 settings.json(判自我修改)全会被拦。遇到删除被拦不要死磕,直接写脚本给用户跑,或让用户手动 `/permissions` 切到 Accept Edits。

## 五、目标

优先清零风险缓存,每步清理后报告释放空间,目标把使用率降到 80% 以下。D 盘空间充足(100G+)时大文件优先迁 D 盘。200G 的盘要降到 80% 需释放约 33G,光靠缓存不够,要提前告诉用户可能还需迁移大程序。
