## v1.0.8（versionCode 9）— 2026-10-21

**检查更新从 16s 降到 5s（K40 实测）—— 慢的不是网络，是「取字段」**

| 阶段 | 优化前 | 优化后 |
|---|---|---|
| 网络（拉清单） | 2.5s | 2.5s |
| 取 12 个字段 | **8.2s** | 0.16s |
| 转义 + 输出 | ~3.4s | ~0.3s |

根因：`upd_jget` 每取一个字段都起 `tr | sed | head` 三个 busybox 进程，单次约 0.2s；
`check-update` 要取 12 个字段 ⇒ 36 个进程。改成「一次 `awk` 解析成 `键=值` 行 + shell 内建查表」，
子进程降到 2 个。另外 `jesc` 加了 `case` 短路：不含 `"` 和 `\` 时直接返回，省 7 次 fork。

**两个 mksh 坑（本次实测确认，已写进源码注释）**

- `${x%%\"*}` 与 `${x%%|*}` 同样不可靠：mksh 在参数展开的模式里既不认引号也不认竖线，
  返回的是串首片段而不是截断结果 ⇒ JSON 取值不能用内建展开。
- busybox `sed` 的替换串**不支持反斜杠换行**：写 `s/,"/"\` + 换行 + `"/g` 会直接报
  `sed: unmatched '/'`，且解析结果全空（看着像「清单拉下来了但一个字段都没有」）。
  改用 `awk` 的 `gsub(/,[ \t]*"/, "\n\"", s)`，`\n` 在 awk 里是真换行，可靠。

**回归**：首源改成不可路由地址仍能降级成功；清单换成非 JSON（README.md）仍正确报错，
不会静默显示「已是最新」。

> ★ 升级不覆盖 `config.conf`。

# smsGateway 更新日志

> 更新清单：[smsgateway.json](smsgateway.json)　·　源码仓（私有）：DSMFRP1024/smsGateway

## v1.0.7（versionCode 8）— 2026-10-20

**修「手机端升级地址在国内基本不可用」**

- 根因：清单和安装包的地址只有两个 —— `raw.githubusercontent.com` 与
  `cdn.jsdelivr.net`。国内实测**两个都不通**（前者 25s 超时、后者 8s SSL 超时），
  于是每次点「检查更新」都要先白等 8 秒再碰运气，经常整条失败。
- 现在按序回退 **6 个源**，国内可达的排前面：

  | 顺序 | 源 | 国内实测 | 实时性 |
  |---|---|---|---|
  | 1 | `gh-proxy.com` | ✓ 0.35s | 实时 |
  | 2 | `ghproxy.net` | ✓ 2.0s | 实时 |
  | 3 | `ghfast.top` | ✓ 5.0s | 实时 |
  | 4 | `gcore.jsdelivr.net` | ✓ 0.9s | **有缓存** |
  | 5 | `raw.githubusercontent.com` | ✗ 25s 超时 | 实时 |
  | 6 | `cdn.jsdelivr.net` | ✗ 8s 超时 | **有缓存** |

  raw / cdn 保留在最后 —— 海外或挂代理时它们反而最快。**第一个源命中就返回**，不再串行等待。
- ★ **jsDelivr 必须排在实时镜像后面**：它对 `@main` 分支有缓存，本次发版后实测
  gcore 仍在返回上一版（v1.0.6）。有 versionCode 门控，不会误装旧版，但会让
  「有新版本」显示成「看起来已是最新」。实时镜像在前就不会踩到。
- Magisk 模块页的 `updateJson` 换成 gh-proxy 包装的实时地址：模块页终于能提示新版本；
  清单里的 `zipUrl` 从 `cdn.jsdelivr.net` 换到 gcore（文件名带版本号，不存在缓存旧内容的问题），
  在模块页点安装不会再下不下来。
- 清单内容校验加严（必须同时满足「版本号像版本 + `zipFile` 非空 + `size` 为正」）：
  原来只要文件非空就采信，镜像返回错误页时会**静默显示「已是最新」**；
  更阴的情况是返回的是别的文档（README 里带 JSON 示例），会抠出别人的版本号显示成「有新版本」。
- WebUI「清单来源」显示实际命中的源名。
- 单源连接超时 8s → 6s：不通的源在建连阶段就卡死，等再久只会把 6 个源串成一分钟假死。

> ★ 升级不覆盖 `config.conf`。

## v1.0.6（versionCode 7）— 2026-10-19

**修「手机一出门（只有蜂窝）控制台就再也收不到上报」**

- 根因：`hub_url` 只能填**一个**地址。配的是内网 `http://192.168.2.12:8789`，手机切到移动数据后
  这个地址根本不可达 —— 上报全失败，但手机端既不提示、日志也不明显，看起来就像「模块装了、
  控制台却不认这台设备」。
- 现在 `hub_url` 支持 `|` 分隔**多个地址**，按顺序自动回退：

  ```
  hub_url=http://192.168.2.12:8789|https://sms.example.org
  ```

  - 顺序＝优先级；成功后记住它（`state/hub_active`），之后直接用，不再每轮先撞一次连不上的；
  - 启动时先探测一次（`/api/health`），把 active 定下来；
  - 用户改了配置、旧地址已不在列表里时会自动清掉 active，避免状态里显示作废地址。
- **GET 改为按 HTTP 状态码判成败**：旧版把 GET 结果直接打给 stdout，`/api/cmd/next` 连不上和
  「没有指令」两种情况一模一样，等于把连接故障吞掉了。
- **手机端 WebUI 状态页新增「控制台连接」**：正常 / 连不上（连续 N 次）+ 当前生效地址 + 时间 +
  原因，`/api/status` 也多返回 `hub:{active,ok,ts,fail,err}`。
- 失败日志去噪：只在第 1 次与每 20 次各记一条（15 秒一轮会刷爆），恢复时记「控制台连接已恢复」。
- 单地址 HTTP 超时 25s → **10s**：两个地址各超时会吃掉一整个轮询周期。

> 配套：控制台 `sms-gateway-hub` 1.3.1 修了「最后心跳从不更新」，并允许远程下发 `hub_url` /
> `hub_token`（`cfg-set` 指令有效期放宽到 7 天），这样失联设备下次上线能自己把地址补成双地址。

## v1.0.5（versionCode 6）— 2026-10-17

**新增「检查更新 + 一键安装」**

- 手机端 WebUI 新增「模块升级」卡片：当前版本 / 最新版本 / 清单来源（raw 或 jsDelivr）/
  发布时间 / 包大小 / 更新摘要；有新版时出现「下载并安装 vX.Y.Z」按钮。
- 后端 `ctl.sh check-update` / `ctl.sh update-install`：
  拉公开仓清单 → 比对 `versionCode` → 下载 zip（raw → jsDelivr → 清单里的 zipUrl 三级回退）
  → 校验 `size` 与 `sha256`（半截包直接丢弃）→ `magisk --install-module`。
- **装完落在 `modules_update/`，必须重启手机才生效**，界面与返回文案都写明了这一点。
- `module.prop` 加 `updateJson` —— Magisk 模块页会自己按清单提示并有「更新」按钮。
- 升级安装**不覆盖** `/data/adb/smsgateway/config.conf`（hub 地址、令牌、webhook 全保留）。

**同时补上 v1.0.4 的入口（此前确实没有）**

- 新增根目录 `action.sh`。Magisk 认的模块入口只有这一个（Magisk 26+ 才有「操作」按钮）；
  `webroot/` 只有 KernelSU / APatch 认，Magisk 完全不看 —— 所以早期版本装完在 Magisk 里
  一个入口都没有，功能在却像没做。
- 点「操作」= 确保 httpd 在跑 + `am start` 直接拉起浏览器打开 WebUI。

## v1.0.4（versionCode 5）— 2026-10-17

- 新增 `action.sh`：Magisk 模块卡片上的「操作」按钮，一键打开手机端 WebUI。
- 刷机包 / 热部署四处同步：`build_module.py`、`customize.sh`、`工具/deploy.py`、`工具/sg_push.sh`。

## v1.0.3（versionCode 4）— 2026-10-17

**修「历史短信一条都推不上来，而且完全不报错」**

- `SmsMain.pickSmsDb()`：按 `sms` 表 `max(_id)` 最大者选库（同分再比 mtime），
  不再取「第一个存在的文件」。K40（Android 16 / HyperOS）上
  `/data/user_de/0/.../mmssms.db` 是空库、`/data/data/.../mmssms.db` 才有 166 条，
  旧逻辑挑到空库 ⇒ `maxid` 恒 0 ⇒ 静默不推。
- `openSmsDb` / `cmdInfo` / `cmdDel` 统一走新逻辑；测试注入器 `InjectSms.java` 同步修正
  （它犯的是同一个错，会让端到端测试「假通过」）。

## v1.0.2（versionCode 3）

- 新增手机端 WebUI：busybox httpd + CGI，只监听 `127.0.0.1:8790`。
  运行状态 / 双卡 / 最近短信 / 日志、发短信（选卡）、改配置、立即同步、重启守护进程。

## v1.0.1 / v1.0.0

- 首个可用版本：app_process dex 读 `mmssms.db`，实时推送到飞牛控制台或自定义 webhook，
  接受 `send-sms` / `sync-sms` / `sims-now` / `sms-del` / `cfg-set` / `ping` 指令，支持双卡指定卡槽。
