# smsGateway 更新日志

> 更新清单：[smsgateway.json](smsgateway.json)　·　源码仓（私有）：DSMFRP1024/smsGateway

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
