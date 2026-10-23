# TraceCleaner CHANGELOG

## v1.4.2（2026-10-23）

**KernelSU 桥自检 + 补齐 v1.4.1 漏掉的第二层差异**

v1.4.1 只统一了「返回值形态」，但真机上还有第二层：**回调的实参形态**。
部分 KSU 分支的回调是 `cb({errno,stdout})` 或 `cb({code,data})` 这种**单参数对象**，
而页面按 `cb(errno,stdout,stderr)` 三个位置参数解构 —— 于是 `errno` 位被塞进一个对象、
`stdout` 位是 `undefined`，归一化出空字符串，界面依旧显示「命令无输出」。
也就是说：报错文案变了，功能还是不通。

本版改动：

- **回调实参按个数分派**：1 个参数时依次判断 对象 / 字符串 / 数字；≥2 个时按位置参数。
- **启动时探测调用签名**：用只读的 `tc json status` 依次试
  `exec(cmd)` / `exec(cmd,{},cb)` / `exec(cmd,cb)` / `exec(cmd,null,cb)`，
  2.5s 无结果就换下一种，第一个真正出数据的签名被缓存，后续全部命令都用它。
  探测只用 `status`，**幂等无副作用**，不会重复触发扫描或清理。
- **桥全局名放宽**：`ksu` / `$ksu` / `kernelsu` / `apatch` / `mmrl`，取第一个带 `exec` 的。
- **页面顶部显示可用签名**（`桥 m2`），一眼看出走的是哪条路。
- **连不上时显示诊断卡片**：桥对象是否存在、`exec.length`、四种签名的探测结果、最后一次原始返回。
  截图即可定位，不必再靠猜。
- 探测失败后不再每 6 秒重试（旧版会反复失败刷屏）。
- 回归测试从 5 种形态扩到 8 种（新增 `cbobj` / `cbdata` / `cbstr`），并加了 `noexec` 负例：
  桥完全不返回时必须给出诊断而不是白屏或无限等待。

## v1.4.1（2026-10-23）

**修复 KernelSU 上 WebUI 报「命令无输出：TypeError: Cannot use 'in' operator to search for 'stdout'」**

真机环境：Android 16 / SDK 36 / KernelSU。该版本 `ksu.exec(cmd)` **同步返回 stdout 字符串**，
而页面按「返回 `{errno, stdout, stderr}` 对象」去解析，`"stdout" in <字符串>` 直接抛 TypeError，
被 catch 成 stderr，界面显示成「命令无输出」。

KernelSU 各版本的 `exec` 返回形态实测有四种，本版统一兼容：

| 形态 | `exec(cmd)` 返回 |
|---|---|
| 老版同步 | stdout 字符串 |
| 新版异步 | `Promise<{errno,stdout,stderr}>` |
| 回调式 | 调用 `exec(cmd, cb)`，返回 `undefined` |
| 混合 | 同时返回 Promise 且调回调 |

实现要点：

- **只执行一次命令**（带回调调用 + 同时检查返回值，先到先用）。多调一次会让 `scan-start` 跑两遍。
- 返回值统一归一化：字符串 → 当 stdout；对象 → 兼容 `stdout`/`output`/`result` 与 `errno`/`code`。
- 错误提示带上退出码与可复现命令，不再是一句无信息量的「命令无输出」。
- `result` 解析失败时提示「返回内容被截断」，引导改用浏览器版（KSU 的输出通道有长度上限）。
- 打包时由 `webroot/index.html` 生成 `web/ksu.html`（此前是安装脚本 cp，容易残留旧副本）。

回归测试：本地用假 `ksu` 桥模拟上述四种形态（命令真实转发到设备执行），
四种全部出数且 `exec` 调用次数一致；另加真实数据断言（425 已装 / 1 已卸载 / 1 残留 / 13 强证据）。

## v1.4.0（2026-10-23）

**补上「没有操作按钮」和「KernelSU 打不开 WebUI」**

之前只给了 `tc web start` + `adb forward` 那条路，管理器里既没有按钮，KernelSU 也认不到页面。本版把入口补齐：

- **`action.sh`（模块页「操作」按钮）**
  管理器只在模块根目录存在 `action.sh` 时显示按钮。点一下 = 启动 httpd + 用**手机浏览器**打开
  `http://127.0.0.1:6789/?token=…`（页面读到 `?token=` 自动登录，地址栏随后抹掉参数）。
  等价的命令行入口：`tc web open`。
- **`webroot/index.html`（KernelSU / APatch / MMRL 内置 WebUI）**
  管理器只认 `webroot/index.html`，且页面必须通过 `ksu.exec` 调 root 命令——httpd/CGI 那套在 WebView 里跑不起来。
  新页面为手机排版，四个标签：概览（统计卡+状态）/ 扫描（六类明细+搜索）/ 清理（L1-L3 + appdata 档位 +
  预演/真删，需输入 CONFIRM）/ 更新（检查+安装+日志）。**不依赖 httpd、不联网、不用 adb。**
- **`tc json <op>` 纯 JSON 操作层**（`lib/json.sh`）
  `status / scan-start / result / clean-start / job / update-start / check-update`，一行 JSON，无 HTTP 头。
  管理器 WebUI 用 `ksu.exec` 调它；浏览器 WebUI 走新增的 `cgi-bin/json` 转调它——两边同一套逻辑，不会跑偏。
- 同一份页面也装到 `web/ksu.html`，手机浏览器可直接用（`?token=` 免登录）。

**实测（MI 8 / Magisk，v1.4.0）**

- `action.sh` → 浏览器被拉起，页面显示 v1.4.0 / 端口 6789，免令牌登录
- 管理器版页面在手机浏览器里渲染正常：425 已装 / 1 已卸载 / 1 残留孤儿 / 13 强证据痕迹文件
- 清理预演 L2（后台+轮询）：`rc=0`、319 项、合计 318 项约 56 MB，报告落到 `/sdcard/TraceCleaner/`
- 扫描（后台+轮询）：`running=true` → 完成出结果；`check-update` JSON 正常
- `cgi-bin/json` 无令牌返回 401 语义、有令牌正常

**修复**

- `cgi-bin/json` 取不到第一个查询参数：busybox httpd 的 `QUERY_STRING` **不带前导 `?`**，
  按 `.*[?&]op=` 匹配永远落空 ⇒ 改为按 `&` 切分后逐项匹配。

## v1.3.3（2026-10-23）

**修复回退源把版本"倒退"的问题（真机踩到）**

- 现象：`check-update` 显示 v1.3.2 有更新，紧接着点安装却返回「已是最新（v1.3.0）」。
  原因是 raw 偶发不可达时回退到 jsDelivr，而 jsDelivr 对 `@main` 缓存很久，回的是旧清单。
- 改为**多源都问一遍、取 `versionCode` 最大的那份**；一旦某个源给出的版本已高于当前版本，
  立即停止（不必等更慢的源）。
- 镜像按 MI 8 实测重排：`raw` → `ghfast.top`（raw 的国内代理，同样回源）→ `cdn.jsdelivr.net`（最后兜底）。
  实测不可达的 `gcore.jsdelivr.net` / `raw.gitmirror.com` / `cdn.statically.io` 已剔除。

## v1.3.2（2026-10-23）

- 清单请求带 `?ts=<当前秒级时间戳>`：raw 与 jsDelivr 的边缘缓存会让**刚发布**的版本"看不见"
  （真机实测：v1.3.1 已推送，设备仍拿到 v1.3.0 的清单并装成 v1.3.0）。加时间戳强制回源。
- 安装包 URL 不加时间戳（含版本号，内容固定，可放心吃缓存）。

## v1.3.1（2026-10-23）

**修复真机暴露的更新链路问题（v1.3.0 首发即踩）**

- 同名 zip 重发过两次（55650 → 56640 字节）导致 jsDelivr 边缘节点清单与包不同步，
  安装前的 `size` 校验必然失败。**发版纪律：内容一变就升 `versionCode` 并换文件名，
  绝不在同一版本号下重发不同内容的包。**
- `tc update install` 改为**逐源校验**：某个源给的包大小或 sha256 对不上就自动换下一个源，
  而不是直接判失败；全部不一致才报错，并提示"等 5~10 分钟让 CDN 缓存同步"。
- 镜像顺序按国内真机实测调整：`raw` 与 `gcore.jsdelivr.net` 在 MI 8 上直连不通，
  只有 `cdn.jsdelivr.net` 稳定可达 ⇒ 顺序 raw → cdn.jsdelivr → gcore，且清单超时压到 20s。
- `updateJson` / `zipUrl` 一并改用 `cdn.jsdelivr.net`（Magisk 应用侧同样走可达域名）。

## v1.3.0（2026-10-23）

**新增在线升级（检测更新 + 一键安装）**

- `module.prop` 配 `updateJson`，Magisk 应用的模块页可直接点更新
- 命令行：`tc update`（检查）· `tc update install`（下载并安装）· `tc update json`（机读）
- WebUI 新增「更新」标签：检查更新 / 更新说明与包大小 sha256 / 下载并安装 / 安装输出；
  有新版时导航上的「更新」带红点，进页面静默探一次
- 实现要点：
  - 清单走公开分发仓 `DSMFRP1024/magisk-releases`（设备端无 GitHub 凭据，私有源码仓拉不到）；
    **raw 优先、jsDelivr 回退**，来源回传界面
  - 下载器探测式：系统 `curl` 优先（校验证书），Magisk busybox `wget` 兜底（不校验证书）
  - 无 jq：`tr -d '\r\n'` 压行后 sed 取值（**不能连空格删**，否则吃掉 notes 里的空格）
  - 校验：先比 `size`，再比 `sha256`，一致才交给 `magisk --install-module`
  - 装在 `modules_update/`，**必须重启才生效**；界面与命令行都显式提示
  - 版本判定用 `versionCode` 门控（`[ lat -gt cur ] 2>/dev/null`），清单滞后的 CDN 只会晚提示、不会误装旧版

## v1.2.0（2026-10-23）

**新增设备端 WebUI（浏览器操作与查看）**

`tc web start` 在手机上拉起 busybox httpd（默认 `127.0.0.1:6789`），
电脑 `adb forward tcp:6789 tcp:6789` 后浏览器打开 `http://127.0.0.1:6789` 即可：

- **概览**：统计卡片 + 服务/任务状态
- **扫描结果**：七类明细（已安装/已卸载/残留孤儿/历史来源/应用清单缓存/应用数据快照），可搜索、可排序
- **清理**：选 L1/L2/L3 + appdata 档位 + LSPosed/force/apks；先「预演」看输出，再手输 `CONFIRM` 真删
- **日志**：clean.log / scan.log / 最近清理输出，10 秒自动刷新

实现要点：

- 零外部依赖：busybox httpd + CGI（shell），页面是单文件原生 JS，设备不用联网
- 扫描**异步**（后台 setsid 跑，轮询自动刷新）；清理**同步**（前端 loading）；scan/clean 任务互斥
- 安全：默认只听回环；`--lan` 时全部敏感接口校验 32 位随机令牌（`tc web status` 查看）；
  确认清理必须输入 `CONFIRM`；令牌可用 `tc web token` 更换
- `service.sh` 开机自启（`touch web/disabled` 可关）
- MI 8 实测：status / scan（后台约 2 分钟）/ result（121 KB JSON）/ clean dry-run（306 项 56 MB）/ verify / log 全部通过

**修复**

- busybox grep 的 BRE 不认 `\|`  alternation，导致 WebUI 清理统计 `items` 恒为 0；改为 `grep -E`。
- `tc web` 端口文件路径拼接多了一层 `/web`（`.../web/web/port`）导致写入失败。
- httpd 默认自行 daemon 化会与 start-stop-daemon 的 pid 文件脱同步，启动命令加 `-f` 由 ssd 守护。

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
