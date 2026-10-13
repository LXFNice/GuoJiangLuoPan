# 果酱罗盘 (GuoJiangLuoPan)

基于「通易罗盘指南针」的研究版重构包，包名 `com.gj.luopan.compass`。

## 版本

| 版本 | 说明 |
| --- | --- |
| 5.2.1.14 (5031) | **r14（推荐）**：修复 r13 启动闪退 + 首页区块被全部清空；功能同 r13 |
| 5.2.1.13 (5030) | r13：首页精简（去掉祈福 / Ai / 其他工具）；更新下载改为 App 内置下载器（进度条 + 自动安装）。⚠️ 该包启动即闪退，请勿使用 |
| 5.2.1.12 (5029) | r12：精简「更多设置」页面（移除 7 项，仅留隐私政策 / 用户协议 / 版本号） |
| 5.2.1.11 (5028) | r11：修复检查更新取到旧数据（禁用 HttpURLConnection 缓存 + 时间戳） |
| 5.2.1.10 (5027) | r10：新增独立「检查更新」页面（当前版本 + 检查更新按钮 + 弹框） |
| 5.2.1.9 (5026) | r9：点击版本号可检查更新（弹框 + 立即安装下载） |
| 5.2.1.7 (5024) | r7：修复弹框路径；更新源切到直连镜像 |
| 5.2.1.6 (5023) | r6：检查更新链路打通 |
| 5.2.1.5 (5022) | r5：点击版本号可检查更新 |

## 首页工具区块（r13 起）

首页已移除三个工具区块：**祈福工具**、**Ai 工具**、**其他工具**。
保留：排盘工具 / 每日工具 / 学习工具 / 罗盘工具。

> 首页区块由服务端数据驱动（`oms.mmc.repository.dto.model.AdBlockModel` +
> `oms.mmc.fast.multitype.RAdapter` 多类型列表），所以用隐藏 View 的方式会留下空白占位。
> 实际改法：在数据提交总入口 `RAdapter.銢鷷顓眓婺鄠鑖馺賾敲鮰(List, DiffUtil$Callback)`
> 插入 `gjFilter()`，按 `AdBlockModel.getTitle()` 剔除指定标题的区块。
> 该方法同时被 `絹譾頼(List)` 的桥接路径调用，两条入口都被覆盖。

### 踩坑记录（r13 → r14 修复）

**坑 1：`.registers` 分配不足 → 启动闪退**

smali 里 `.registers N` 指的是**总寄存器数**（含参数寄存器）。
非静态方法中 `p0` 不是独立寄存器，而是映射到 `v(N - 参数个数)`。

r13 的 `gjFilter` 用了 `.registers 5`，于是 `p0`(this) → `v3`、`p1`(List) → `v4`，
而方法体内又写了 `const/4 v3, 0x0`，**把 this 引用覆盖成了 int**。
ART 校验器直接拒绝整个类：

```
java.lang.VerifyError: Verifier rejected class oms.mmc.fast.multitype.RAdapter:
  java.util.List gjFilter(java.util.List) failed to verify:
  [0x42] returning register with conflict
```

→ 打开 App 立刻闪退（`InflateException: FastListView` 只是表面现象，根因在 RAdapter）。

**修复**：`.registers` 提到 7，参数落在 `v5`/`v6`，方法体只用 `v0`~`v4`。

**原则**：写 smali 时先算 `p` 寄存器落点（`vN = 总寄存器数 - 参数个数`)，
再挑本地寄存器区间，绝不重叠；同时保证**每个寄存器全程只保持一种类型**，
避免 verifier 在分支汇合点报类型冲突。

**坑 2：跳转条件写反 → 整页清空**

三个关键字判断原写成 `if-eqz v4, :gj_loop`（结果为 0 才跳），
实际语义变成「**不包含**关键字就丢弃」，结果把所有区块全过滤掉了，
首页只剩一个空壳。

→ 改为 `if-nez v4, :gj_loop`（包含关键字才跳），恢复正常。

## 应用内检查更新

**入口**：我的 → 更多操作 → 更多设置 → 点击「版本号」

点击后进入**独立的「检查更新」页面**：

1. 页面顶部显示「返回」+「检查更新」标题
2. 页面内容显示「当前版本」+ 版本号（读 `PackageInfo.versionName`）
3. 点「检查更新」按钮 → 显示转圈进度条 + 「正在检查更新...」
4. 请求 `update.json` 并比对版本：

| 场景 | 行为 |
| --- | --- |
| 有新版本 | 弹出「发现新版本」对话框（含更新日志），按钮「下载更新」/「取消」 |
| 已是最新 | 弹出「已是最新版本」对话框 |
| 网络异常 | 弹出「检查更新失败」对话框 |

判断依据：`update.json` 的 `appVerion` 与本地 `versionName` 比较，相等即视为已是最新。

### 下载与安装（r13 起改为 App 内置下载器）

点「下载更新」后**不再跳转系统下载器**，而是 App 自己下载：

1. 弹出「正在下载更新」进度条弹窗，实时显示百分比（` ProgressDialog` 水平样式）
2. APK 通过 `HttpURLConnection` 下载到 `getExternalFilesDir("Download")`
   （App 私有目录，Android 10+ 分区存储下无需任何存储权限）
3. 下载完成 → 自动关闭弹窗并**跳转系统安装界面**覆盖安装
4. 失败 → 提示「下载失败，请检查网络后重试」

> 背景：系统 `DownloadManager` 在部分机型上会出现「跳转后下载不了 / 通知栏进度看不到 /
> 下载完还要手动点通知才能安装」的问题，内置下载器规避了整条链路。

**线程切换**复用 `PrivacyWebActivity` 已实现的 `Runnable`，用 `gjMode` 分派：

| gjMode | 执行线程 | 动作 |
| --- | --- | --- |
| 0 | UI | `gjShowResult()`（检查更新结果） |
| 1 | 后台 | `gjDoDownload()`（下载，中途 setProgress） |
| 2 | UI | `gjInstall()`（关弹窗 + 跳安装界面） |
| 3 | UI | `gjDlFail()`（关弹窗 + 提示失败） |

> `ProgressBar.setProgress()` 本身就是线程安全的，所以后台线程可以直接更新进度条，
> 不需要额外 Handler。

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

## 更多设置页面（r12 起）

**入口**：我的 → 更多操作 → 更多设置

页面仅保留三项：

| 项 | 行为 |
| --- | --- |
| 隐私政策 | 打开隐私政策网页 |
| 用户协议 | 打开用户协议网页 |
| 版本号 | 显示当前版本，点击进入「检查更新」页面 |

已移除：个人信息采集清单、第三方信息共享清单及 SDK 目录、应用权限申请与使用情况说明、
联系客服、通知提醒、个性化推荐、备案号。

> 实现方式：在 `MineSettingActivity.initView()` 末尾对相应控件调用 `setVisibility(GONE)`；
> 「个性化推荐」与「备案号」两行隐藏的是其**父容器**（整行），保证无残留空白。布局 XML 未改动。

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
