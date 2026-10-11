# 果酱罗盘 GuoJiangLuoPan

> **非官方逆向研究版**，基于一款商业风水罗盘 App（`原包名 com.mmc.fengshui.pass` 5.2.1）重建，仅供 Android 逆向与学习研究使用。

## 声明

- 本项目**不是原作者的官方版本**，与原始开发者无任何关联。
- 仅限个人学习研究，得到的二进制请勿分发；**禁止用于任何商业用途**。
- 原应用的商标、品牌、美术素材与知识产权归原权利人所有。
- 使用本仓库内容所产生的一切后果由使用者自行承担。

## 功能

研究版保留原应用的本地功能主体，主要包含：智能罗盘/实景罗盘/鲁班尺/水平仪、黄历择日、八字排盘、紫微斗数等模块界面与本地计算逻辑。

**会员校验已改为本地恒通过**（`UserVipManager` 单点），因此原会员墙被移除。但注意：

- 依赖服务端的排盘、订单、支付、社区等接口仍指向原服务器，研究环境下**不可用**，这是预期行为。
- 厂商推送、微信支付回调因包名变更同样失效。

## 目录结构

```
GuoJiangLuoPan.apk    安装包（存放于 Releases，见下方说明）
update.json           自更新版本清单
README.md             本文件
docs/
  ├── usage.md        使用说明
  ├── agreement.md    服务协议
  └── privacy.md      隐私政策
```

> **APK 为什么在 Releases 而不是仓库根目录？**
> APK 为 129 MB，超过 GitHub 单文件 100 MB 硬上限（超过即拒绝入库，且 raw 域名同样限 100 MB）。
> 因此 APK 以 Release 资产形式托管，`update.json` 的下载地址直接指向 Release：
> `https://github.com/LXFNice/GuoJiangLuoPan/releases/download/v5.2.1-research/GuoJiangLuoPan.apk`
>
> 文档类文件（`update.json`、三个 docs、README）体积小，仍走仓库根目录的 raw 域名，
> 这样 App 内「服务协议 / 隐私政策」页才能被 GitHub 渲染成网页。

## 构建信息

| 项 | 值 |
|---|---|
| 文件 | `GuoJiangLuoPan.apk` |
| 大小 | 129,405,592 字节 |
| SHA-256 | `a5d10082f6d6aaf92d04e795f4788e569a39f053f4c2ab24347fda7ecbfa232a` |
| 包名 | `com.gj.luopan.compass` |
| versionName / versionCode | `5.2.1` / `5021` |
| 签名 | MT 自签（V1 + V2 + V3），证书 SHA-256 `a40da80a59d170caa950cf15c18c454d47a39b26989d8b640ecd745ba71bf5dc`（**与线上 APK 必须一致，重编译后需同步更新本表**） |
| 最低 / 目标 SDK | 24 / 33，arm64-v8a |

## 发版流程

1. 用 MT 管理器编译出新 APK。
2. 把新 APK 作为资产挂到新的 Release tag（tag 递增，如 `v5.2.2-research`），
   或替换 `v5.2.1-research` 里的同名资产。
3. 更新 `update.json`：新 APK 地址 + `version` 加 1 + `isUpdate` 置 `1`。
4. 提交 `update.json` 推送，用户端「检查更新」即可拉到新版本。

### update.json 字段

App 内置的更新检查（`oms.mmc.version.update.UpdateRepository`）用 `JsonCallback<UpdateInfoModel>` **直接把响应体映射成模型**，字段名为 `appVerion / appTitle / appContent / appUrl / isUpdate / type / guideType / width / height / isMyself / appBundle / background_url / appPackageHash`。

为了同时兼容通用字段命名，本仓库的 `update.json` 采用**两套字段并存的超集**写法：

- 通用：`version`(int)、`forceupdate`(1=强制更新)、`url`(APK 地址)、`title`、`message`
- App 模型：`appVerion`、`appTitle`、`appContent`、`appUrl`、`isUpdate`(1=提示更新) 等

发新版本时把 `version` 加 1、把 `isUpdate` 置 `1`、更新 `appUrl`/`url` 指向新 APK 即可。

> App 内的更新地址已在 `oms.mmc.version.update.UpdateConfig` 中改为
> `https://raw.githubusercontent.com` + `/LXFNice/GuoJiangLuoPan/main/update.json`。

## 技术要点（复现笔记）

| 项 | 说明 |
|---|---|
| 是否加壳 | **否**。dex 完整可读，无需脱壳 |
| 技术栈 | Flutter 混合开发（业务 Dart 代码在 `lib/arm64-v8a/libapp.so`）+ 原生 Kotlin 模块 |
| 会员单点 | `com.wz.platform.base.pay.vip.UserVipManager`：构造函数 5 个 VIP 字段直接置 true，5 个对外 getter 恒返回 true；`UserVipInfoBean.isCurrentAvailable()` 恒 true 防止服务端刷新回来 |
| 改包名 | 只改 `AndroidManifest.xml` 的 `package`。authorities / 权限名 / 推送 category 均为自有字符串，与 `getPackageName()` 无关，保留无损；dex 与 so 内无包名硬编码。**注意**：provider authority、`<permission>` 名必须一并改前缀，否则报 `CONFLICTING_PROVIDER` / `DUPLICATE_PERMISSION` |
| 链接改指向 | 服务协议 / 隐私政策 URL 改为本仓库的 GitHub 渲染页 |
| **签名绑定密钥（最大的坑）** | `oms.mmc.encryption.RawKey` 用**签名证书的 SHA1** 做白名单，决定使用哪个 `*_key.dat`：`C8611436…` → `suanming_key.dat`，`67F9AF…` → `debug_key.dat`，`D4DED8…` → `duliban_key.dat`，`300777…` → `other_key.dat`，**都不匹配则回退 `simple_key.dat`**（本包 assets 里根本不存在该文件）。重签名后签名变了 → 回退 → `FileNotFoundException` → 黄历解密失败 → 释放出 0 字节数据库 → `no such table: huangli_cn` 闪退 |
| 修法 | 不改密钥本身，而是让 `oms.mmc.encryption.SignKeyTool.呢褖鍒掲(Context,String)` **恒返回原版证书**（把原版 `META-INF/*.RSA` 里的证书 DER 取出来，Base64 硬编码进 smali，用 `new Signature(Base64.decode(...))` 返回）。这样 `RawKey` 的 SHA1 白名单必定命中 `suanming_key.dat`，并以原版证书公钥完成 RSA 解封，完整复刻原版行为 |
| 资源解密链路 | `assets/data_hlwfestival.dt` → `AES/ECB/PKCS5Padding` 解密 → ZIP → 取条目 `db_hlwfestival_data.db` → 释放为 `files/hldb/<versionCode>db_hlwfestival_data.db`（2,051,072 字节） |
| 校验签名是否修对 | `adb shell ls -l /data/data/<pkg>/files/hldb/`，文件应为 2,051,072 字节而非 0 |

## 免责声明

本项目仅用于技术学习与研究，不构成任何形式的授权或许可。请在下载后 24 小时内删除研究产物，支持正版应用。
