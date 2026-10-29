# iscsiMount 更新日志

> 完整历史见源码仓：https://github.com/DSMFRP1024/iscsiMount

## v1.5.3（versionCode 19）
**修看门狗「误杀」：把慢 IO 当成死锁的判定彻底去掉。**

### 用户侧现象（2026-10-29 真机）

- 海阔视界下载列表：`海阔视界_V8.82_C5590_db53-64位` 显示 **40.15MB / 0B /「文件创建失败」**
- 下载目录里留下 0 字节的 `.apk.download` 与空 `.apk.temp/` 目录，下载完全下不去
- logcat：`java.io.IOException: close failed: ENOTCONN (Transport endpoint is not connected)`
- `service.log` 显示开机 26 分钟内看门狗**连判两次卡死**并 abort 全部挂载：

```
19:43:30 看门狗: union 卡死（waiting=7 连续 1 轮不变，state=D 连续 3 轮）→ 自动救活
19:46:23 看门狗: union 卡死（waiting=3 连续 0 轮不变，state=D 连续 3 轮）→ 自动救活
```

### 根因

判据过时，不是 union 坏了。`union_fuse_stuck()` 只读 `/proc/<pid>/stat` 的**主线程**状态：

```sh
_st=$(awk '{print $3}' "/proc/$_p/stat")
[ "$_st" = "D" ] && return 0
```

这是**单线程时代**的写法（那时主线程 D = 全体卡死）。v1.5.1 起 union_fuse 是 4 个 worker，
主线程只是其中之一 —— 它写 upper（upper 在 iSCSI/NAS 上）时会连续 D 几十秒到上百秒，
而另外 3 个 worker 照样在服务。

铁证：两次命中的 waiting 分别是「**连续 1 轮**」「**连续 0 轮**」不变 —— 请求一直在流转，
根本不是死锁。abort 一执行，所有 App 已打开的 fd 立刻变 `EIO/ENOTCONN`，
正在下载的文件当场写失败；同时 exfat 每次都被判为「非正常卸载」。

### 四条改动

| # | 内容 |
|---|---|
| ① | 新增 `union_thread_stats()` 遍历 `/proc/<pid>/task/*` 统计全部 worker 的 D 数 |
| ② | `union_fuse_stuck()` 改为「**≥线程数-1 个线程同时在 D**」才算无可用 worker（容许 1 个例外） |
| ③ | 命中判据后、执行 abort 前加一道活性探测 `timeout 3 ls /mnt/union`，能返回就判定为慢 IO 并跳过 |
| ④ | 阈值 3 轮（90s）→ **6 轮（180s）**，真死锁照样救得回 |

`do_union_check` 的 D 轮次也只统计「几乎无可用线程」的情况，1~2 个线程在等 NAS 一律不计。

### 验证

| 项目 | 结果 |
|---|---|
| `sh -n` | OK |
| `union_thread_stats` 真实环境 | `tot=4 d=0`，`union_fuse_stuck rc=1`（判未卡死） |
| PID 不存在 / 指向非 union_fuse | 均 `rc=1`（防 PID 复用） |
| PID 指向真实 union_fuse | `tot=4 d=0`，`rc=1` |
| 判据表驱动 6 场景 | 含 10-29 误杀现场（waiting=7、1/4 线程 D）全部不再命中；4/4 D 或 waiting 不变 6 轮仍命中 |
| 热替换后跑一轮 `union-check` | `rc=0`，`union.stuck=「0 0 0」` |
| 模拟下载器完整流程 | 写 10 MiB → `sync` → `rename` 收尾，1 秒完成，0 残留 |

`ctl.sh` 由 `service.sh` 每轮以新进程调用（每轮都重新读磁盘上的文件），
所以本次**热替换即生效、零中断**，不需要重启 union，也不需要重启设备。

## v1.5.2（versionCode 18）
**加「自我递归」启动自检：把「运行中永久卡死」变成「启动即报错」。**

### 事故经过（2026-10-18 真机）

`/sdcard` 整体卡死：`ls` 25 秒不返回，而 `/mnt/iscsidisk` 0.4 秒正常。

| 指标 | 实测 |
|---|---|
| `/sys/fs/fuse/connections/213/waiting` | 9，三轮采样一动不动 |
| union_fuse 的 4 个 worker | 全在 `newfstatat`，其中 1 个 `D`、3 个 `S` |
| `/proc/<pid>/cmdline` | `--upper /storage/emulated/0`（**正是 union 自己的 bind 目标**），且**没有 `--workers`** |
| 父进程 | `sh ctl.sh union-guard`，而现行 `ctl.sh` 的 case 表**已无该子命令** |

根因不是 v1.5.1 的多 worker 改造，而是一个 **10-05 开机时起的遗留看门狗进程**：
设备 uptime 13 天，它跑的是旧 inode 里的旧代码（旧参数、旧 upper 算法）；
`ctl.sh` 升级到 v1.5.1 之后它依然每 20–30 秒判定「卡死」并用**同一错误参数**重启，
upper 指向自己的挂载点 ⇒ 读 upper 的请求递归回本连接 ⇒ 所有 worker 等自己回复自己。

### 三道防线

1. **`union_fuse` 启动自检**：挂载前、挂载后各查一次 `/proc/self/mountinfo`
   （纯内核文本，**不触发任何 FUSE 请求**，所以不会在检查时把自己卡住）。
   `upper` 若等于或落在任何 `union` 挂载点之内，**拒绝启动 / 立即卸载退出**并打印命中的挂载点。
   挂载后那一次还能抓住「`--upper` 恰好在 `--mount` 之下」这种挂载前查不出来的情形。
2. **`ctl.sh` 的重叠检查扩到全部 union 挂载点**：原来只比对 `$union_mp` / `$union_target` 两个固定路径，
   但 bind 会把它传播到 `/mnt/{user,installer,androidwritable}/0/emulated/0`，
   还可能残留上一次启动的别的路径 —— 现在一律从 `/proc/mounts` 现取，命中即拒绝启动。
3. **`union-guard` 子命令废弃**：显式报错并 `exit 2`。常驻看门狗只能由 `service.sh`
   驱动单轮 `ctl.sh union-check`，从结构上消除「脚本里有第二套常驻循环」这个隐患。

### 验证

| 场景 | 结果 |
|---|---|
| `--upper /storage/emulated/0`（事故参数） | **0 秒拒绝启动**，`rc=1`，无残留挂载 |
| `--upper /storage/emulated/0/DCIM/x`（子路径） | 同样拒绝 |
| `--upper /mnt/union`、`/mnt/union/x` | 拒绝（ctl.sh 逻辑测试） |
| 合法 upper（隔离点） | 正常挂载、146 条目可读、4 线程、`kill` 后 0 残留 |
| 生产（MI8） | 5 个挂载点齐全、`waiting=0`、`ls /sdcard` 0.43 s、读写实测 OK、看门狗在线 |

> 老设备若已连跑多天，升级后**建议重启一次**，让 `service.sh` 重新拉起看门狗循环
> （旧守护进程不会被文件替换掉）。

## v1.5.1（versionCode 17）
**union_fuse 从单线程改多 worker**：慢请求只占住一个 worker，其余继续服务。

### 为什么改：一次卡死，整个 `/sdcard` 陪葬

单线程时，只要有一个请求卡在慢 IO 上（upper 在外置盘，一次 copy-up 可能几百毫秒），
后面积压的全部请求一起排队。实测（2026-10-17）：

| 指标 | 故障时 |
|---|---|
| 状态为 `D` 的进程 | **13 个**（3 个 `df` 卡 594–975 s + 10 个 App） |
| `loadavg`（6 核） | **28.40** |
| `/sys/fs/fuse/connections/104/waiting` | 15 → 16，**只增不减** |
| `/mnt/iscsi`、`/mnt/iscsidisk` 响应 | **0 秒**（iSCSI 层完全健康） |
| `/mnt/union`、`/sdcard` 响应 | **超时 5–8 秒** |

iSCSI 层四个 worker 全在 `ppoll` 等活 —— 故障**不在** iSCSI，而在合层。

### 三条硬约束（少一条就出静默损坏或收工挂死）

1. **`/dev/fuse` 必须 `O_RDWR | O_NONBLOCK`**。两个 worker 同时 poll 到同一条请求时，
   抢输的那个会在阻塞 `read` 上永久卡住，回不到 poll 复查 `g_stop`，收工 `pthread_join` 直接挂死；
   `read` 拿 `EAGAIN` 必须 `continue` 不能 `break`（`break` = worker 永久退出，线程数悄悄变少）。
   `reply` 的 `write` 同样要 EAGAIN 重试，丢了回复 = 内核里那条请求永远挂着。
2. **请求 / 回复 / 读暂存三块缓冲必须 worker 私有**。老实现是全局 `g_obuf` / `g_rbuf`，
   多线程下会把 A 文件的数据回给 B，**而且完全不报错**。
3. **锁只罩共享元数据，真实 IO 一律在锁外**，否则并发等于没加。

### 两个不是「加把锁」就完事的地方

- `id2path` 改为返回 **4 槽 `__thread` 轮转缓冲的拷贝**。`id_forget` 会 `free(path)`，
  裸指针在多线程下就是 UAF；4 槽是因为 RENAME 分支会**同时**持有 `oparent` 与 `nparent`，
  单槽会互相覆盖成错路径。
- `copy_up` 用**独立**的 `g_cp_lock`。不加锁 → 两个 worker 同时写同一个未 copy-up 的文件会
  双双 `O_TRUNC` 交错写坏；用 `g_lock` → 搬几 GB 文件会把 node/fh 表和 LOOKUP 全堵死，
  等于换个地方复现单线程停摆。

`INIT` 的 `max_background` 12→128、`congestion` 9→96（旧值是单线程时代定的，
排队一到 12 就停住，4 个 worker 喂不饱）。

### 顺带补上的自救能力

- `ctl.sh` 新增 `fuse_conn_of` / `do_union_abort` / `union_fuse_stuck` / `do_union_check` /
  `do_union_revive`；`do_union_stop` 重写为「**先判 D → `abort` 断连 → TERM/KILL → 逐个带超时卸载**」。
  关键认知：FUSE 卡死时 **`kill -9` 无效、`umount` 超时**（`request_wait_answer(nofail)`
  走 `TASK_UNINTERRUPTIBLE`），唯一解药是往
  `/sys/fs/fuse/connections/<minor>/abort` 写 `1`。
- `do_union_start` **删掉 `df` 验证**改纯文本判据（`df` 走 `statfs` → FUSE 请求，
  union 一卡 `df` 就进 D，SIGKILL 都杀不掉，实测把 `ctl.sh start` 拖挂 180 s+）。
- `service.sh` 加 **30 s 常驻看门狗**，连续 3 轮异常才动手 —— 正常 IO 也会有瞬时 D，
  单轮判 D 会误杀。

### 验证

隔离挂载点（完全不碰生产）：并发读 8/8、混合 8 读 + 8 写 8/8、4×4 MiB 并发读 4/4 **全部内容一致**；
`TERM` 后 **0 秒干净退出**、无挂载残留、线程全回收。

切生产后：5 个挂载点齐全、8 MiB `/sdcard` ↔ upper **md5 MATCH**、8 路并发读 8/8 正确、
`D` 进程 **0**、`loadavg` 0.53、`union-check` 0 s 返回、重连 0 / 写失败 0。

> 退回单线程不用换二进制：`ctl.sh` / `config.default` 里把 `union_workers` 改成 `1`。

## v1.5.0（versionCode 16）
**并发 I/O 窗口（QD>1）**：写 **13 → 35~38 MB/s**（约 **2.9x**，四轮稳定）、
读 **26~28 → 43~47 MB/s**。写侧之前那句「16 MB/s 就是 NAS 写天花板」**是错的** ——
那只是 **QD=1 的往返延迟上限**。

### 真因：写路径「发一条等一条」，队列深度恒为 1

`scsi_write_raw` 是同步发一条等一条，外加全局互斥 ⇒ 队列深度只有 1，吞吐被往返延迟锁死。
协议级 A/B（同一 LUN、同一偏移窗口、字节级 readback 校验）：

| 场景 | 吞吐 |
|---|---|
| 写 1 MiB `QD=1` | 18.8 MB/s |
| 写 1 MiB `QD=2` | 33.97 MB/s |
| 写 1 MiB `QD=8` | **50.8 MB/s** |
| 写 1 MiB `QD=16` | 47.0 MB/s |
| 写 1 GiB `QD=8` | 47.7 MB/s（**不是**目标缓存假象） |
| 读 1 MiB `QD=1 / 2 / 4` | 40 / 55.1 / 57.5 MB/s |

### 改了什么

1. **新增 `iscsi_io()` 并发窗口**：一次放飞 N 条 SCSI 命令，**按 ITT 路由收包**
   （目标确实会乱序回），`ExpStatSN` 取「最大值」以容忍乱序。
   整批仍持 `cmd_lock` ⇒ v1.4.8 的保活线程与重连语义**无需改动**。
2. 回写改为**攒够 8 个整脏块一次批量放飞**（`--wqd` 默认 8）。
3. 4 MiB 跨度预取**拆成 4×1 MiB 并发**（`--rqd` 默认 4）。
4. FUSE 守护改 **4 个 worker**（`--workers` 默认 4；回复缓冲与读暂存**每线程独立** ——
   全局缓冲在多线程下会**静默串数据**）；块缓存与预取游标加 `cb_lock` 保护。

### 但部署后写吞吐纹丝不动（仍 13 MB/s）—— 插桩才挖出真因

内核送下来的写**全是 4 KiB 碎片**（`单条最大=4096 B`），而块缓存一满（16 MiB）就
**无条件 `cb_flush_all()`** ⇒ 每新攒 1 MiB 就冲掉刚攒的那 1 块，**队列深度恒为 1**。
先改 `cb_alloc` **优先挑干净块复用**（只有全脏才真 flush）＋ 缓存 16→32 块，
仍只到 21~26 MB/s。

**关键杠杆**：内核 INIT 里**明确提供了 `FUSE_WRITEBACK_CACHE`（capability bit16，
`kflags=0x001ffffb`）而我们从没要过**。开启后内核用**页缓存攒批**，把 4 KiB 碎片
合并成大块再下送：

- `单条最大` **4096 B → 131072 B**
- `批量回写` **1.24 → 10.7 段/次**
- 缓存命中率 **89%**

### 真机实测（256 MiB、`conv=fsync`）

| 测法 | v1.5.0 | v1.4.9 |
|---|---|---|
| 新写 `/mnt/iscsidisk` | **35~38 MB/s** | ~13 |
| 覆盖写（同文件） | 22~28 MB/s | ~13 |
| 冷读回（exfat+loop） | **43 MB/s** | 26~28 |
| 冷读 loop 窗口 | **46 MB/s** | ~30 |
| FUSE 直读（绕开 loop） | **47 MB/s** | 36~40 |
| 经 `/sdcard`（双层 FUSE） | 写 25~29 / 读 45 MB/s | — |

### 正确性零妥协

写回缓存是**异步**的，因此用**最严**的方式验证：
「源 → `/sdcard` → 裸 exfat → **重启守护 + 丢全部缓存**再读」，**四路 md5 全一致**
（`30ed5a26…`），并把样本 **pull 回 PC 逐字节交叉核对一致** ⇒ 写回缓存下**无静默数据损失**。

> 已知特性：**同一区域反复覆盖写**（22~28 MB/s）慢于新写（35~38），休息后只回到 22~28
> —— 不是 NAS 一次性节流，数据均正确。

## v1.4.9（versionCode 15）
**读吞吐 +29%**（FUSE 层实测 28 → 36 MB/s；真机 `/mnt/iscsidisk` 顺序读 25 → 30 MB/s）。
这一版是**纯读路径优化**，写路径与功能语义完全不变。

### 根因：读吞吐几乎只由「单条 SCSI READ 的大小」决定

探针在同一个 LUN 上只读实测（每个尺寸搬 32 MiB）：

| 单条 READ 大小 | 吞吐 |
|---|---|
| 256 KiB | 24.5 MB/s |
| 1 MiB（旧上限） | 37.9 MB/s |
| 2 MiB | 39.3 MB/s |
| **4 MiB（新上限）** | **42.6 MB/s** |
| 8 MiB | 44.5 MB/s |

⇒ 4 MiB 之后收益趋平（4→8 MiB 只 +5%），所以取 4 MiB。读不受写侧 `MaxBurstLength` 限制 —— Data-In 由目标自己流水发，命令要多少它给多少。

### 改了什么

1. **读单独用 4 MiB 上限**：新增 `RCHUNK`（写路径仍是 `CHUNK` = 1 MiB，不动）。
2. **顺序读跨度预取**：FUSE 层一条命令一次回源**连续 4 个 1 MiB 块**（合并成一条大读），
   判据是「上一次预取段的末尾」（`g_ra_next`）——
   **随机读保持单块**不变，避免把 4 KiB 随机读放大成 4 MiB（那会让延迟涨 3 倍以上）。
3. 缓存统计新增 **`跨度预取=N x 4块`**，可直接验证是否生效。
4. 新增 `--ra N` 开关（默认 4；`--ra 1` = 关闭跨度预取），便于现场对比。

### 顺带纠正一个流传已久的错误结论

写路径把 `MaxBurstLength` 抬到 1 MiB **其实没有生效** —— Synology LIO 只回 `262144`，
`1 MiB` 写依旧是 5 次往返（64K R2T + 3×256K R2T + Response）。
而写侧天花板在 **NAS 盘本身**：有线 SMB 实测并发写 23 MB/s、单流 17 MB/s，
我们 16 MB/s 已贴着单流上限 ⇒ **客户端已无优化空间，故本次不动写路径**。

### 测量与验收

- FUSE 层交替顺序 A/B（各 2 轮）：`--ra 1` = 30/30 MB/s，`--ra 4` = 37/36 MB/s。
- 真机：`重连=0 / 写失败=0`；`跨度预取=1408 x 4块`；
  exFAT 与 union 两条路径各做「写入 → 冷读（清页缓存）」md5 双向比对，**4/4 PASS**。
- 全路径（loop + exFAT）只到 28–30 MB/s：瓶颈已转移到 **loop 设备**（FUSE 层 36 → loop 后 30）。
  实测调 `read_ahead_kb` / `rotational` 只有 ~5%（噪声级），调度器本来就是 `none`，故未采纳。

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
