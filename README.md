# windows-disk-cleanup

一个给编码 Agent 用的 Windows 磁盘清理 skill。核心不是"删得多"，而是**先报备、后动手，每一步都可验证、可回滚**。

AI 助手清理磁盘最容易犯的错，是看到"缓存"两个字就直接 `Remove-Item`。这份 skill 把"必须先把清单交给用户批准"写成第一条铁律，并给出一套经过实际使用的归因表和验证流程。

## 它解决什么

- **不擅自删任何东西。** 删除、移动、卸载前先出清单（路径 / 大小 / 性质 / 风险），获批才执行；一批一报，不"顺手一起清"。
- **发现事实不符立即停手。** 目录名看着像安装包、里面有可执行程序时，回去重新报备，不按原批准继续。
- **迁移优先于删除。** 聊天记录、用户文档这类写死路径的数据，搬到别的盘再用目录联接（junction）保持原路径可用。
- **系统文件只走官方通道。** pagefile 用 WMI 迁移，hiberfil 用 `powercfg`，WinSxS 只走 DISM，绝不手删。
- **每步验证。** 重测空闲空间、确认目标消失、抽查相关程序仍可用。

## 安装

**DeepSeek Harness / 兼容 `skills/` 目录的 Agent：**

```
%USERPROFILE%\.dsh\skills\windows-disk-cleanup\SKILL.md
```

**Claude Code：**

```
%USERPROFILE%\.claude\skills\windows-disk-cleanup\SKILL.md
```

把 `SKILL.md` 放到上述目录即可，Agent 会按 frontmatter 里的 `description` 自动匹配"C 盘满了""整理 D 盘""清理磁盘"这类请求。

## 内容结构

| 阶段 | 做什么 |
|---|---|
| 铁律 | 六条不可协商的约束：先报备后动手、一批一报、信息不符立即停手、迁移优于删除、不碰用户数据、每步验证 |
| 一 · 采集事实 | 分区/介质/内存；定向扫描纪律（不全盘递归、跳过 junction、大扫描转后台） |
| 二 · 分类定价 | 八类归因：可再生缓存 / 安装残留 / 旧版本 / 安装介质堆 / 云盘缓存 / 手机同步 / 用户数据 / 系统文件，逐类给风险等级 |
| 三 · C 盘系统级 | pagefile 迁移、hiberfil 休眠、WinSxS 的 DISM 两级、提权脚本模式、常见缓存目标 |
| 四 · 数据盘 | 云盘缓存、手机备份、游戏库、更新残留、根目录散落文件 |
| 迁移技法 | robocopy → 字节校验 → 改名 → `mklink /J` → 验证 → 删源 |
| 验证清单 | 每次操作后的回读校验 |
| 已知坑 | 实际踩到的九个坑 |

## 报备格式

skill 要求每次动手前给出这样一张表：

| 项目 | 大小 | 性质 | 风险 |
|---|---|---|---|
| `%LOCALAPPDATA%\Temp`（>2 天） | 3.5 GB | 可再生缓存 | 零 |
| `...\NVIDIA app\UpdateFramework\ota-artifacts` | 8.3 GB | 驱动安装包缓存 | 零 |
| 旧版本 IDE 目录 | 1.1 GB | 已废弃版本 | 极低 |
| 云端同步缓存 | 44 GB | 可能是唯一副本 | 中 |

风险用「零 / 极低 / 低 / 中 / 高」，不给"应该没问题"这种模糊结论。用户只选部分时，只做选中的。

## 几个真会踩到的坑

- `Get-WmiObject` 在 PowerShell 7 里不存在，涉及虚拟内存的脚本要用 `powershell.exe`（5.1）跑。
- `pagefile.sys` / `hiberfil.sys` 在非提权下读不到大小，别据此判断文件不存在。
- `C:\Users\All Users` 是 `ProgramData` 的联接，全盘扫会重复统计。
- `Remove-Item` 遇到被占用的文件会静默失败，删完必须回读确认。
- 目录改名成 `.bak` 不会释放空间，真的删掉才算。
- `msiexec` 返回 1603 通常是缺权限，不是安装包损坏。
- robocopy 退出码 0–7 都算成功，8 以上才是失败。
- 卸载"英文文档包"这类组件，可能释放的比主程序本身还多。

## 适用范围

Windows 10 / 11。Windows PowerShell 5.1 与 PowerShell 7 都可，个别命令已注明只能用 5.1。涉及系统级操作时需要管理员权限（UAC）。

## License

MIT
