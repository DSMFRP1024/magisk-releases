# magisk-releases —— 更新分发包

本仓是 [magisk_packs](https://github.com/DSMFRP1024/magisk_packs) 各模块的**更新清单与安装包**分发点。
模块**源码仓保持私有**，本仓只公开「最新版」的 update json 与 zip，供设备端匿名检测与下载。

## 当前版本

| 模块 | 更新清单 | 最新版 | 包大小 | 源码仓（私有） |
| --- | --- | --- | --- | --- |
| **momGuard** | [momguard.json](momguard.json) | v1.5.7 | 171 329 B | [DSMFRP1024/momGuard](https://github.com/DSMFRP1024/momGuard) |
| **iscsiMount** | [iscsimount.json](iscsimount.json) | v1.5.0 | 143 225 B | [DSMFRP1024/iscsiMount](https://github.com/DSMFRP1024/iscsiMount) |

> 另有配套服务端应用 **momGuard-hub**（飞牛 fnOS 上的 `.fpk`，接收手机上报的位置 / 步数 / 用量），
> 不在本仓分发。

## updateJson 格式

`*.json` 遵循 [Magisk 官方 updateJson](https://topjohnwu.github.io/Magisk/guides.html#module-prop) 约定：

```json
{
  "version": "v1.5.0",
  "versionCode": 28,
  "zipUrl": "https://…/momGuard-v1.5.0.zip",
  "changelog": "https://…/CHANGELOG-momguard.md"
}
```

并额外带几个供**模块 WebUI** 使用的字段（Magisk 会忽略不认识的键）：

| 字段 | 用途 |
| --- | --- |
| `zipFile` / `changelogFile` | 文件名（不含 URL），WebUI 用它拼「当前可用的镜像源」 |
| `size` / `sha256` | 下载后校验完整性，防止半截包被安装 |
| `date` / `notes` | 界面展示的一行摘要 |

## 访问方式（两个镜像源）

- **raw**：`https://raw.githubusercontent.com/DSMFRP1024/magisk-releases/main/<file>` —— 无缓存延迟、最实时
- **jsDelivr**：`https://cdn.jsdelivr.net/gh/DSMFRP1024/magisk-releases@main/<file>` —— 国内可达性好

模块端（`bin/ctl.sh` 的 `check-update`）先试 raw、失败自动回退 jsDelivr。

## 发新版本时怎么更新

1. 把新 zip 复制进本仓根目录；
2. 改对应 `*.json` 的 `version` / `versionCode` / `zipFile` / `size` / `sha256` / `date` / `notes`，
   以及 `zipUrl` / `changelog` 两个完整 URL；
3. 更新对应 `CHANGELOG-*.md`；
4. 同步本文件上方「**当前版本**」表的版本号与包大小（最容易忘，改完对着 `*.json` 核一遍）；
5. 提交并推送 `main`。
