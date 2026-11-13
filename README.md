# LocalSend iOS 未签名 IPA —— 自编译方案

给需要自己签名的场景用。产物是**未签名 ipa**，签名方式你自定。

> **已实际构建成功**（2026-11-13）
>
> | 项目 | 值 |
> |---|---|
> | 仓库 | https://github.com/hayatlia58-blip/localsend-ipa-build |
> | 源码 | `v1.18.2` |
> | 预设 | `sideload` |
> | 工具链 | Xcode 26.6 / Flutter 3.41.9 |
> | 耗时 | 12 分 41 秒 |
> | 产物 | `LocalSend-1.18.2-unsigned.ipa`，25.7 MB |
> | SHA256 | `96b5219871b38856a68c7c2aafab65e1c5ac165c357b197287b9a9c3112ff2d2` |
>
> 验证：Bundle ID `org.localsend.localsendApp`、版本 1.18.2 (64)、arm64、`cryptid=0`（未加密，可签名）、ShareExtension 已移除。
>
> **必须用 `macos-26` runner**，原因见文末「已踩过的坑」。

---

## 一、签名前必须知道的三个事实

这三条决定了你的 ipa 能不能装成功、装成功后能不能自动发现设备。都是我从仓库配置里核实过的，不是推测。

### 1. 要签的是两个二进制，不是一个

`app/ios/` 下有两个 target：

| Target | Bundle ID | 说明 |
|---|---|---|
| Runner | `org.localsend.localsendApp` | 主 App |
| ShareExtension | `org.localsend.localsendApp.ShareExtension` | 分享扩展（从其他 App 分享文件到 LocalSend） |

扩展是独立二进制，签名时需要**它自己的 App ID 和 provisioning profile**。很多代签工具只处理主 App，遇到扩展会报 `Failed to sign ...appex` 或者签完装上后主 App 直接闪退。

所以工作流提供 `sideload` 预设，**直接把扩展从产物里移除**。代价只是失去"从其他 App 分享到 LocalSend"这个入口，主功能（收发文件、文本）完全不受影响。

### 2. App Group 权限必须剥离

两个 target 的 entitlements 里都声明了：

```xml
<key>com.apple.security.application-groups</key>
<array><string>group.org.localsend.localsendApp</string></array>
```

这个 Group ID 属于原作者。你的证书如果没在开发者后台把它注册进自己的 App ID，签名时一定失败。免费个人账号、企业证书、绝大多数代签服务都拿不到。

`developers` 预设会把 entitlements 清空来绕过，但那样扩展也用不了共享容器——所以要么清空 + 保留扩展（可能运行时报错），要么按 `sideload` 预设直接删掉扩展。**推荐后者。**

### 3. 自动发现依赖一个需要 Apple 审批的权限

`Runner.entitlements` 里有：

```xml
<key>com.apple.developer.networking.multicast</key><true/>
<key>aps-environment</key><string>development</string>
```

`com.apple.developer.networking.multicast`（多播网络权限）**需要在 Apple 开发者后台单独提交申请、人工审批**，才能在 profile 里带上。LocalSend 靠它往 `224.0.0.167:53317` 发 mDNS 广播，也就是"打开就看到附近设备"。

对你的实际影响：

| 签名证书是否含 multicast | 后果 |
|---|---|
| 不含（绝大多数证书、代签服务） | 装好后**设备列表是空的**，收不到也发不出广播。必须在电脑端用「手动输入 IP」直连，传输本身正常 |
| 含（自己向 Apple 申请并通过） | 功能完整，和 App Store 版一致 |

补充一句：这个 entitlement 个人开发者账号也能申请，在 Apple 的 Multcast Networking 申请表里说明用途（局域网设备发现）即可，审批周期通常几天到几周。如果你长期用，值得申请一次。

另外 `aps-environment: development` 是推送环境声明，个人证书带不上，一并清掉最稳，不影响主功能。

---

## 二、三个预设怎么选

工作流运行时选 `profile`：

| 预设 | entitlements | ShareExtension | 适合 |
|---|---|---|---|
| **`sideload`**（默认，推荐） | 清空 | **移除** | 用个人/企业证书、代签服务。最容易签过 |
| `developers` | 清空 | 保留 | 自己有付费账号，能为扩展单独建 App ID 并配好 profile |
| `trollstore` | **原样保留** | 保留 | TrollStore（iOS 14.0–17.0）。ldid 签名不走证书，也不需要 App Group 授权，功能最完整 |

拿不准就选 `sideload`。

---

## 三、操作步骤

### 1. 建仓库

GitHub 新建一个仓库（私有也行）。不需要 fork localsend，工作流自己去拉上游源码。

### 2. 放入工作流

把 `.github/workflows/build-ipa.yml` 按原路径提交：

```
你的仓库/.github/workflows/build-ipa.yml
```

### 3. 触发构建

仓库页 → **Actions** → 左侧 **Build LocalSend unsigned IPA** → 右侧 **Run workflow**：

- `ref`：填稳定 tag，目前最新是 **v1.18.2**。填 `main` 是开发分支，有编不过的风险
- `profile`：`sideload`

等 20–40 分钟。

### 4. 下载

构建记录页面底部 **Artifacts** → `LocalSend-sideload-unsigned-ipa-v1.18.2`，里面有：

- `LocalSend-sideload-unsigned.ipa` —— 你的未签名包
- `BUILD-INFO.txt` —— 本次构建的参数和版本信息

Artifact 保留 7 天，下完存本地。

---

## 四、签名安装

未签名 ipa 必须签名才装得上。你既然有渠道，按你的方式签。几个容易踩的点：

**签完检查清单**

1. 用 `codesign -dv --entitlements - Payload/Runner.app` 确认签名生效
2. 如果产物里有 `Payload/Runner.app/PlugIns/`，说明扩展还在，需要一并签
3. 装上后首次打开若提示"不受信任的开发者"：
   **设置 → 通用 → VPN 与设备管理 → 开发者应用 → 信任**

**首次启动必做**

LocalSend 会请求「本地网络」权限，**必须点允许**。路径：
**设置 → 隐私与安全性 → 本地网络 → 打开 LocalSend**

不给这个权限，即使有多播 entitlement 也收不到任何连接。

---

## 五、没有自动发现时怎么用

`multicast` 被剥离的情况下，设备列表为空是正常现象，不是装坏了。

### 电脑 → iPhone（最稳）

1. iPhone 打开 LocalSend，停在"接收"页，记下本机 IP（接收页右上角 `⋯` 或设置里，形如 `192.168.1.37`）
2. 电脑端 LocalSend 点"发送" → "附近设备"下方找 **手动输入 / 添加设备**
3. 填这个 IP，端口默认 53317
4. iPhone 弹出确认，点接受

### iPhone → 电脑

在 iPhone 端的手动输入入口填电脑的局域网 IP。部分版本 iOS 端入口较深，找不到就用反方向，或临时跑下面那套网页方案。

### 环境检查

- 两台设备同一 Wi-Fi，避开访客网络和 AP 隔离
- Windows 防火墙放行 TCP + UDP 53317
- iPhone 关掉 VPN，或让 VPN 允许局域网
- IP 会随 DHCP 变化，连不上就重新看一次

---

## 六、编译失败排查

| 现象 | 原因 | 处理 |
|---|---|---|
| `Swift Compiler Error: Value of type 'NWPath' has no member 'isUltraConstrained'` | runner 的 Xcode 太旧，缺 iOS 26 SDK 的新 API | `runs-on: macos-26`（**已踩过，这是最初失败的原因**） |
| `pub get` 报 SDK 版本不符 | Flutter 版本与 runner 不匹配 | 工作流已按 `.fvmrc` 自动取版本；仍失败就把 `flutter-version-file` 换成硬编码版本号 |
| `xcodebuild` 报 Xcode 不兼容 | Flutter 与默认 Xcode 版本错位 | 在 `Toolchain versions` 步骤看实际版本，必要时用 `sudo xcode-select -s` 切 Xcode |
| `Runner.app` 不存在 | 更早阶段就挂了 | 看 `Build iOS app without code signing` 的完整日志 |
| 签名报 `.appex` 失败 | 扩展还在，缺独立 profile | 换 `sideload` 预设重新构建 |
| 签名报 `application-groups` 错误 | entitlements 没清干净 | 确认 `profile` 不是 `trollstore` |
| 装上闪退 | 用户装的是带 DRM 的 App Store 包，或 entitlements 不匹配 | 用自己编的 ipa，别用 `ipatool` 抓的包 |

> 顺便提醒：从 App Store 抓下来的 ipa 是 FairPlay 加密的（`cryptid=1`），重签后必定闪退。只有自己编译的无 DRM 二进制才能用。

---

## 七、附：完全不想折腾时的备用方案

同目录外的 `../lan-transfer/` 里有一个只用 Python 标准库写的局域网传输页。电脑上双击启动，手机 Safari 打开地址就能互传文件和文本，不需要装任何 App、不需要 Apple ID。

功能比不上 LocalSend（没有多播发现、界面更简单），但胜在零成本、立刻可用，适合临时救急或验证网络是否通。

---

## 八、已踩过的坑（真实记录）

### 1. macOS runner 版本不对，Xcode 编译直接失败

第一版用 `macos-15`，跑到编译阶段报：

```
Swift Compiler Error (Xcode): Value of type 'NWPath' has no member 'isUltraConstrained'
```

`NWPath.isUltraConstrained` 是 iOS 26 的 Network framework 新增 API，`macos-15` 自带 Xcode 16（iOS 18 SDK），没有这个符号。换成 `macos-26`（自带 Xcode 26.6）后一次通过，全程 12 分 41 秒。

### 2. workflow 里不能用顶格的 heredoc

`run: |` 是 YAML block scalar，内容必须保持缩进。写 `cat > file <<EOF` 时如果 heredoc 正文顶格，会提前终止 scalar，整个 workflow 解析失败。改用 `{ echo ...; } > file` 的形式。

### 3. 公开仓库跑 macOS runner 才免费

私有仓库的 macOS runner 按 **10 倍**消耗 Action 分钟数。免费账号每月 2000 分钟，一次 13 分钟的构建约吃掉 130 分钟额度。公开仓库完全免费，所以本仓库设为 public —— 里面只有构建配置，没有个人信息。

### 4. 注意 Flutter 版本随 tag 变化

`v1.18.2` 的 `.fvmrc` 是 **3.41.9**，而 `main` 上是 3.47.9。工作流按 checkout 下来的 `.fvmrc` 取版本，所以用 tag 构建时 Flutter 版本和 main 不同，这是正常的。

---

## 九、合规

LocalSend 是 Apache-2.0，编译、修改、自用、再分发都允许。再分发保留版权声明和 LICENSE。不要用 "LocalSend" 名称和图标做自己的发行版（商标不在授权范围内）。
