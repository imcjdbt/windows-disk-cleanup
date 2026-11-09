---
name: windows-disk-cleanup
description: 安全清理 Windows 磁盘空间，适用于「C 盘满了」「D 盘整理」「磁盘不够用」「清理系统文件」等请求。强调先报备后动手、按风险分级、迁移优先于删除、每步可验证可回滚；覆盖缓存/安装残留/旧版本/系统文件（pagefile、hiberfil、WinSxS）/云盘缓存/目录联接迁移与提权执行。
metadata:
  trigger: 用户要求清理磁盘、腾出空间、整理某个分区、处理 System 文件占空间
  revision: "2026-11-09"
---

# Windows 磁盘清理

清理磁盘的目标是**拿回空间且不弄坏任何东西**，不是删得越多越好。用户的数据、聊天记录、照片、游戏存档，无论多大都不是"垃圾"。

## 铁律

1. **先报备，后动手。** 任何删除、移动、卸载之前，把清单（路径 + 大小 + 性质 + 风险）报给用户并取得批准。不要因为"这是零风险缓存"就跳过报备——用户要的是知情权，不是你的风险判断。
2. **一批一报。** 不要把多个目录合并成"顺手一起清了"。每批操作单独列清单、单独获批。
3. **描述与事实不符时立即停手。** 如果目录名看起来像安装包，检查后发现有可执行程序，必须回去重新报备，不能按原批准继续。用户的批准是基于你给的信息，信息错了批准就失效。
4. **迁移优先于删除。** 能搬走的不要删。搬走的数据还能用，删掉的就没了。
5. **不碰用户数据。** 聊天记录、相册、下载、备份、游戏库、文档，除非用户明确点名，一律不动。
6. **每次操作后验证。** 重测空闲空间、确认目标确实消失、抽查相关程序仍可用。不要假设命令成功就等于结果正确。

## 阶段一：采集事实

先搞清楚盘子有多大、谁在吃、能不能吃。

```powershell
# 分区与介质
Get-ChildItem D:\ -Force | Select-Object Mode, LastWriteTime, Name
Get-PhysicalDisk | Select-Object FriendlyName, MediaType, BusType, @{n='GB';e={[math]::Round($_.Size/1GB)}}
Get-Partition | Where-Object DriveLetter | Select-Object DriveLetter, DiskNumber, @{n='GB';e={[math]::Round($_.Size/1GB)}}

# 内存（决定 pagefile / hiberfil 体积）
(Get-CimInstance Win32_ComputerSystem).TotalPhysicalMemory / 1GB

# 目标目录定向扫描（关键：只扫目标，不要全盘递归）
Get-ChildItem $path -Recurse -Force -File -ErrorAction SilentlyContinue |
  Measure-Object Length -Sum
```

**扫描纪律**

- 不要 `Get-ChildItem C:\ -Recurse`。WinSxS 和 C:\Users 会各跑两分钟以上。只扫你关心的目录。
- 跳过联接点（junction）。`C:\Users\All Users` 指向 `C:\ProgramData`，全盘扫会把 ProgramData 统计两遍。用 `(Get-Item $p).LinkType` 判断。
- 大范围扫描转后台 job，不要卡住会话。
- 需要按目录排名时，对每个顶层目录单独递归求和，不要一次扫根。

**读不到的尺寸**：`pagefile.sys`、`hiberfil.sys` 这类系统文件，非提权下 `Get-Item` 可能返回 0 或直接失败。不要据此判断"文件不存在"，改用 `Get-ChildItem C:\ -Force -File` 列出的值，或直接用磁盘空闲差值推断。

## 阶段二：分类与定价

每一块都归入下面一类，并给出风险等级。风险等级要写进报备表。

| 类别 | 判据 | 典型体积 | 风险 |
|---|---|---|---|
| 可再生缓存 | Temp、CrashDumps、npm/pip/uv/pnpm 缓存、浏览器 Cache、驱动安装包、AI 索引 | 数 GB | 零 |
| 安装残留 | 更新残留 `.new-{GUID}`、已卸载软件的 MSI 源目录、解压暂存 | 数百 MB ~ 数 GB | 零 |
| 旧版本 | 旧 IDE、旧运行时、旧客户端 | 数百 MB | 极低 |
| 安装介质堆 | 一堆 msi/exe/cab 堆在一个文件夹 | 数 GB | 低（重装需重新下载） |
| 云盘同步缓存 | AnyShare/百度网盘等的 Cache 目录 | 数十 GB | 中（原件不在云端就丢了） |
| 手机备份/截图 | 备份目录、截图同步目录 | 数 GB ~ 数十 GB | 中（是照片） |
| 用户数据 | 聊天记录、下载、相册、游戏 | 数十 ~ 数百 GB | 高（只报告不处置） |
| 系统文件 | pagefile、hiberfil、WinSxS、Windows\Installer | 数十 GB | 提权 + 按官方方式 |

**安装介质堆要特别当心**：目录里有 `.exe` 不代表它是安装包。先看有没有快捷方式指向它、注册表有没有同名程序、别处有没有副本。可能是便携程序本体。

## 阶段三：C 盘系统级

这三项通常是 C 盘最大的机会，都必须提权。

### 虚拟内存（pagefile.sys）

体积约等于物理内存，自动管理时可到 3 倍。**C、D 是同一块物理盘的两个分区时，搬到 D 盘没有任何性能损失**，这只是换账本。

用 **Windows PowerShell 5.1**（`powershell.exe`）执行，PowerShell 7 没有 `Get-WmiObject`：

```powershell
$cs = Get-WmiObject Win32_ComputerSystem -EnableAllPrivileges
$cs.AutomaticManagedPagefile = $false
$cs.Put() | Out-Null
Get-WmiObject Win32_PageFileSetting | ForEach-Object { $_.Delete() | Out-Null }
Set-WmiInstance -Class Win32_PageFileSetting -Arguments @{
  Name = 'D:\pagefile.sys'; InitialSize = 16384; MaximumSize = 32768
}
```

初始大小按内存的 0.7 倍左右给，上限给到 1.3 倍。**重启后** C 盘原文件才真正释放，其间不要读旧文件大小做结论——新文件已经在目标盘按初始大小占位了，所以目标盘会先少掉这部分。

C 盘设成"无分页文件"会让蓝屏转储无法生成。介意的话给 C 盘留 2048 MB。

### 休眠（hiberfil.sys）

约为内存的 40%，同时承载"快速启动"。

```powershell
powercfg /h off            # 完全关闭，省全部
powercfg /h /type reduced  # 只留快速启动，省约一半
```

可逆，`powercfg /h on` 恢复。台式机基本无感，笔记本要提醒失去休眠。

### WinSxS

**绝对不能手删**，手删会导致无法更新甚至无法启动。只能走 DISM：

```powershell
# 保守：只删被取代的旧组件，仍可卸载已装更新
Dism /Online /Cleanup-Image /StartComponentCleanup

# 激进：连历史版本一起删，之后无法回滚补丁
Dism /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

保守版通常回收 2–7 GB，耗时 10–30 分钟。默认先用保守版，`ResetBase` 要单独获得批准。

### 提权执行模式

写一个 PS1 到临时目录，用 UAC 启动，日志落盘后轮询。**脚本内容只用 ASCII**，避免 5.1 读无 BOM 文件时中文乱码。

```powershell
$p = Start-Process powershell.exe -Verb RunAs -PassThru -Wait `
  -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File',$script
$p.ExitCode
```

脚本里每一步都 `Out-File -Append` 到日志，包括 `admin: True/False` 和前后空闲空间。提权进程等待期间用独立的短命令读日志判断进度，不要盲目等。UAC 弹窗需要用户点击，执行前必须提醒。

### 其他常见目标

- `%LOCALAPPDATA%\Temp`：只删 2 天以上的文件，跳过被占用的。
- `%LOCALAPPDATA%\CrashDumps`、`npm-cache`、`pip\Cache`、`uv\cache`、`pnpm\store`。
- `ProgramData\<厂商>\...\UpdateFramework\ota-artifacts`：驱动安装包缓存，动辄数 GB。
- `ProgramData\Comms\MagicAI`：某些 OEM 的 AI 索引缓存。
- 浏览器 `Default\Cache`。
- `C:\WINDOWS\Installer` **不要手删**。要用 `msiexec` 或官方工具。
- 已卸载软件的英文文档包：程序本体可能装在别的盘，C 盘那几 GB 全是文档，卸文档包即可，主程序不受影响。先用 `Get-ItemProperty HKLM:\...\Uninstall\*` 看 `InstallLocation` 确认。

## 阶段四：数据盘

按体积排前列的目录逐个看，但只报告不处置，除非用户点名。

- **云盘缓存**：企业/校园网盘客户端（如 AnyShare）的分块缓存目录、百度网盘缓存。里面的 `.d26` + `.idx` 是分块缓存，不是能直接打开的文件，只能由客户端还原。删除前要确认原件在云端。
- **手机同步**：手机厂商同步目录下的 `Backup`、`Screenshot`，以及各类 `手机图片备份`。
- **游戏库**：steam 等。只能靠卸载不玩的游戏或外接盘，不能"清理"。
- **更新残留**：`<App>.new-{GUID}` 是升级时的新副本，程序正常运行在原目录时这些是纯垃圾。
- **根目录散落文件**：大 zip、旧安装器。没有内容信息就不要建议删除。

## 迁移而非删除：目录联接

把数据目录搬到别的盘，同时保持原路径可用。适用于聊天记录、用户文档这类程序写死路径的数据。

```powershell
# 1. 复制（保留结构，不移动）
robocopy $src $dst /E /R:1 /W:1 /NFL /NDL /NJH /NJS

# 2. 校验：文件数 + 总字节都要一致，不一致就停
#    robocopy 退出码 0–7 都算成功，8 以上才是失败

# 3. 改名源目录
Rename-Item -LiteralPath $src -NewName "$name.bak"

# 4. 建联接
cmd /c mklink /J "$src" "$dst"

# 5. 验证联接可读、抽查内容
# 6. 删除 .bak
```

**必须校验字节数**，不能只比文件数。校验通过再删源，否则回滚。

程序如果用绝对路径写数据，联接能完美骗过它——路径不变，文件在别处。

## 验证清单

每次操作后：

- 重测 `Get-CimInstance Win32_LogicalDisk` 的空闲空间，和操作前对比。
- 确认删除的路径 `Test-Path` 为 False。
- 保留项确认可执行文件仍在（如 `codeblocks.exe`、`gcc.exe`、`MathKernel.exe`）。
- 迁移后确认联接的 `LinkType` 是 `Junction`，且能列出内容。
- 卸载后确认主程序目录和二进制仍在，注册表项已清。

## 输出与交互

报备用表格，列：项目 / 大小 / 性质 / 风险。风险用「零 / 极低 / 低 / 中 / 高」，不要模糊成"应该没问题"。

选择用多选问题，每个选项标注可释放的体积和代价。给出推荐项并说明理由。用户只选部分时，只做选中的，不要"顺手"扩大范围。

最终汇报要包含：本轮释放多少、前后空闲对比、还有哪些没动、下一步建议。

## 已知坑

- `Get-WmiObject` 在 PowerShell 7 里不存在，提权脚本用 `powershell.exe` 跑。
- 提权后 `Get-PSDrive` 反映的是当前进程视角，读空闲空间要以 `Win32_LogicalDisk` 为准。
- 格式化字符串里对字节数直接 `/1MB` 再 `-f` 容易触发类型错误，先算好变量再格式化。
- `Remove-Item` 遇到正被占用的文件会静默失败，删完必须回读确认。
- 卸载 MSI 用 `msiexec /x {GUID} /qn /norestart /L*v log`，1603 通常是缺权限，不要当成"包已损坏"。
- 目录改名成 `.bak` 不会释放空间，只有真的删掉才算。
- 用户说"尽量保守"时，意味着只做零风险和已卸载软件的残留，连旧版本程序都要单独问。
