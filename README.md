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
| 大小 | 129,782,424 字节 |
| SHA-256 | `8371a406ab3d51ae6e9eb8ba786da0568f2128795fd24ba11055f75cfeabbf59` |
| 包名 | `com.gj.luopan.compass` |
| versionName / versionCode | `5.2.1` / `5021` |
| 签名 | MT 自签（V1 + V2 + V3），证书 SHA-256 `a40da80a59d170caa950cf15c18c454d47a39b26989d8b640ecd745ba71bf5dc`（**与线上 APK 必须一致，重编译后需同步更新本表**） |
| 最低 / 目标 SDK | 24 / 33，arm64-v8a |

### 本次变更（v5.2.1-research r3）

r2 已完成底部 Tab 栏 5→3（移除「服务」「消息」）。r3 在首页与「我的」页做深度精简：

| 页面 | 变更 |
|---|---|
| 首页 | ① 直接罗列原「更多」页的 7 组工具（排盘/祈福/每日/学习/Ai/罗盘/其他），无需再点「更多」；② 过滤顶部横幅卡片（悬赏问答/张三/续费VIP）；③ 移除祈福台入口与「测算」tab |
| 我的 | 整页只保留「排盘设置」与「更多操作」：删除罗盘VIP卡、我的财富（元宝/金豆/优惠券/会员特权）、我的订单（测算/咨询/订单/悬赏）三个区块，以及设置列表里的其他 6 项 |

首页工具铺列的实现：`HomeFragment` 的 pageId 由 761/592 改为 596（原「更多」页容器 pageId），首页即按「更多」页数据源渲染全部工具组；横幅卡片在渲染层按卡片类型过滤，不影响下拉刷新与工具点击。

**「我的」页崩溃修复（r3 关键坑）**：删除上述三个区块时，被删区块内负责把 v11 归一为 int 的
`const/16 v11, 0x230`（rememberComposableLambda 的 key 与 changed-flags 常量）一并被删，
导致幸存代码末尾的 section composable 调用读 v11 时类型冲突：

```
java.lang.VerifyError: Verifier rejected class ...HomeMineFragment$InitView$1$3:
void ...$InitView$1$3.镯鈷昳墒鸐鄘(Composer, int) failed to verify:
[0xFC] register v11 has type Conflict but expected Integer
```

该错误在 `HomeMineFragment.坦槾砓東鄩` 的 `new-instance $InitView$1$3`（类加载/校验点）抛出，
被友盟 UCrash 的 uncaughtException 吞掉（logcat 无 FATAL，只有 `Process is going to kill itself`），
表现为「我的」页打开即闪退。定位手段：在 InitView 内部 4 个关键路径点埋 `Log.e("GJAD", tag)` 探针
（colorResource / LiveData check-cast / RefreshLayoutState / VerticalRefreshableLayout），
最后一条探针 IV_STATES 之后崩溃，再拉取设备上 `crashsdk/ucrash/*.log.gz` 拿到完整堆栈。

修法：在幸存的那个 section 调用前补回 `const/16 v11, 0x230`，恢复原始寄存器状态。
**教训**：删除 smali 区块时，必须检查「区块内写入、区块外读取」的寄存器——v11 这类跨区块复用的
归一化常量最隐蔽，线性扫寄存器用法发现不了，要靠差分分析（orig/post 读点状态对比）或实机验证。

### 紫微命盘解读付费墙移除（r4）

紫微排盘（`ZiWeiAnalysisActivity` / 命书 `ZiWeiMingShuFragment`）原有付费墙：

- 进入命盘分析页即弹「·紫微命盘解读·」购买框：命盘解读 ¥98、紫微全解+八字全解 ¥228、VIP 会员 ¥128/3 个月、¥288/1 年
- 页内常驻「解锁全部」悬浮按钮，点击同样弹购买框

实现：付费弹窗统一由 `FortuneOrAnalysisPayDialogUtil` 的两个公开方法构建并 show——
`雀啘阇噛厗峽冞(FragmentActivity, Z, String, Function2)`（命书/命盘解读）与
`饈禼慯荀悉睥鹦翽(FragmentActivity, Z, I, String, Function2)`（流年运势）。
将两个方法体替换为直接 `return-void`（保留签名与注解），所有调用方均不再弹窗；
「解锁全部」按钮的点击链路（埋点 + `嗕偀卺邬渜胎迤孙琖肃毨()` → 上述方法）随之成为无动作死按钮。
命书/流年的「全部解读」按钮渲染方法（`ZiWeiMingShuFragment.衛艞庴` /
`ZiWeiFortuneYearFragment.铿紫摏`）同样清空，命书与流年页不再渲染该按钮；
排盘页（命宫 tab）的悬浮按钮为另一处 Compose 渲染，视觉残留但点击无任何反应。

实机验证：命盘页与命书页进入即显示完整内容（命宫/主星/星象/福德宫/官位等），
无弹窗、无计费跳转；「解锁全部」点击无动作；首页 7 组工具与「我的」页 2 项保持 r3 成果。

#### 附：r2 的 Tab 精简技术要点

Tab 由 `HomeViewModel.鹇劔呶鮕痿巻叝灠汊()` 动态绑定，分两条分支取列表——

| 条件 | 取用的方法 | 本次是否改动 |
|---|---|---|
| `AppStatusManager.嘇膅寞巈獿钏蔥崑兰鄓秒梟()` 为真（已登录） | `脁太圐窷猶杘駪煇()`，实现在 `FSHomeTabGenerateImpl` | ✅ |
| 否则（未登录，实际走这条） | `镭凯娷噬麎顴糦缾塳莖詳()`，实现在父类 `HomeTabHandler$HomeTabFragmentGenerate` | ✅ |

两条分支都必须改，否则未登录状态下底栏仍显示 5 项。

底栏布局 `res/Fhe.xml` 里固定写死 5 个 `RadioButtonWithRedPointNotify`，代码按索引绑定
（`List.get(index)`），超出列表长度的会被自动 `setVisibility(GONE)`，所以无需改布局。

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
