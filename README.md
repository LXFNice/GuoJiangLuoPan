# 果酱罗盘 (GuoJiangLuoPan)

基于「通易罗盘指南针」的研究版重构包，包名 `com.gj.luopan.compass`。

## 版本

| 版本 | 说明 |
| --- | --- |
| 5.2.1.11 (5028) | r11：修复检查更新取到旧数据（禁用 HttpURLConnection 缓存 + 时间戳） |
| 5.2.1.10 (5027) | r10：新增独立「检查更新」页面（当前版本 + 检查更新按钮 + 弹框） |
| 5.2.1.9 (5026) | r9：点击版本号可检查更新（弹框 + 立即安装下载） |
| 5.2.1.7 (5024) | r7：修复弹框路径；更新源切到直连镜像 |
| 5.2.1.6 (5023) | r6：检查更新链路打通 |
| 5.2.1.5 (5022) | r5：点击版本号可检查更新 |

## 应用内检查更新

**入口**：我的 → 更多操作 → 更多设置 → 点击「版本号」

点击后进入**独立的「检查更新」页面**：

1. 页面顶部显示「返回」+「检查更新」标题
2. 页面内容显示「当前版本」+ 版本号（读 `PackageInfo.versionName`）
3. 点「检查更新」按钮 → 显示转圈进度条 + 「正在检查更新...」
4. 请求 `update.json` 并比对版本：

| 场景 | 行为 |
| --- | --- |
| 有新版本 | 弹出「发现新版本」对话框（含更新日志），按钮「下载更新」/「取消」；点「下载更新」→ 系统下载器开始下载，通知栏显示进度 |
| 已是最新 | 弹出「已是最新版本」对话框 |
| 网络异常 | 弹出「检查更新失败」对话框 |

判断依据：`update.json` 的 `appVerion` 与本地 `versionName` 比较，相等即视为已是最新。

> 注意：Android 的 `HttpURLConnection` **默认带响应缓存**，发版后 App 内可能长时间读到旧 `update.json`。
> 本版已通过 `setUseCaches(false)` + 请求头 `Cache-Control: no-cache` + URL 时间戳解决，
> 每次点「检查更新」都会拉取线上最新内容。

**更新源**（App 内 `oms.mmc.version.update.UpdateConfig`）：

```
https://ghproxy.net/https://raw.githubusercontent.com/LXFNice/GuoJiangLuoPan/main/update.json
```

> 为什么不用 jsDelivr：jsDelivr 对 `@main` 这类**分支引用**存在约 12 小时缓存，
> 且 `purge.jsdelivr.net` 对分支引用实测**不生效**，导致发版后长时间仍拿到旧数据。
> 也试过 statically.io / githack / gitmirror / ghfast.top / gh.llkk.cc，国内设备均不可达。
> ghproxy.net 实测稳定（5/5 成功，0.9~2.2s），缓存只有几分钟。

## 发版流程（重要）

**每次发版都必须「新增 Release + 写更新日志」，不要替换已有 Release 的资产。**

1. 修改 `AndroidManifest.xml` 的 `versionCode` / `versionName`
2. 构建并签名 APK（V1 + V2 + V3）
3. **新建** Release，tag 形如 `v5.2.1-research-rN`，正文写本次更新日志
4. 上传 APK 到该 Release
5. 更新 `update.json`：
   - `version` 自增
   - `appVerion` = 新版本名（**必须与 APK 内的 versionName 一致**）
   - `appUrl` / `url` = 新 Release 的资产下载地址
   - `appPackageHash` = 新 APK 的 sha256
   - `appContent` = 本次更新日志（多行用真实换行）
   - `isUpdate` = 1
6. 等 3~5 分钟让 raw / 镜像缓存刷新，再验证

> 历史 Release 一律保留，方便回滚与追溯。

## 签名

- 证书 sha256：`a40da80a59d170caa950cf15c18c454d47a39b26989d8b640ecd745ba71bf5dc`
- 方案：V1 / V2 / V3
