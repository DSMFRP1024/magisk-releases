# iscsiMount 更新日志

> 完整历史见源码仓：https://github.com/DSMFRP1024/iscsiMount

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
