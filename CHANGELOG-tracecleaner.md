# TraceCleaner CHANGELOG

## v1.1.0（2026-10-22）

**新增「其他软件数据中的安装记录」检测与清理（`--appdata`）**

用本机 425 个真实包名当特征，扫描第三方 App 的 `databases` / `shared_prefs` / `files`
（深度 2、<=4MB、只挑 `.db`/`.xml`/`.json`/`.dat`/`.txt`/`.cfg`/`.ini`/`.conf`/`.prefs`），
按文件内命中的包名个数分级：

| 分级 | 判据 | MI 8 实测 |
| --- | --- | --- |
| strong | 同一文件内 >=5 个本机包名 | 13 条 |
| weak | 1-4 个（可能误报，只报告不动） | 144 条 |

清理策略：默认只删 **cache 类**（App 会自行重建）；`shared_prefs` / `databases` /
管理类工具的库必须显式 `--force` 才动。已安装应用的 `/data/data/<pkg>` 仍一律不碰。

MI 8 上清掉的 13 条强证据分布：App Manager（`databases/apps.db`，362 hits）、
资源分享（110）、MIUI 垃圾清理（66/49/28/9，4 个文件）、夸克浏览器（25/16）、
三方应用异常分析（15）、Scene（13）、酷安（12）、扩展屏（9）、小米相册（8）。

**修复**

- busybox `find` 不认 `-size -4M`（报 `invalid number` 直接退出），导致「应用清单缓存」
  一项恒报 0 的假结果。改为 `-size -4096k` 后实测 22 条。
- busybox `find` 要求路径写在表达式之前，`xargs find <paths>` 会把路径追加到表达式
  末尾导致解析失败。改为一次扫描整个 `/data/data` 再按包名过滤。

## v1.0.0（2026-10-22）

首个版本。

- `tc scan`：只读扫描，枚举所有能反推「装过什么」的来源（已安装/已卸载/残留/孤儿/
  历史来源库/应用清单缓存）。
- `tc clean --level 1|2|3`：分级清理，**默认 dry-run**，`--confirm` 才真删，
  删前 `guard_path()` + `cp -a` 备份 + manifest。
  - L1 日志统计（usagestats/netstats/procstats/dropbox/recent_tasks/anr/tombstones）
  - L2 卸载残留 + 孤儿目录
  - L3 历史库整库（商店 `market_2.db`、酷安 `download.db`、安全中心安装来源+月报、
    设置 `screen-time-database`）
  - L4 LSPosed 配置（需 `--xposed`）
- `tc verify`：清理后复核；`tc export`：导出历史库供 PC 端解析。
- 安全设计：无 `service.sh` / `post-fs-data.sh`，不会开机偷偷清理；
  硬保护 `packages.xml`/`packages.list`/`runtime-permissions.xml`/`appops.xml`/`role.xml`、
  已安装包数据目录、模块自身。
