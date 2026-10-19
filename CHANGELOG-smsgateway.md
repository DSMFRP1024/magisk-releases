# smsGateway 更新日志

> 更新清单：[smsgateway.json](smsgateway.json)　·　源码仓（私有）：DSMFRP1024/smsGateway

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
