# iscsiMount 更新日志

> 完整历史见源码仓：https://github.com/DSMFRP1024/iscsiMount

## v1.4.5（versionCode 11）
- 新增「**SD 卡可见**」：在保留原有挂载点（`/mnt/...`）的同时，把 iSCSI 盘**再 bind 一份到
  `/storage/emulated/0/<名字>`**，于是文件管理器（MT管理器等）与各类 App 能在 **`/sdcard/<名字>`**
  下直接看到并**读写**。
  - 关键点：这次 bind **必须做在 init 的挂载命名空间**（`nsenter -t 1 -m -- mount --bind`），
    这样它会沿 shared 传播链自动出现在**所有 App 的命名空间**里（同时落到 `/mnt/user/0`、
    `/mnt/installer/0`、`/mnt/androidwritable/0` 等别名）。
  - 卸载同样跨命名空间清理；并记录「上次用过的 SD 路径」，改过文件夹名后旧路径也会被清掉。
- 更正此前结论：早先说「`/sdcard` 下面挂不上、App 看不到」——那是**把挂载点直接选在 /sdcard 里**
  时 sdcardfs 会跨设备报 `EXDEV`；**正确做法是另做一次 bind**（本次实现），实测有效。
- WebUI 新增「SD 卡可见」开关 + 文件夹名（默认 `iscsidisk`），状态区显示当前 SD 路径与是否已挂上。
- 真机验收（MI 8 / Android 13）：MT管理器**自身命名空间**可见 `/sdcard/iscsidisk`；
  以 MT管理器真实 uid（10286 + 其属组）**读、写均通过**。

## v1.4.4（versionCode 10）
- 新增「检测更新」：WebUI 底部「检查更新」卡片可查看当前/最新版本与更新说明，
  并可**一键下载安装** —— 下载后先校验 sha256，再交给 `magisk --install-module` 安装（重启手机后生效）。
- `module.prop` 接上 Magisk 原生 **updateJson**，Magisk 应用的模块页也会提示更新。
- 更新清单托管在公开仓 [DSMFRP1024/magisk-releases](https://github.com/DSMFRP1024/magisk-releases)
  （模块源码仓仍是私有的）；设备端先走 raw、失败自动回退 jsDelivr。

## v1.4.3（versionCode 9）
- 修好格式化本身：mkfs 命令按设备探测（Android 上 FAT32 用 toybox 的 `newfs_msdos`，
  原先写死的 `mkfs.vfat` 真机上根本不存在；`mkfs.f2fs` 同样缺失），且 mkfs 失败立即报错中止
  （原先不检查返回值，失败也会继续往下挂载，把真因吞掉）。
- 「格式化」由常驻开关改为**一次性按钮**（旧写法一旦忘关，每次开机都会清盘）。
- 「挂载参数」改为按文件系统分组的**预设下拉**（`uid=/gid=/umask=` 那组只对 exfat/vfat 提供，
  ext4/f2fs 不提供）+ ✎ 手动输入。

## v1.4.2（versionCode 8）
- 「挂载位置」可选：WebUI 里挂载点由手输框改为下拉（预设 + ✎ 手动输入），后端加危险路径黑名单，
  并拒绝挂载点与临时目录互相包含。
- 修掉一个真 Bug：改了挂载点再重启时旧挂载点没人卸（`do_umount` 只卸配置里当前那个），
  会变成僵尸挂载并占住 loop，导致 `losetup` 报 `EBUSY`、「改完挂载点就再也挂不上」。

## v1.4.1（versionCode 7）
- 修好「填 IP 就列出 NAS 上全部目标」：发现会话漏协商 `MaxRecvDataSegmentLength`，
  Synology 的 LIO 会因此回空列表。

## v1.4.0（versionCode 6）
- 建立发现会话（按 RFC 7143 发 SendTargets）；新增 `--luns-info` 一次登录取回全部 LUN 的
  容量 / 文件系统 / 型号。WebUI 变为「扫描 → 目标下拉 → LUN 下拉」。

## v1.3.0（versionCode 5）
- 自动识别目标与 LUN：只填 IP + 端口即可。

## v1.2.1（versionCode 4）
- Magisk 模块页「操作」按钮直接打开 WebUI；按需求去掉访问密码（httpd 仅监听 127.0.0.1）。

## v1.2.0（versionCode 3）
- 首个版本：用户态 iSCSI initiator（C/NDK 自研）→ FUSE 假块文件 → losetup → 内核按真实文件系统挂载
  → nsenter 进 zygote 命名空间让 App 可见。
