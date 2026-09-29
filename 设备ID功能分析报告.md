# ChaoxingSignFaker「设备ID / 设备码」功能分析报告

> 分析对象：aquamarine5/ChaoxingSignFaker（超星学习通第三方签到客户端，Android/Kotlin）
> 重点版本：**1.19.1-rc9-260929**（versionCode `1_1901_9009`，2026-09-29 发布，预发布版）
> 报告日期：2026-09-29
> 说明：本报告基于仓库实际源码逐行分析写成。除特别注明外，所有"新版/最终版"均指 `v1.19-apologize-dev` 分支（即 1.19.1-rc9 源码），"旧版"指 `main` 分支（1.19.0-stable-260918）。

---

## 0. 重要前提：版本与源码位置说明

- 本地 `main` 分支（同时也是 GitHub 仓库 main 分支最新）= **1.19.0-stable-260918**（versionCode `1_1900_6766`）。
- **1.19.1-rc9-260929 的源码并不在 main 分支上**，而在开发分支 **`v1.19-apologize-dev`**（最新提交 `7d187e3`，提交信息 `1.19.1@26`，2026-09-29 00:56）。该分支共 26 个 `1.19.1@N` 系列提交，GitHub Release 的 `1.19.1-rc9` 预发布 APK 即由此分支构建。
- 1.19.1 系列 GitHub Release 情况：`1.19.1-rc1`（2026-09-21）与 `1.19.1-rc9`（2026-09-29）两个预发布。**设备码绑定功能是 rc1 之后（开发提交 `1.19.1@14`，2026-09-24）才加入的，因此它是 rc9 相对 rc1 的核心新能力之一**，与用户所述"1.19.1-rc9-260929 版本支持了绑定设备ID"完全吻合。

---

## 1. 「设备ID」在这个项目里到底指什么

代码中与"设备ID"相关的标识不止一个，必须先分清，否则极易混淆。项目里实际存在 **四类** 设备标识：

| # | 名称 | 生成方 | 存在形式 | 主要用途 |
|---|------|--------|----------|----------|
| 1 | **deviceCode（设备码）** | 本 App 自行生成 | 64 字节二进制 → Base64 字符串（88 字符） | 签到请求 URL 的查询参数，**1.19.1 绑定功能的主角** |
| 2 | **deviceUniqueId / cdid / device_id** | 本 App 自行计算 | 设备信息 JSON 中的字段 | RSA 加密后上报给超星服务器（设备指纹） |
| 3 | **clientId** | 超星服务器签发 | RSA 加密字符串（`ChaoxingUserEntity.clientId`） | 服务器端客户端标识；解密后用于人脸签到 signToken 计算 |
| 4 | **OAID / mediaDrmId / android_id** | 系统或第三方 SDK | 原始字符串 | 设备指纹的组成来源；1.19.1 中 OAID 成为 deviceCode 的种子 |

### 1.1 deviceCode（设备码）——本文核心

这是签到请求中直接提交给超星服务器的设备标识，格式为 Base64 字符串。**它完全由客户端自己生成，服务器只负责接收**。它的生成算法（随机版）：

```
rawData = SHA-256( UUID1去掉横线 + UUID2去掉横线 )   // 32 字节
deviceCode = Base64( rawData + rawData )             // 64 字节，Base64 后 88 字符
```

见 `ChaoxingDeviceInfoHelper.randomizedDeviceCode()`（新版 L108-115）与旧版 `ChaoxingHttpClient.generateDeviceCode()`（旧版 L156-162，新版已标记 `@Deprecated` 并转发到前者）。

### 1.2 deviceUniqueId / cdid / device_id（设备指纹 JSON）

`ChaoxingDeviceInfoHelper.buildDeviceInfo()`（新版 L138 起）构造一个 JSON，其中 `deviceUniqueId` 的计算方式：

```
deviceUniqueId = sha256( "包名:ANDROID_ID:Build.FINGERPRINT" )
```

该 JSON 同时包含 `cdid`、`device_id`（三个字段值相同）、`android_id`、`mediaDrmId`、`oaid`（固定空串）、机型/系统/分辨率/学习通签名摘要等约 20 个字段，经 RSA-1024 公钥（`DEVICE_INFO_PUBLIC_KEY`，分段 117 字节，`RSA/ECB/PKCS1Padding`）加密后，作为 `data` 表单字段 POST 到超星用户信息接口 `https://sso.chaoxing.com/apis/login/userLogin4Uname.do`。若本机装了学习通，还会读取**学习通 APK 的真实签名摘要和版本号**来增强伪装。

### 1.3 clientId（服务器签发的客户端标识）

上述接口的响应中会返回 `clientId` 字段，存入 `ChaoxingUserEntity.clientId`（L32）。它本质是**服务器用同一对 RSA 密钥加密的设备信息回执**。`ChaoxingDeviceInfoHelper.decryptClientId()`（L48-72）用公钥做 `modPow` 手工解密（PKCS#1 v1.5 去填充），得到含 `cid`、`sc` 等字段的 JSON。人脸签到时 `ChaoxingFaceHelper.addSignToken()`（L112-134）用它计算：

```
signToken = MD5( 排序字段拼接(cxcid, cxtime, currentFaceId, ...) + sc )
```

即人脸签到的防伪签名，属于"设备ID"体系的服务器侧闭环。

### 1.4 OAID（匿名设备标识符）

由友盟 SDK `UMConfigure.getOaid()` 异步获取。旧版 `buildDeviceInfo()` 里 `oaid` 恒为空串；**1.19.1 起 OAID 被用作 deviceCode 的唯一种子**（详见第 3 节），是"绑定真实设备"的关键数据源。

---

## 2. 为什么要绑定设备ID（功能背景）

超星学习通的服务器会对签到请求做设备维度的风控。第三方客户端在旧版（≤1.19.0）中的行为是：

- `ChaoxingHttpClient` 构造函数中 `val deviceCode: String = generateDeviceCode()` —— **每次创建客户端实例都随机生成一个新设备码，且不持久化**（旧版 L57）。App 重启、重新登录、切换账号后设备码都会变。
- 同一台手机上代签多个小号（otherUser）时，所有小号各自持有一套随机生成的设备码。

这带来两个风控弱点：

1. **同一账号的设备码频繁变化**：真实用户的学习通客户端设备码是长期稳定的，一个账号的设备码每次签到都不一样，明显异常。
2. **代签场景下设备码与被代签者真实手机不一致**：A 的学习通平时在手机 X 上使用（设备码为 X 的），某天 B 用自己的手机给 A 代签时提交的是 B 手机上随机生成的设备码，服务器可以看到"该账号突然换了设备+异地"，这是代签检测的典型信号。

1.19.1 的绑定设备ID功能就是针对这两点：**让本机主账号使用一个稳定、且与真实设备关联的设备码；让被代签的小号使用"被代签者本人手机"的设备码**。

---

## 3. deviceCode 的两种生成方式（新版核心算法）

新增代码全部位于 `ChaoxingDeviceInfoHelper.kt`（相对旧版 +73 行）。

### 3.1 本机设备码 `getLocalMachineDeviceCode()`（L75-86）——与真实设备绑定

```
OAID（友盟 UMConfigure.getOaid()，1 秒超时）
   │ 无效（空/全零/超时失败）→ 返回空串 ""（不生成）
   ▼ 有效
AES/ECB/PKCS5Padding 加密（密钥 "QrCbNY@MuK1X8HGw"，常量 DEVICE_FLAG_INFO_KEY）
   ▼
Base64 编码 → deviceCode
```

细节（`getLocalMachineUniqueId()`，L116-136）：

- 通过 `suspendCancellableCoroutine` 把 `UMConfigure.getOaid()` 回调包装为挂起函数，外层 `withTimeout(1.seconds)`。
- 有效性校验 `INVALID_UNIQUE_ID_REGEX = ^0{16,64}$`：OAID 去掉 `-` 后若为 16~64 个 0 视为无效（部分国产 ROM 会返回全零）。
- **注意版本差异**：`1.19.1@14` 初版中 OAID 无效时曾用 `UUID.randomUUID()` 兜底；`1.19.1@19`（2026-09-26）重构后**去掉了 UUID 兜底，OAID 无效即返回空串**。也就是说：拿不到 OAID 的设备（如未装友盟可用组件、某些 ROM），本机设备码为空，系统自动退回随机设备码逻辑（见 4.2），不会绑定失败。
- AES 密钥 `QrCbNY@MuK1X8HGw` 硬编码在代码中（即超星客户端内使用的同一密钥，属于逆向学习通所得的"设备指纹信息"加密密钥，函数名 encryptFlagInfo 印证其用于 flag/deviceinfo 接口）。

### 3.2 随机设备码 `randomizedDeviceCode()`（L108-115）——与传统一致

即第 1.1 节算法，与旧版 `generateDeviceCode()` 完全相同，仅迁移了位置。用于没有绑定来源的场景（主要是小号默认值）。

### 3.3 缓存逻辑 `getCachedLocalMachineDeviceCode()`（L89-107）

- 优先读 DataStore `loginSession.deviceCode`，非空直接返回（保证同一账号设备码跨启动稳定）。
- 为空则调 3.1 生成；生成结果非空时写入 `loginSession.deviceCode` 并置 `isNotRandomizedDeviceCode = true`（"这是绑定的真实设备码，不是随机值"），再返回。
- 注意这里只写 `loginSession`（主账号），不涉及小号。

---

## 4. 「绑定」的三条路径（1.19.1-rc9 的完整绑定机制）

### 4.1 路径一：主账号自动绑定本机设备码

登录/恢复会话时：

- `ChaoxingHttpClient.create()`（新版 L276-277）：
  ```kotlin
  deviceCode = session.deviceCode.takeIf { it.isNotEmpty() }
      ?: ChaoxingDeviceInfoHelper.getCachedLocalMachineDeviceCode(context)
  ```
- `ChaoxingHttpRequester.loadFromDataStore()`（新版 L303-304）逻辑相同。

即：**主账号（登录会话）的设备码 = 已保存值，否则自动生成本机设备码并持久化**。此后无论重启 App 还是重新登录（`session.deviceCode` 非空时优先），设备码保持稳定，且由 OAID 派生、与本机真实设备一一对应。

### 4.2 路径二：小号（其他用户）默认随机设备码 + 持久化

`ChaoxingHttpRequester` companion 中的扩展函数 `resolveDeviceCode()`（L92-110）：

```kotlin
private suspend fun ChaoxingOtherUserSession.resolveDeviceCode(context: Context): String =
    deviceCode.takeIf { it.isNotEmpty() } ?: ChaoxingDeviceInfoHelper.randomizedDeviceCode()
        .also { code -> /* 写回 session：setDeviceCode(code), setIsNotRandomizedDeviceCode(false) */ }
```

即：**代签小号首次使用时生成一个随机设备码并永久保存**（`isNotRandomizedDeviceCode = false` 标记它是随机值）。设计意图很明确：不能让多个小号都绑定到代签者本机的同一个 OAID 设备码（那等于向服务器暴露"一堆账号都在同一台设备上签到"），所以小号默认保持随机但稳定。

### 4.3 路径三：分享导入时绑定对方的设备码（"绑定设备ID"最直接的体现）

这是功能的核心交互闭环，改动分布在 `ChaoxingOtherUserHelper.kt`：

**（a）分享端**——`getSharedUrl()`（L92-107）生成用户分享链接（二维码/文本链接同源）：

```
http://cdn.aquamarine5.fun/?phone=...&pwd=...&name=...&face=...&dc=<本机设备码>
```

新增的 `dc` 参数携带**分享者本机的设备码**（`getCachedLocalMachineDeviceCode(context)`，即 OAID 派生值；首次分享时若无缓存会即时生成并缓存）。

**（b）接收端**——两个入口都能解析 `dc`：

- 扫二维码/打开链接：`ImportOtherUserActivity.kt`（L100 `getQueryParameter("dc")`，L137 传入实体）；
- 手动粘贴链接：`OtherUserScreen.kt`（L1940-1956）。

**（c）落地绑定**——`saveOtherUser()`（L205 起）与私有扩展 `applySharedDeviceCode()`（L44-61）：

```kotlin
private suspend fun ChaoxingOtherUserSession.applySharedDeviceCode(
    context: Context, deviceCode: String?
): ChaoxingOtherUserSession {
    if (deviceCode.isNullOrEmpty()) return this          // 老版本分享链接没有 dc，不影响导入
    if (deviceCode == this.deviceCode && isNotRandomizedDeviceCode) return this  // 已绑定同一码，幂等
    val updatedSession = toBuilder()
        .setDeviceCode(deviceCode)
        .setIsNotRandomizedDeviceCode(true)              // 标记为"真实绑定"，非随机
        .build()
    /* 写回 DataStore */
}
```

- 对**已存在**的小号：导入时直接把对方的设备码覆盖绑定上去（即使原来是随机码也会被替换）。
- 对**新导入**的小号（`saveOtherUser` L336-339）：同样 `setDeviceCode(dc)` + `setIsNotRandomizedDeviceCode(true)`。

**业务含义**：A 想让 B 长期代签。A 在自己手机上打开"分享用户数据"生成二维码，链接里带着 A 手机的真实设备码；B 扫码导入 A 的账号后，B 的 App 之后给 A 签到时提交的 `deviceCode` 就是 A 手机的设备码。**在超星服务器看来，A 的账号始终在 A 自己的手机上签到**，设备维度完全一致，从而规避"换设备签到"这一代签检测特征。

### 4.4 deviceCode 何时被真正使用（签到请求）

5 个签到器全部携带，均在签到 URL 中以查询参数 `deviceCode=...` 提交：

| 签到器 | 代码位置（新版） |
|---|---|
| 位置签到 | `ChaoxingLocationSigner.kt` L74 |
| 手势签到 | `ChaoxingGestureSigner.kt` L89 |
| 密码签到 | `ChaoxingPasswordSigner.kt` L91 |
| 拍照签到 | `ChaoxingPhotoSigner.kt` L73 |
| 二维码签到 | `ChaoxingQRCodeSigner.kt` L136 |

目标接口：`https://mobilelearn.chaoxing.com/pptSign/stuSignajax`（及 preSign 等）。旧版同样携带 deviceCode 参数，区别只在值的来源（旧版每次随机，新版持久化绑定）。

---

## 5. 数据结构与架构变更

### 5.1 DataStore（protobuf）schema 变更（`app/src/main/proto/datastore.proto`）

```proto
message ChaoxingLoginSession{        // 主账号会话
  ...
  optional uint32 puid = 6;          // 新增：账号 puid 持久化
  optional string deviceCode = 7;    // 新增：绑定的设备码
  bool isNotRandomizedDeviceCode = 8;// 新增：设备码是否为"真实绑定"（非随机）
  string name = 9;                   // 新增：用户名缓存
}

message ChaoxingOtherUserSession{    // 小号会话
  ...
  optional uint32 puid = 9;
  optional string deviceCode = 10;
  bool isNotRandomizedDeviceCode = 11;
}
```

另：顶层 `string deviceCode = 5 [deprecated = true]`（2025-03 就存在的旧字段，配套旧版已废弃的 `Companion.saveDeviceCode()`）继续保留但彻底弃用——新实现把设备码下沉到了**每个会话**级别，支持不同账号各自独立绑定。

### 5.2 网络层重构（为设备码绑定服务的架构铺垫）

`1.19.1@19`（提交 `5651f18`，2026-09-26）完成了一次大规模网络层重构：

- 新增基类 **`ChaoxingHttpRequester`**（新文件，309 行）：持有 `okHttpClient / phoneNumber / name / puid / deviceCode / configuredFid / otherUserSession`，负责"轻量会话客户端"的构造（含设备码解析、puid/fid 解析与持久化）。
- `ChaoxingHttpClient` 改为继承 `ChaoxingHttpRequester`（登录/完整客户端），`ChaoxingHttpClientPool` 更名为 `ChaoxingHttpRequesterPool`（带 Mutex 的按手机号懒加载池）。
- 意义：代签时每个小号不再需要构造完整的 `ChaoxingHttpClient`（旧的 `loadFromOtherUserSession` 每次都要拉用户信息），`Requester` 级别即可携带各自的 `deviceCode` 完成签到；同时 `puid` 也一并持久化（`getSessionPuid()` 支持从 cookie `_uid` 兜底恢复），减少网络请求。
- 签到器基类 `ChaoxingSigner` 的 `client` 类型从 `ChaoxingHttpClient` 放宽为 `ChaoxingHttpRequester`。

### 5.3 UI 层

- `1.19.1@15` 曾在 SettingScreen 加过一个手动 `GetLocalMachineDeviceCode` 按钮（生成并保存本机设备码到 loginSession），`1.19.1@16` 即删除。**最终版 rc9 没有任何设备码的手动管理界面，绑定完全自动/随分享流程隐式完成**。
- 新增图标 `ic_tablet_smartphone_check.xml` / `ic_tablet_smartphone_x.xml`（`1.19.1@21` 添加，最终代码中未被引用，应为预留/遗留资源）。
- DataStore 调试查看器 `DataStoreTree.kt` 增强了对 deprecated 字段和 optional 字段（`hasXxx`）的展示能力——开发者模式下可以在设置页的 DataStore 树中直接看到 `deviceCode` / `isNotRandomizedDeviceCode` 的值。

---

## 6. 版本演进时间线（设备码相关）

| 时间 | 提交/版本 | 设备码相关变化 |
|---|---|---|
| 2025-03-25 | `ff6e42d`（q7，远古版本） | DataStore 顶层已有 `deviceCode` 字段 5 + `saveDeviceCode()`（随机生成、全局单值、后来废弃） |
| 2026-09-18 | `17e4397` 1.19.0@1 → **1.19.0-stable**（main 现状） | deviceCode 每次 `ChaoxingHttpClient` 实例化时随机生成，不持久化；5 个签到器携带随机码 |
| 2026-09-21 | `1.19.1@5/@6` → **1.19.1-rc1** | 尚无绑定功能（rc1 与 stable 差异主要在其他方面） |
| 2026-09-24 | `52dff38` **1.19.1@14** | **绑定功能落地**：`getLocalMachineDeviceCode`（OAID+AES，当时还有 UUID 兜底）、分享链接 `dc` 参数、`applySharedDeviceCode`、5 个签到 Screen 适配 |
| 2026-09-24 | `e102d9f` 1.19.1@15 | SettingScreen 增加手动 GetLocalMachineDeviceCode 按钮 |
| 2026-09-24 | `ef35884` 1.19.1@16 | 删除上述手动按钮；`getCachedLocalMachineDeviceCode` 缓存化 |
| 2026-09-26 | `5651f18` **1.19.1@19** | 大重构：`ChaoxingHttpRequester`/Pool；OAID 无效不再 UUID 兜底而是返回空串（自动退回随机码）；deviceCode/puid 持久化解析 |
| 2026-09-26 | `de79348` 1.19.1@21 | 新增平板/手机图标（未使用） |
| 2026-09-29 | `7d187e3` 1.19.1@26 → **1.19.1-rc9-260929** | 最终形态：绑定全自动，无手动 UI |

---

## 7. 速查问答（用户关心的三个问题）

**Q1：设备ID（deviceCode）是什么？**
客户端在签到请求 `deviceCode` 参数中提交的 Base64 设备标识（88 字符，解码后 64 字节）。它由客户端自行生成——要么由本机 **OAID 经 AES(密钥 QrCbNY@MuK1X8HGw)/ECB/PKCS5 加密** 得到（真实设备码），要么由 **双 UUID 的 SHA-256 重复拼接** 得到（随机设备码）。服务器以此识别"签到来自哪台设备"。

**Q2：有什么用？**
超星以它做设备维度风控。绑定后：① 主账号设备码跨启动/重登录稳定，符合真实用户特征；② 代签小号使用**被代签者本人手机**的设备码，服务器看到该账号始终在同一台设备签到，规避"换设备/异地代签"检测。配套的 `clientId`（服务器回执）还参与人脸签到 signToken 防伪计算。

**Q3：怎么获取/绑定？**
- **本机主账号**：全自动。登录或恢复会话时自动生成（依赖友盟 OAID，1 秒超时；拿不到 OAID 则退回随机码）并写入 DataStore `loginSession.deviceCode`。
- **给他人代签（绑定对方设备ID）**：对方在"其他用户"页分享用户数据（二维码或链接，形如 `...&dc=xxx`），你扫码/粘贴导入即可。导入后该小号会话的 `deviceCode` 被覆盖为对方设备码并标记 `isNotRandomizedDeviceCode=true`。旧版本（无 dc 参数）分享链接仍可正常导入，只是不绑定设备码。
- **查看当前值**：无正式 UI；开发者模式（isDevelopedMode）下可通过设置页的 DataStore 树查看 `loginSession/otherUsers` 下的 `deviceCode` 与 `isNotRandomizedDeviceCode` 字段。

---

## 8. 关键代码索引（均为 1.19.1-rc9，即 v1.19-apologize-dev 分支）

| 内容 | 文件:行 |
|---|---|
| AES 密钥 / OAID 有效性正则 | api/ChaoxingDeviceInfoHelper.kt:42-43 |
| RSA 公钥（设备指纹/clientId 共用） | api/ChaoxingDeviceInfoHelper.kt:39-40 |
| clientId 解密 | api/ChaoxingDeviceInfoHelper.kt:48-72 |
| 本机设备码生成（OAID+AES） | api/ChaoxingDeviceInfoHelper.kt:75-86 |
| 本机设备码缓存 | api/ChaoxingDeviceInfoHelper.kt:89-107 |
| 随机设备码 | api/ChaoxingDeviceInfoHelper.kt:108-115 |
| OAID 获取（1 秒超时） | api/ChaoxingDeviceInfoHelper.kt:116-136 |
| 设备指纹 JSON（deviceUniqueId 等） | api/ChaoxingDeviceInfoHelper.kt:138-202 |
| Requester 基类（deviceCode 属性） | api/ChaoxingHttpRequester.kt:23-31 |
| 小号设备码解析（随机+持久化） | api/ChaoxingHttpRequester.kt:92-110 |
| 主会话设备码解析（本机码） | api/ChaoxingHttpRequester.kt:303-304 |
| 登录时设备码初始化 | api/ChaoxingHttpClient.kt:276-277 |
| 旧随机生成（已废弃转发） | api/ChaoxingHttpClient.kt:190-191 |
| 分享链接 dc 参数 | api/ChaoxingOtherUserHelper.kt:102-107 |
| 绑定对方设备码 | api/ChaoxingOtherUserHelper.kt:44-61 |
| 新导入绑定设备码 | api/ChaoxingOtherUserHelper.kt:336-339 |
| 二维码/链接导入 dc 解析 | ImportOtherUserActivity.kt:100,137；screen/OtherUserScreen.kt:1940-1956 |
| 5 个签到器的 deviceCode 参数 | signer/Chaoxing{Location:74,Gesture:89,Password:91,Photo:73,QRCode:136}Signer.kt |
| clientId 用于人脸 signToken | api/ChaoxingFaceHelper.kt:103-134 |
| DataStore 设备码字段 | proto/datastore.proto:129-131,144-146 |
| clientId 实体字段 | entity/ChaoxingUserEntity.kt:32 |

（路径前缀：`app/src/main/java/org/aquamarine5/brainspark/chaoxingsignfaker/`）

---

## 9. 小结

1. **"绑定设备ID"= deviceCode 从"每次随机、不落盘"改为"持久化 + 真实设备关联 + 可跨设备转移"**。主账号绑本机 OAID 派生码，小号默认随机码，代签场景通过分享链接 `dc` 参数把被代签者的设备码绑定到小号会话。
2. 功能入口在 `1.19.1@14`（2026-09-24），成熟于 `1.19.1@19` 的网络层重构（ChaoxingHttpRequester 架构），最终随 **1.19.1-rc9-260929** 发布；main 分支（1.19.0-stable）尚未包含该功能。
3. 实现上有两处"退路"设计值得注意：OAID 获取失败（超时/全零/空）时本机码为空、自动退回随机码；旧格式分享链接（无 dc）仍可正常导入，只是不绑定。
4. 与设备ID相关的还有一套并行的"设备指纹"体系（deviceUniqueId/cdid/device_id + RSA + 服务器 clientId + 人脸 signToken），1.19.1 未改变其逻辑，只是 deviceCode 的持久化让整套设备伪装更完整自洽。
