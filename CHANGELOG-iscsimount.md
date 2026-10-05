# iscsiMount 更新日志

> 完整历史见源码仓：https://github.com/DSMFRP1024/iscsiMount

## v1.4.8（versionCode 14）
根治两个老问题 —— 一个是**真 Bug（会导致断线重连）**，一个是**NAS 策略导致的误报**。

### ① 「重连次数 / 写失败」不断上涨 —— 真因是空闲期不读 socket，已加后台保活线程

- **根因**：守护进程只在「有命令在飞」的时候才读那条 TCP。空闲时（fuse 阻塞在 `/dev/fuse` 等请求）
  **根本没人读 socket**，而目标（群晖）每 **30 s** 发一次 NOP-In 保活 ping —— 这些 ping 全部堆在
  内核接收缓冲区里**无人应答**，目标约 **60 s** 后判定发起方失联、**直接关 TCP**。
  下一次写才会发现：先补答堆积的 ping，紧接着读到 EOF ⇒ 「写失败 + 重连」各 +1。
- **铁证**：PDU 环形留档里 8 次失败**全部**呈现同一个形态 ——
  失败前的最大空档是 **40–60 s**，且末尾三个已发 PDU 恒为 `05(Data-Out) / 01(Write Cmd) / 40(NOP-Out 补答)`。
- **修法**：新增**后台保活线程**。**只在没有命令在飞时**（`trylock` 成功）才碰 socket：
  - `poll(fd, POLLIN, 0)` 有数据 ⇒ 读到 NOP-In 就**立刻回 NOP-Out**（这正是空闲期缺的那一环）；
  - 读失败 / `POLLHUP|POLLERR` ⇒ 判定连接已断，**主动重建会话**；
    于是下一次读写根本不会撞上坏连接，「写失败」自然归零。
  - 命令路径与保活线程共用一把锁，保证**绝不会互相抢 PDU**、也不会同时重连。

### ② 「检测不到 LUN / login failed」 —— 是群晖拒绝第二个会话，改为复用当前会话

- **定性（三组对照实验）**：
  1. 本机：`Target-11 / Target-12` 登录成功，`Target-1 / default-target / Target-13` 失败；
  2. **换一台设备（K40，零会话）跑同一个探针：结果完全一样** ⇒ 不是"本机已占用"；
  3. **把探针二进制里的 initiator IQN 改成 `:mi8init99` 再跑：照样失败**
     ⇒ 不是 initiator 名 / ISID 冲突（这条路已证伪，别再试）；
  4. 失败连接停在 `TIME_WAIT`（收到对端 FIN 后我们才关）⇒ 是**目标直接掐掉 TCP**。
- **结论**：群晖**拒绝对"已连接的 LUN"再开一个 iSCSI 会话**，这是 NAS 侧策略，协议层改不动。
- **修法**：
  - 要查的 target 正是本机正在挂载的那个时，**直接复用当前会话已知的信息**（如 `500.0 GiB / exFAT`），
    **不再登录**，也就不会再报 `login failed`；
  - 登录失败时给出**人话提示**：「NAS 拒绝新的 iSCSI 会话…如需重新扫描请先停止挂载」；
  - 前端拿到 `nologin` 标记就**跳过 0–7 盲扫**（那必然每个都被拒、白白等 8 次超时）。

### 顺带修掉
- `live_lun_info()` 的容量换算：mksh 的算术是 **32 位**的，而 500 GiB = 536870912000 = **125 × 2^32**，
  对 `2^32` 取模正好是 **0** ⇒ `$(( ))` 算出 `0.0 GiB`。改用 `awk` 换算。

## v1.4.7（versionCode 13）
修两处**只影响日志与自愈能力**的问题（功能本身正常，但会把人带偏），外加一处"失败没往上抛"。

- **开机那条「合并扩容: bind 成功但 /storage/emulated/0 走的不是 union」是假警报。**
  原因：bind 挂在 **init 的挂载命名空间**上，而复核用的 `df` 跑在模块自己的（service）命名空间，
  开机瞬间**挂载传播还没到位**，于是读到旧的 sdcardfs 就判失败 —— 每次开机必报一条 ERR，
  而实际上 union 是好的（`union_fuse` 的进程起始时间是开机那一次、`/sdcard` 也确实是合并视图）。
  现在复核改为「**init ns 优先、再退当前 ns，并重试 10 次**」，判据从"最后一行是 union"放宽为
  「任意一行是 union」。
- **掉线日志里的 errno 数值不可信。** 上报失败原因原先直接打印裸 `errno`，而 `errno` 会被后续
  任何系统调用覆盖：最典型的是 `send()` 被信号打断（EINTR=4）、`continue` 重试成功后 **errno 残留 4**；
  此后若对端关闭连接（recv 返回 0=EOF），调用点读到的就是那个**陈旧的 4** ——
  于是日志把「对端关闭连接」误报成「被信号打断(EINTR)」，排查方向被整个带偏。
  现在统一改用 `recvn` 维护的 `g_last_errno`，并翻译成人话：
  `对端关闭连接(EOF)` / `读超时(EAGAIN)` / `对端复位(RST)` / `被信号打断(EINTR)` / `errno=N`。
- **EINTR 从「掉线」里摘出来**：EINTR 时读写都**原地重试、不拆会话**。重连会白丢一条 TCP 会话、
  刷日志，短时间内反复 re-login 还可能撞上对端的会话表。
- **失败终于会往上抛了**：`do_start` 的「可见性」步骤（union / SD 可见）失败时现在返回非 0，
  于是 `service.sh` 那套 **6×10s 重试**真正生效 —— 覆盖「开机时 `/storage/emulated` 还没就绪、
  当次 union 被跳过」这类场景。（v1.4.6 里 union 的失败被吞掉，重试从不触发。）

> 真机回归（MI 8 / Android 13）：`df /sdcard` 仍为 **546G**；写新文件落 upper、copy-up 不动 lower、
> 删除后白障生效且 lower 原文件完好、同名重建/目录重建都不复活旧内容 —— 全部复测通过；
> `union-stop` 在未启用时仍是完全 no-op（不会误卸系统存储）。

## v1.4.6（versionCode 12）
- 新增「**合并扩容**」——让**内部存储和外置盘在 `/sdcard` 上合并成同一个视图，容量相加**。
  - 为什么必须自研：本机内核 `# CONFIG_OVERLAY_FS is not set`（overlay 合并做不了）、
    `sm list-disks` 为空（vold 采纳/可合并存储也做不了）⇒ 模块自带一个**零依赖的用户态
    union FUSE**（`bin/union_fuse`，同样是自写的 FUSE 7.26 协议，不依赖 libfuse）。
  - 语义：读 = 外置优先、缺了再回内部；写一律落外置；改内部旧文件先 **copy-up**；
    删 = 在 upper 打 `.wh.<名字>` **白障**（内部原文件不动）；`statfs` 上报**两盘容量之和**。
  - 落地：合并视图先挂到 `/mnt/union`，再 `bind` 到 `/storage/emulated/0`
    （只有挂在 FUSE 路径上 App 才看得见；bind 到 FUSE 底层 `/data/media/0` 会被回 `Cross-device link`）；
    卸载顺序必须是「先拆合并 → 再拆 SD 绑定 → 最后卸主挂载点」（upper 目录就在主挂载点里）。
  - WebUI「挂载位置」新增开关与合并目录名（`union_enabled` / `union_dir`，**默认关**，高侵入；
    与 `sd_visible` 二选一、合并优先）；也可 `ctl.sh union-start|union-stop` 手动排错。
- **真机验收（MI 8 / Android 13）**：`df /sdcard` 由 46G → **546G**（46G 内部 + 500G 外置）；
  合并列表正确去重；写入落外置、copy-up 后内部原文未变；删除后**白障生效**且内部原文件仍在；
  删除后同名重建、目录删后重建**都不会让旧内容复活**；在**真实 App 进程**（launcher / systemui /
  settings）的命名空间里 `df` 与 `ls` 均为合并视图，且从 App ns 写入的文件确实落在外置盘。
- 修掉 4 个在白障/删除语义上的缺陷（都是这会引入的）：
  1) `resolve()` 完全不看白障 ⇒ 删了之后文件**还在**（最要命的一个）；
  2) `readdir` 没让白障压过 upper ⇒ RMDIR 对非空目录失败留下的残留目录会带旧内容复现；
  3) 新建同名文件/目录时不清旧白障 ⇒ 新文件会被自己的白障从列表里挡掉；
  4) `RMDIR` 只打白障不真删 upper 子树 ⇒ 用户重建同名目录时**旧内容复活**。
- 修掉一个**开机时序坑**：FBE 未解锁时 `/storage/emulated` 还是空壳，此时 `bind` 会先落在
  tmpfs、随后被真正的 FUSE 挂载盖住 —— `/proc/mounts` 里有记录、状态显示"已接管"，
  但实际完全没生效。现在 `do_union_start` 会先等 `/storage/emulated` 真正就绪，
  绑完还用 `df` 复核一次 source 是不是 `union`，不通过就报错交给 `service.sh` 重试。
- 加了一道**安全闸门**：只有 `/proc/mounts` 里 source 为 `union` 的挂载才会被卸。
  此前 `do_union_stop` 会无条件 `umount /storage/emulated/0` —— 而它平时是**系统真实的
  sdcardfs 挂载**，一旦在未启用合并时被调用，会让全部 App 掉存储。
- ⚠️ 已知取舍：① 每 App 隔离失效 —— `/sdcard/Android/data` 原本是 sdcardfs 挂载（按 App 隔离），
  被合并视图盖住后**内容不缺**（实测与下层完全一致）但不再隔离；② 删除是"打白障"，
  内部存储的原文件不被真删、只是被遮住；③ 外置盘掉线时 `/sdcard` 只剩内部那份内容（读取正常、写入报错）。

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
