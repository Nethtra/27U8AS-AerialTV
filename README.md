# Aerial 电视桌面 · 从零部署指南

> 面向：没碰过 Android 调试、也没写过 Android 代码的人
> 对照版本：官方文档 **Aerial v1.52**（2026-10-26 整理）｜实机验证 **v1.69**（2026-10-29）
> 适用环境：Windows 10/11 电脑 + 一台国产 Android 电视（小米、TCL、创维、索尼、海信、盒子、投影仪等）
> 阅读顺序：第一、二章别跳过，它们决定了你后面会不会白折腾。
> 文档状态：已含 **雷鸟 27U8A-S（Google TV 侧）全程实机验证**，见 6.10；附录 **8.5** 附自编译 TV 快捷方式 App 的完整工具链，**8.6** 是第二个实例（无障碍入口）+ 预装应用精简。

---

## 目录

- [一、先看懂这个项目（5 分钟，能省 1 小时）](#一先看懂这个项目5-分钟能省-1-小时)
- [二、开装前的三个自检](#二开装前的三个自检)
- [三、部署流程（9 步）](#三部署流程9-步)
- [四、验收：怎么确认"真的成功了"](#四验收怎么确认真的成功了)
- [五、首次启动后的 5 项设置](#五首次启动后的-5-项设置)
- [六、常见问题速查](#六常见问题速查)
- [七、日常使用、升级与还原](#七日常使用升级与还原)
- [八、附录](#八附录)

---

## 一、先看懂这个项目（5 分钟，能省 1 小时）

### 1.1 它到底是什么

一句话：**Aerial 是一个替换电视自带桌面的 Android 应用（Launcher）**，参照 Apple TV 的 tvOS 做的：没有广告位、没有推荐流，只有你的 App、壁纸和航拍屏保。

这决定了"部署"这两个字的含义——**它不是服务端项目，没有 Docker、没有 Node.js、没有需要编译的前端**。所谓部署，就是把一个 APK 装进电视，再给它几个普通 App 拿不到的权限，让它能抢回主页键。

如果你原本以为要"架一台服务器""配 Nginx""改 config.yaml"，那可以先松口气：**全程只有一个依赖（adb），全程不碰电视的系统分区，不 root，随时可还原。**

### 1.2 技术栈与关键参数

| 项目 | 内容 |
|---|---|
| 语言 / UI | Kotlin + Jetpack Compose（全自绘，没用 androidx.tv 和 Material） |
| 包名 | `app.orchard.tv`（历史名 Orchard，一直没改，改包名会丢用户数据和授权） |
| 最低系统 | **Android 5.0（API 21）**；targetSdk 34，compileSdk 36 |
| 构建 | Gradle Kotlin DSL，单模块（`app/`） |
| 仓库规模 | 约 7.6 MB 源码，2026-10-20 建仓，6 天 5301 star、46 个版本 |
| 许可 | MIT（字体用 SIL OFL；航拍影片版权归 Apple，实时从 Apple 服务器串流，不打包进 APK） |

> 注意：`minSdk 21` 只代表"能装"。Android 5/6 的老电视体验会打折，官方支持表里对老机型普遍标注了"按键后要等几秒"。

### 1.3 为什么安装步骤这么"麻烦"——接管机制

这是全文最该理解的一节。理解了它，后面每条命令你都知道在干什么，出错也知道该查哪。

国产电视的固件普遍只认自家桌面，而且**不许第三方 App 声明成 HOME**（TCL 的 `LauncherStrategy` 就是典型）。所以 Aerial 用了两层接管：

```
第 1 层：设为系统默认桌面（standard 版才有的 HOME intent-filter）
        原生 Android TV / 小米部分机型可用；TCL 等固件直接无视

第 2 层：无障碍服务"桌面守护"（HomeGuardService）—— 真正的主力
        · 原桌面一露头，立刻切回 Aerial
        · 主页键：不是默认桌面时把按键吃掉，直接开 Aerial
        · 亮屏 / 开机 / 解锁后等 1.5 秒，前台还是原桌面就抢回来
        · 只判断"哪个窗口在前台"，不读屏幕内容（这点对隐私很重要）

第 3 层：被系统杀了怎么复活
        · 小米待机会 killAllThirdApp → 系统移除无障碍授权
        · 通知监听（RevivalListener）负责让系统把进程拉回来
        · 低内存 Android 9 机型（如小米 MFTP0）系统不许通知监听，
          改用"同步适配器"（RevivalSync），30–60 秒后被拉起
        · BootReceiver 兜底真重启
```

由此推出三个必然结论，也是新手最容易困惑的地方：

1. **必须给 `WRITE_SECURE_SETTINGS`**——只有它能改"无障碍服务开关"这种系统设置，而这条权限只能靠 adb 授予，App 自己申请不到。
2. **必须开无障碍 + 通知监听**——否则守护活不下来，按主页键就回原桌面。
3. **小米的 `resolve-activity` 显示的还是 PatchWall 是正常的**——接管靠守护，不靠改系统默认桌面。看到这个别以为装失败了。

### 1.4 两个 APK 的区别（决定你该下哪个）

**差异只有 3 行 XML。** 两个包代码完全相同、包名相同（`app.orchard.tv`）、签名相同，唯一的区别是 `standard` 变体多合并了一个 manifest，给主 Activity 加了两个 intent-filter：

```xml
<!-- app/src/standard/AndroidManifest.xml（只有 standard 有） -->
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.HOME" />     <!-- 声明"我是桌面" -->
    <category android:name="android.intent.category.DEFAULT" />
</intent-filter>
<intent-filter>
    <category android:name="android.intent.category.HOME.CC" />  <!-- 创维/酷开的私有桌面类别 -->
</intent-filter>
```

`guard` 变体只做了一件事：`versionNameSuffix = "-guard"`，**什么声明都不加**。所以它只是个普通 App，从不声称自己是桌面。

**这一处差异，连锁出四个完全不同的后果**

| | `standard` | `guard` |
|---|---|---|
| 能否装到固件封锁的电视 | 会被拦（TCL `LauncherStrategy` 直接拒装） | ✅ 能装 |
| 能否设为系统默认桌面 | ✅ 可以（`role add-role-holder` 才可能成功） | ❌ 永远不行，`resolve-activity` 永远不会是它 |
| 主页键靠什么接管 | 系统自己就是桌面，**不需要守护** | **只能靠无障碍守护**，一被关就掉 |
| 稳定性 | 高（原生 ATV 上最干净） | 中（依赖守护进程活着） |
| 版本号显示 | `1.52` | `1.52-guard` |

**为什么不能合成一个包**：因为"声明 HOME"这件事在安装阶段就会被固件检查。TCL 一类固件看到要装一个声明了 HOME 的第三方包，直接拒装——所以必须准备一个"不声明 HOME"的版本，先混进去，再靠无障碍把主页键抢回来。这是被固件逼出来的双版本。

**判断口诀**

```
先试 standard → 装不上（固件拒装桌面类 App）→ 换 guard
装 guard 之后 → 第 6 步"设为默认桌面"直接跳过，验收只看第 3、6 条
```

**三个容易踩的坑**

1. **装 guard 版后，验收第 2 条永远不合格**。`resolve-activity` 永远不会显示 `app.orchard.tv`，因为它在系统眼里根本不是桌面。这不是失败，别反复重装。
2. **两者不能共存**（同包名），但可以互相覆盖：`adb install -r` 换版本时，桌面布局、壁纸、设置全部保留。
3. **别以为 guard 更"高级"**。它只是退而求其次的方案——真正被系统认成桌面的 standard 才更稳，因为不用跟固件抢按键，也不会被系统清理杀掉。能用 standard 就别用 guard。

> 你的情况：雷鸟 Android 14 手动装 → 只能用 guard（standard 会被拒）；走"停用安装器"那条绕行路线 → **必须用 standard**，因为你正是要它那两行 HOME 声明，才能被设成默认桌面。见 [6.7 雷鸟专章](#67-雷鸟ffalcon专章按上下左右没反应怎么办)。

### 1.5 成熟度与边界（先说清楚，免得期待错位）

**可以放心的地方**

- MIT 开源、作者持续日更（6 天 46 个版本），有完整机型笔记（`docs/devices/`）、Issue 模板、只读诊断脚本 `report.sh`。
- 不 root、不停用系统 App、不删文件，官方提供完整还原命令。
- 只换桌面层：会员、账号、小爱同学、有线电视、原来的桌面 App 全部保留（原桌面会作为普通 App 留在列表里）。

**要有心理准备的地方**

- **开机广告去不掉**。那是固件在 Aerial 启动之前播的，任何第三方桌面都做不到。
- **机型差异极大**。同一品牌不同固件可能是 ✅ 也可能 ❌，装之前务必查官方支持表。
- **261 个 open issue**，说明适配还在快速补。遇到怪问题先去 Issues 搜机型名。
- **不是官方产品**，与 Apple、小米、TCL 无关。出问题只能自己还原。

---

## 二、开装前的三个自检

三个全过再往下走。任何一条不过，后面 90% 的时间都会浪费在排查上。

### 自检 1：系统与品牌

| 检查项 | 怎么查 | 合格标准 |
|---|---|---|
| 系统版本 | 设置 → 关于 → 系统版本 / Android 版本 | **Android 5.0 及以上** |
| 品牌 | 电视背面铭牌或设置 → 关于 | 不是 LG webOS、三星 Tizen、华为鸿蒙智慧屏 |
| 是否开放 ADB | 见下一节 | 能打开 |

❌ **确定不支持**：LG（webOS）、三星（Tizen）、华为智慧屏（鸿蒙）、Android 4.4 及更老。
⚠️ **能装但接管不了主页键**：雷鸟 55S595C Ultra / 鹤 6 2025（只有"无线调试"，拿不到授权）、创维 7T641 / 75A6E、OPPO K9（要改包名）。

### 自检 2：能不能打开 ADB

打开方式见 [第 2 步](#第-2-步打开电视的-adb-调试)。**雷鸟用户请直接看 [6.7 雷鸟专章](#67-雷鸟ffalcon专章按上下左右没反应怎么办)**——2025 款的入口和 TCL 不一样：可能藏在「无线调试」里（有 6 位配对码），也可能完全没有、只能走有线。有个笨但有效的判断法：**在设置里能不能找到"开发者选项 / ADB 调试 / USB 调试 / 无线调试"这一项**。找不到就按第 2 步的品牌路径去"敲"出来；实在敲不出来（极米 H3、老乐视、荣耀智慧屏、老康佳看不到版本号），就是没戏，别硬试。

### 自检 3：网络与 IP

- 电脑和电视**连同一个路由器、同一个网段**（不能一个连路由器 WiFi、一个连光猫 WiFi）。
- 关掉路由器的"AP 隔离 / 访客网络隔离"，否则同一 WiFi 下也互相 ping 不通。
- **记下电视 IP**：设置 → 网络 → 已连接的网络（OPPO S1 在这一页最下面）。
- 建议在路由器里给电视做 **DHCP 静态绑定**。否则电视重启后 IP 一变，你所有命令都要重来——这是新手最常见的"昨天还好好的今天连不上"。

---

## 三、部署流程（9 步）

整体路线图：

```
[电脑] 装 adb  →  连电视  →  推 APK  →  授权限  →  设默认桌面  →  开守护  →  屏保  →  启动  →  验收
[电视] 开ADB调试 ↗                                                              ↖ 允许调试弹窗
```

每一步都按 **前置条件 → 操作 → 预期结果 → 不对怎么办** 四段写。命令统一在 **PowerShell** 里执行（不要用 cmd.exe，引号转义规则不同）。

---

### 第 1 步：在电脑上装 ADB

**前置条件**：电脑能上网；有解压权限。

**这一步在干什么**：装 Google 官方的 `platform-tools`，里面的 `adb.exe` 是你和电视之间唯一的桥梁。

**操作**（整段复制进 PowerShell）：

```powershell
cd $env:USERPROFILE
curl.exe -fL -o platform-tools.zip https://dl.google.com/android/repository/platform-tools-latest-windows.zip
Expand-Archive -Force platform-tools.zip .
$env:Path = "$env:USERPROFILE\platform-tools;$env:Path"
adb version
```

**预期结果**：屏幕打印类似

```
Android Debug Bridge version 1.0.41
Version 36.0.0-xxxxxxx
```

**不对怎么办**

| 现象 | 原因 | 处理 |
|---|---|---|
| `curl.exe : 无法将...识别为` | 系统太老没自带 curl（Win10 1803 以下） | 浏览器手动下载上面的 zip，右键"全部解压缩"到 `C:\Users\你的用户名\` |
| 下载卡住 / 报 SSL 错 | 代理或网络 | 关掉代理再试；`dl.google.com` 国内一般可直连 |
| `adb version` 报"不是内部或外部命令" | PATH 只在**当前窗口**生效 | 见下方"永久生效" |
| 杀软弹窗拦截 `adb.exe` | 正常误报 | 允许；`platform-tools` 是 Google 官方签名 |

**让 adb 永久可用**（强烈建议，不然每开一个新窗口都要重设 PATH）：

```powershell
[Environment]::SetEnvironmentVariable('Path', [Environment]::GetEnvironmentVariable('Path','User') + ';' + "$env:USERPROFILE\platform-tools", 'User')
```

关掉 PowerShell 重开，`adb version` 能出版本号就成功。

> ⚠️ **绝对不要用 `setx PATH`**。它会把超过 1024 字符的 PATH **静默截断**，轻则其他命令行工具全失效，重则系统环境变量被毁。上面那行 `SetEnvironmentVariable` 是安全写法。

---

### 第 2 步：打开电视的 ADB 调试

**前置条件**：电视已联网；遥控器在手边。

**这一步在干什么**：让电视允许局域网内的电脑连进来调试。这一步全在电视上操作，电脑不参与。

**操作**：对照你的品牌。

| 电视品牌 | 操作路径 |
|---|---|
| **小米 / Redmi** | 设置 → 关于 → 连续按「产品型号」直到提示进入开发者模式 → 回设置 → 账号与安全 → 打开「**ADB 调试**」和「**安装未知来源的应用**」 |
| **TCL / 雷鸟** | 设置 → 关于 → 本机信息 → 用遥控器**上、下、左、右**（间隔不超过 1 秒）→ 底部出现「更多」→ 打开「ADB」 |
| **OPPO（含 S1）** | 设置 → 系统 → 关于本机 → 设备型号 → 按**上、上、右、右**再确认 → 设置 → 系统 → 开发者选项 → 打开「USB 调试」 |
| **海尔 / 模卡 MOOKA** | 按**机身**上的信号源键 → 弹出菜单后用遥控器按**左、左、上、右** → 进入「海尔工厂菜单」→ 把「ADB 调试开关」设为开。**只改这一项，千万别碰"工厂复位"** |
| **原生 Android TV / Google TV / 索尼 / 海信** | 设置 → 设备偏好设置 → 关于 → 连按「**内部版本号**」（不是"Android 版本"，按错了会出彩蛋，按返回退出）7 次 → 开发者选项 → 打开「USB 调试」（也可能叫「网络调试」） |

**同时记下电视 IP**：设置 → 网络 → 已连接的网络 → 点进去看 IP 地址，形如 `192.168.1.20`。

**预期结果**：电视上 ADB 相关开关处于"开"的状态；你手里有一个 IP。

**不对怎么办**：找不到「产品型号」/「本机信息」这类入口 → 说明该机型固件把入口藏得很深或干脆没有，属自检 2 不通过，停在这里。

---

### 第 3 步：连接电视

**前置条件**：第 1 步 `adb version` 有输出；第 2 步 ADB 已开、IP 已知；电脑和电视同网段。

**操作**（把 `192.168.1.20` 换成你的 IP）：

```powershell
adb connect 192.168.1.20:5555
adb devices
```

**预期结果**

- 电视屏幕弹出「**允许 USB 调试吗？**」，用遥控器勾选「**始终允许**」再选「确定」。
- `adb devices` 输出两行，关键看状态列是 `device`：

```
List of devices attached
192.168.1.20:5555       device
```

**不对怎么办**

| `adb devices` 显示 | 含义 | 处理 |
|---|---|---|
| 什么都没有 / 只有 `List of devices attached` | 没连上 | 电视 ADB 没开或 IP 不对，重做第 2 步 |
| `unauthorized` | 电视上的授权弹窗没点 | 用遥控器点「允许」并勾「始终允许」，再跑一次 `adb devices` |
| `offline` | 连接建立但握手失败 | `adb disconnect` 后再 `adb connect`；仍不行就重启电视的 ADB 开关 |
| `failed to connect` / `Connection refused` / 超时 | 端口不通 | ① 电视 ADB 真没开 ② IP 变了 ③ 不在同一网段 ④ AP 隔离 |

排查网络最快的三条命令（在 PowerShell 里）：

```powershell
ping 192.168.1.20
Test-NetConnection 192.168.1.20 -Port 5555
adb kill-server; adb start-server
```

`ping` 通但 `Test-NetConnection` 的 `TcpTestSucceeded` 是 `False` → 网络通但 ADB 端口没开，回第 2 步。`ping` 就不通 → 网络问题，别在 adb 上浪费时间。

> 补充：ADB 无线调试默认端口是 **5555**。Android 11+ 的"无线调试"（带配对码）用的是随机端口，走的是另一套 pairing 流程，Aerial 的官方脚本不覆盖这条路——雷鸟那类只有"无线调试"的机型就是卡在这里。

---

### 第 4 步：下载并安装 APK

**前置条件**：`adb devices` 里状态是 `device`。

**这一步在干什么**：把 2.9 MB 的安装包推进电视并安装。**先装 standard 版**。

**操作**

```powershell
cd $env:USERPROFILE
curl.exe -fL -o aerial-standard.apk https://aerial.qust.me/download/aerial-standard.apk
adb install -r aerial-standard.apk
```

> `-r` = 覆盖安装并保留数据（升级时也用它）。
> 官网下载失败就用 GitHub 直链：`https://github.com/pengchujin/aerial-tv/releases/latest/download/aerial-standard.apk`

**预期结果**：最后一行是

```
Success
```

**不对怎么办**：看报错里的错误码，这一步的报错最有特征，基本能一码定位。

| 错误码 | 含义 | 处理 |
|---|---|---|
| `INSTALL_FAILED_VERIFICATION_FAILURE` | **TCL**：固件拒绝一切 adb 安装 | 走下面的「TCL 专用通道」 |
| `INSTALL_FAILED_USER_RESTRICTED` | 固件在拦 adb 安装（创维常见） | 见下方「创维」 |
| `INSTALL_FAILED_NO_MATCHING_ABIS` | 架构不支持（极罕见） | 放弃 |
| `INSTALL_FAILED_OLDER_SDK` | 系统低于 Android 5.0 | 放弃，机型不支持 |
| `INSTALL_PARSE_FAILED_*` | 下载文件损坏 | 删掉重下，比对大小约 2.94 MB |
| 提示"不允许安装桌面类 App" | 固件禁止第三方桌面 | 改用 guard 版：把文件名和 URL 里的 `standard` 换成 `guard` |

**创维的特殊处理**（`INSTALL_FAILED_USER_RESTRICTED`）：

```powershell
adb shell settings get global verifier_verify_adb_installs   # 先记下原值！
adb shell settings put global verifier_verify_adb_installs 0
adb install -r aerial-standard.apk
```

装完**把原值写回去**（原值是 `null` 就用 `adb shell settings delete global verifier_verify_adb_installs`）。别留着 `0`，那是把安装校验永久关了。

**TCL 专用通道**（adb 装不上，只能走电视自带的"多屏互动"）：
需要电脑上有 Python 3（`python --version` 能出版本号；没有就先装）。用 **guard 版**。

```powershell
mkdir aerial-tcl; cd aerial-tcl
curl.exe -fL -o Aerial.apk https://aerial.qust.me/download/aerial-guard.apk
curl.exe -fL -o tcast-push.py https://aerial.qust.me/tools/tcast-push.py
adb reverse tcp:8765 tcp:8765
Start-Process python -ArgumentList '-m','http.server','8765','--bind','127.0.0.1' -WindowStyle Hidden
python tcast-push.py <IP> http://127.0.0.1:8765/Aerial.apk app.orchard.tv Aerial 0 180
```

跑到最后一条时，**看电视右上角**会弹「多屏互动：正在发起应用推送」→ 用遥控器**先按「右」选中「同意」，再按确认**。脚本打印装好后，收尾：

```powershell
Get-Process python | Stop-Process     # 关掉临时 HTTP 服务
adb reverse --remove tcp:8765
```

然后回到第 5 步继续（注意：后面"设为默认桌面"要跳过）。

---

### 第 5 步：授予权限（配置的核心）

**前置条件**：第 4 步输出 `Success`。

**这一步在干什么**：把 11 条"只有 adb 才能给"的权限/开关打开。这就是 1.3 节说的第 2、3 层接管所需的全部前提。**跳过这一步，Aerial 能打开但按主页键会回原桌面。**

**操作**（整段粘贴，逐条执行）：

```powershell
adb shell pm grant app.orchard.tv android.permission.ACCESS_FINE_LOCATION
adb shell pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW allow
adb shell appops set app.orchard.tv WRITE_SETTINGS allow
adb shell appops set app.orchard.tv REQUEST_INSTALL_PACKAGES allow
adb shell appops set app.orchard.tv GET_USAGE_STATS allow
adb shell appops set app.orchard.tv android:get_usage_stats allow
adb shell cmd notification allow_listener app.orchard.tv/app.orchard.tv.guard.RevivalListener
adb shell appops set app.orchard.tv APP_AUTO_START allow
adb shell appops set app.orchard.tv APP_ASSOC_START allow
adb shell am start -W -n app.orchard.tv/.MainActivity
```

**这 11 条分别在干什么**（知道原因，报错时才不慌）

| 命令 | 作用 | 少了会怎样 |
|---|---|---|
| `ACCESS_FINE_LOCATION` | 天气按 IP 定位取城市 | 天气空白 |
| `WRITE_SECURE_SETTINGS` | 改无障碍开关、写屏保设置 | **守护无法自启，主页键失效** |
| `SYSTEM_ALERT_WINDOW` | 悬浮窗（启动/切换时的覆盖层） | 切换时闪一下原桌面 |
| `WRITE_SETTINGS` | 改系统设置（亮度等） | 控制中心亮度调不动 |
| `REQUEST_INSTALL_PACKAGES` | 应用内升级（下载 APK 交给系统安装器） | 无法在电视上直接升级 |
| `GET_USAGE_STATS` ×2 | 老小米 Android 6 的桌面不报窗口变化，靠用量统计判断 | 老小米接管失效 |
| `allow_listener` | 通知监听，被系统杀了之后负责复活 | 待机唤醒后回不去 |
| `APP_AUTO_START` / `APP_ASSOC_START` | 自启动白名单（**只有 TCL 认**） | TCL 上开机不自动进入 |

**预期结果**

- 大部分命令**静默成功**（没有任何输出）。
- 最后两条 `APP_AUTO_START` / `APP_ASSOC_START` 在非 TCL 电视上报 `Error: ...` 是**正常的**，忽略。
- `am start` 有输出，且电视上 Aerial 桌面已经出现。

**成功的判据不是"每条都没报错"，而是"电视上看到 Aerial 首页"**。个别命令报错可以先往下走，最后用第四章的验收来定生死。

**接着让系统把它编译成机器码**（可选但建议，能明显顺一点）：

```powershell
adb shell cmd package compile -m speed -f app.orchard.tv
```

这条要跑几十秒，输出 `Success` 后结束。前提是 Aerial 至少启动过一次。

---

### 第 6 步：设为默认桌面

**前置条件**：Aerial 已在电视上启动过。

**操作**

```powershell
adb shell cmd role add-role-holder --user 0 android.app.role.HOME app.orchard.tv
```

**预期结果**：命令静默成功（无输出）。之后按主页键直接回 Aerial。

**不对怎么办**

- 报错 `Error: ...`（老系统没有 `cmd role`）→ 换备用写法：
  ```powershell
  adb shell cmd package set-home-activity --user 0 app.orchard.tv/.MainActivity
  ```
- 两条都报错、或跑完毫无变化 → **正常现象，不要纠缠**。小米、TCL 等固件会无视这一步，真正生效的是第 7 步的守护。继续往下。
- **装的是 guard 版 → 直接跳过本步。**

**海信 VIDAA 额外一步**（其他品牌跳过）：先查有没有厂商专用属性

```powershell
adb shell getprop persist.vt.use_home_pkg
```

- 有输出（例如 `com.jamdeo.tv.vod`）且不是 `app.orchard.tv` → **记下这个原值**，然后：
  ```powershell
  adb shell settings put global aerial_vt_home_pkg <原值>
  adb shell setprop persist.vt.use_home_pkg app.orchard.tv
  ```
- 没有输出 → 跳过。

> 为什么要先存原值？因为还原时要把 `persist.vt.use_home_pkg` 写回原来的包名，否则你的 VIDAA 桌面就永久变成 Aerial 了。

---

### 第 7 步：启用桌面守护（最关键的一步）

**前置条件**：第 5 步已授 `WRITE_SECURE_SETTINGS`；Aerial 启动过。

**这一步在干什么**：确认无障碍里真的有 Aerial 的守护服务在跑。**这是"按主页键回 Aerial"的唯一保证。**

**操作**

```powershell
adb shell settings get secure enabled_accessibility_services
```

**预期结果**：输出里包含

```
app.orchard.tv/app.orchard.tv.guard.HomeGuardService
```

有这一串，本步就过了，跳到第 8 步。

**没有怎么办（手动补上）**：注意——**必须保留原来的内容**，用英文冒号连接，不要整个覆盖。

```powershell
adb shell "settings put secure enabled_accessibility_services '<原来的值>:app.orchard.tv/app.orchard.tv.guard.HomeGuardService'"
adb shell settings put secure accessibility_enabled 1
```

- 原来的值是 `null` 或空 → 只写 `app.orchard.tv/app.orchard.tv.guard.HomeGuardService`。
- 原来的值里含 `$` → 在 PowerShell 里要写成 `` `$ `` 转义。
- 有些固件上 `settings` 命令会**卡住不返回**（腾讯极光盒子 6 实测等 120 秒后断开）→ 直接跳过，改用下面这条看：
  ```powershell
  adb shell dumpsys accessibility
  ```
  在 `Enabled services` 里找有没有 Aerial。拿到授权后，Aerial 启动时会自己把守护打开，不必强行手改。

**预期结果（补充）**：电视上 Aerial 首页正常显示，按主页键不再回原桌面。

---

### 第 8 步：屏保与"保持 ADB 常开"

**前置条件**：前七步完成。

**这一步在干什么**：两个可选但很值的开关。

**8a. 接管系统屏保**（让航拍屏保在电视自己的屏保时机也能播）

先判断这台电视是否启用了系统屏保：

```powershell
adb shell settings get secure screensaver_enabled
```

- 输出 `1` → 继续执行下面两条：
  ```powershell
  adb shell settings put secure screensaver_components app.orchard.tv/app.orchard.tv.dream.AerialDream
  adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez own_screensaver true
  ```
- 输出 `0` 或 `null` → 跳过。这台电视不用系统屏保（小米就属这类，它走"桌面内屏保"，靠 Aerial 自己空闲 5 分钟后触发）。

**8b. 保持 ADB 常开**（建议开）

```powershell
adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez keep_adb true
```

**为什么建议开**：小米待机唤醒后会自己把 ADB 关掉，导致下次升级连不上，还得重新去设置里翻半天。开了这项，Aerial 会把 ADB 重新打开。

**预期结果**：`am broadcast` 输出 `Broadcast completed: result=0` 或类似成功提示。

---

### 第 9 步：首次启动与验收

**前置条件**：全部前置步骤完成。

**操作**

```powershell
adb shell am start -W -n app.orchard.tv/.MainActivity
```

**预期结果**：输出里包含

```
Status: ok
Activity: app.orchard.tv/.MainActivity
```

这一步只是"启动"，**不代表装好了**。真正的判定在下一章。

---

## 四、验收：怎么确认"真的成功了"

这是全文唯一不能跳的一章。**没跑完这 6 条，就不要说"装好了"。**

**操作**（逐条跑，把原始输出留着）

```powershell
adb shell "dumpsys package app.orchard.tv | grep versionName"
adb shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"
adb shell settings get secure enabled_accessibility_services
adb shell "dumpsys accessibility | grep -c -E 'label=(Aerial|app.orchard.tv)'"
adb shell input keyevent KEYCODE_HOME
adb shell "sleep 3; dumpsys window | grep mCurrentFocus"
```

**怎么读这 6 条**

| # | 命令 | 合格标准 | 不合格说明 |
|---|---|---|---|
| 1 | `versionName` | 有版本号（如 `1.52`） | 没装上，回第 4 步 |
| 2 | `resolve-activity` | 是 `app.orchard.tv` **更好**；是 `com.mitv.tvhome`（小米）或 TCL 自家桌面**也正常** | 报错说明老系统没 `cmd`，改用 `pm resolve-activity` |
| 3 | `enabled_accessibility_services` | 含 `...guard.HomeGuardService` | 守护没开，回第 7 步 |
| 4 | `dumpsys accessibility \| grep -c` | 数字 **> 0** | 守护没在跑，见第六章 |
| 5 | `input keyevent KEYCODE_HOME` | 电视上切到 Aerial 首页 | — |
| 6 | `mCurrentFocus` | **包含 `app.orchard.tv`** | 这就是失败，见下 |

**第 6 条是唯一的最终判据。** 输出形如：

```
mCurrentFocus=Window{2f3a1c u0 app.orchard.tv/app.orchard.tv.MainActivity}
```

看到 `app.orchard.tv` = 真的成功了。

**第 6 条不是 `app.orchard.tv` 时按顺序试**

1. **等 5 秒再跑一次第 5、6 条**。有的电视守护启动慢。
2. **用实体遥控器按主页键**，再只跑第 6 条。`input keyevent` 模拟的按键不经过守护的按键过滤，索尼 Android 9 上会与实体键结果不同，此时以遥控器为准。
3. 还是不行 → 看第六章「装上但主页键回原桌面」。

**最后一步：重启复验**（确认开机直达）

```powershell
adb reboot
```

等 1–2 分钟电视开机完成，重新连接并重复第 5、6 条：

```powershell
adb connect 192.168.1.20:5555
adb shell input keyevent KEYCODE_HOME
adb shell "sleep 3; dumpsys window | grep mCurrentFocus"
```

开机直接进 Aerial、IP 没变（若变了说明没做静态绑定），整个部署才算闭环。

---

## 五、首次启动后的 5 项设置

这些不是"必须"，但决定了它好不好用。全部在电视上用遥控器操作。

| 想做的事 | 在哪 |
|---|---|
| 换壁纸（必应每日 / 内置 14 张 / 手机传图 / 航拍动态） | 控制中心（第一排按**上**键到右上角）→ 个性化 |
| 换屏保、调画质、设开始时间 | 个性化 → 屏幕保护程序。4K / 杜比视界要电视解码器支持，不支持会自动回到 1080p |
| 用手机整理桌面 | 个性化 → 手机扫码 → 手机浏览器里拖动排序、改名、换图标、建文件夹 |
| 加 HDMI 信号源并换图标（Apple TV / PS5 / Switch / Xbox） | 长按 OK 进菜单 → 固定到主屏幕 → 选设备类型 |
| 改单排/双排、时钟样式、按键音 | 个性化 → 个性化面板 |

**家长密码**：个性化里可以设。忘了不用慌，一条命令关掉：

```powershell
adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez clear_pin true
```

---

## 六、常见问题速查

### 6.1 连不上（第 3 步）

| 症状 | 处理顺序 |
|---|---|
| `adb devices` 空 | ① 电视 ADB 开关是否真打开 ② IP 是否变了（`ping` 试） ③ 是否同网段 ④ `adb kill-server; adb start-server` |
| `unauthorized` | 电视上的弹窗必须用**遥控器**点"允许"并勾"始终允许"。弹窗几秒就消失，重跑 `adb devices` 会再弹 |
| `Connection refused` | 端口 5555 没在听：ADB 没开或固件只给"无线调试"（随机端口） |
| 昨天能连今天不能 | 99% 是 IP 变了。去路由器做 DHCP 静态绑定 |

### 6.2 装不上（第 4 步）

见第 4 步的错误码表。三条高频：

- **TCL**：`INSTALL_FAILED_VERIFICATION_FAILURE` → 走多屏互动 + guard 版。
- **创维**：`INSTALL_FAILED_USER_RESTRICTED` → 临时关 `verifier_verify_adb_installs`，装完写回。
- **提示不许装桌面类 App** → 换 `guard` 版。

### 6.3 装上了，但主页键回原桌面

按这个顺序排查，**别跳步**：

1. **先等 5 秒**再按一次主页键。守护有 1.5 秒的"等前台稳定"延迟。
2. 查守护有没有开：
   ```powershell
   adb shell settings get secure enabled_accessibility_services
   ```
   没有 `guard.HomeGuardService` → 回第 7 步手动补。
3. 查守护有没有在跑：
   ```powershell
   adb shell "dumpsys accessibility | grep -c -E 'label=(Aerial|app.orchard.tv)'"
   ```
   - 是 `0` 但上一条有 Aerial → 常见于**升级之后**，系统还在找旧安装目录（小米 Android 9）。强停再开一次：
     ```powershell
     adb shell am force-stop app.orchard.tv
     adb shell am start -W -n app.orchard.tv/.MainActivity
     ```
     等 3 秒重新验收。
   - 是 `0` 且上一条也没有 Aerial → 固件拦了侧载的无障碍服务。去电视的**安全中心 / 权限管理**里给 Aerial 打开"自启动""无障碍"，再 `adb shell am start -n app.orchard.tv/.MainActivity` 重新验收。
4. 全部无效 → 你的机型在支持表里可能是 ❌（创维 7T641 / 75A6E 会黑屏卡死），止损还原。

### 6.4 小米的"正常但看着像失败"

- `resolve-activity` 显示 `com.mitv.tvhome` → **正常**，守护接管，不是失败。
- **千万不要停用 `com.mitv.tvhome`**。那是原桌面，停用它会让电视开不了机。
- 待机后回来慢：小米 Android 9 低内存机型（`ro.config.low_ram` 为 `true`，如 MFTP0）系统会强停 Aerial，靠同步服务 30–60 秒拉回来。**测待机请待机 1 分钟以上，唤醒后等 30 秒再看**，别在这期间按主页键或开别的 App。

### 6.5 其它

| 症状 | 原因 / 处理 |
|---|---|
| TCL 弹「多屏互动：正在发起应用推送」 | 正常。遥控器**先按「右」**选中"同意"再确认 |
| 无障碍开关过一会自己关了 | 电视的"安全中心/电视管家"在清理。把 Aerial 加进自启动白名单 |
| 开机先进原桌面再跳 Aerial | 固件在开机完成前拦截第三方 App。海信 VIDAA 属此类，按一次主页键即可 |
| 首页滑动卡顿 | 个性化 → 效果 → 选"流畅"（1.48 起低配机型会自动选） |
| 屏保黑屏 / 不出画面 | 航拍要连 `sylvan.apple.com`，网络差会 8 秒后换下一段，连续 3 次失败自动换照片屏保。可改选"照片"或"时钟" |
| 天气不显示 | 缺定位权限，补第 5 步第一条；城市也能在手机配置页手动设 |
| 电视上的 Aerial 图标点了没反应 | 它已经是默认桌面了，属于正常 |

### 6.6 确定不支持，别浪费时间

LG webOS、三星 Tizen、华为鸿蒙智慧屏、Android 4.4 及更老；打不开 ADB 的极米 H3、老乐视、荣耀智慧屏、老康佳。

---

### 6.7 雷鸟（FFALCON）专章：按"上下左右"没反应怎么办

雷鸟是 TCL 子品牌，**固件同源但机型跨度极大**，老教程和新机器完全是两套逻辑。先分档，别照着 TCL 的路径瞎试。

> 🟢 **先花 30 秒排除最好走的那一类**：如果你电视的默认桌面是 `com.google.android.apps.tv.launcherx`（Google TV），那它根本不适用本章——直接看下面 **6.10 节**。有些雷鸟是**双系统**机型（国行系统里藏着完整的 Google TV，可手动切过去，比如 27U8A-S），切到那一侧之后 standard 一装、role 一设就完事，10 分钟搞定。

**第一步：确认你是哪一档**

设置 → 关于 → 本机信息，看「系统版本 / Android 版本」和型号：

| 你的雷鸟 | 平台 | ADB 入口 | 能否接管主页键 |
|---|---|---|---|
| Android 9 / 11：鹏6、鹏7、鹤6（2023 及更早）、85S515D PRO、43F295C 等 | MT5879 / MT9652 / m7642 | ✅ 本机信息 → 上下左右 → 更多 → ADB | ✅ 多数可以 |
| **Android 14**：鹤6 Ultra、鹤7 Pro、S595C Ultra、98S595C Ultra-A 等 2025 款 | **MT9655** | ⚠️ 老固件只能靠有线 USB；**新固件可能有「无线调试 + 配对码」**，先查 | ⚠️ 需额外手段，见下 |

**为什么"上下左右"没反应（MT9655）**

不是按错了，是**入口被整个删掉了**。实测结论（issue #52 / #477）：

- 设置里只有 9 个分类：图像 / 声音 / 网络 / 蓝牙 / 信号源 / AI智能 / 通用 / 个性化 / 关于。**没有无障碍**，也没有「本机信息 → 更多」。
- `com.android.settings` 和 `com.android.tv.settings` 两个系统设置包**都不存在**，所以用 Intent 也调不出原生开发者选项页。
- ADB 走的是 **Android 11+ 的 TLS 无线调试**：端口每次开关都变（实测 34897 / 41607 / 46879 / 42803），**5555 / 5554 / 5037 全程 closed**。
- **没有「使用配对码配对设备」入口**：`adb pair` 报 `protocol fault`，`adb connect` 报 `failed to connect`。设备强制 `STLS`，没有明文回落。
  - ⚠️ **但这条只对当时那版固件（`V8-0015T11-LF1V487`，2026-10 中旬）成立。** 后来有用户的更新固件里出现了完整的「无线调试 → 已启用 + **使用配对码配对设备**」页面（能看到 IP 和端口，也有 6 位配对码入口）。**所以顺序要改成：先去设置里找「无线调试」页，有配对码入口就直接 `adb pair`，连得上就不用买线。** 具体做法见 [6.7.1](#671-如果你的电视有无线调试--配对码入口)。

**6.7.1 如果你的电视有「无线调试 + 配对码」入口**

**先用 30 秒确认，能省下一根线和一堆麻烦。** 在设置里找到「无线调试」（可能在 通用 / 系统 / 关于 附近，或从 Projectivy 的 `Additional settings` 进），看这个页面长什么样：

| 你看到的 | 结论 | 下一步 |
|---|---|---|
| 只有「已停用 / 已启用」两个选项，**没有**「使用配对码配对设备」 | 老固件，TLS 端口连不上 | 走下面的有线 |
| 有「已启用」＋ **「使用配对码配对设备」**按钮 ＋ IP 和端口 | ✅ 完整入口 | 按下面做，**不用买线** |

**配对三步**

1. 点遥控器选中 **「使用配对码配对设备」**。会弹出一个新页面，里面有**另一个** `IP:端口` 和一组 **6 位配对码**。
   > 注意：主页面显示的那个「IP 地址和端口」（例如 `172.19.0.1:39135`）是**连接端口**，不是配对端口。两个端口不一样，别混。
2. 在电脑上（PowerShell）执行配对，把端口和码换成电视上的：
   ```powershell
   adb pair 192.168.1.20:41234 123456
   ```
   分两段写也行（`adb pair 192.168.1.20:41234` 回车后按提示输入码）：
   ```
   Enter pairing code: 123456
   Successfully paired to 192.168.1.20:41234
   ```
3. 配对成功后**回到主页面**，用那个「IP 地址和端口」连接：
   ```powershell
   adb connect 192.168.1.20:39135
   adb devices
   ```
   `adb devices` 里出现 `192.168.1.20:39135    device` 就通了。

**两个高频坑**

- **页面显示的 IP 可能不是你的真实局域网 IP。** 见过显示 `172.19.0.1` 的（这是虚拟机/容器网桥常见的地址段），正常家用网段一般是 `192.168.x.x`。先用 **设置 → 网络 → 已连接的网络** 确认真实 IP，配对和连接时用真实 IP 替换掉页面上那个。
- **端口每次开关无线调试都会变**，重启后要回来重新看端口。配对信息本身会保留，通常不用重新配对，直接 `adb connect <新端口>` 即可；连不上就再 pair 一次。

配对通了之后，直接跳到 [路线 B0](#67-雷鸟ffalcon专章按上下左右没反应怎么办)——装 guard 版、写无障碍设置、搞定。

---

**MT9655 的备用入口：一根 USB-A 双头线**

如果上面那个页面没有配对码入口（老固件），这是唯一的路。

社区多位用户实测（issue #588 / #52 评论）：**先用有线连一次电脑完成授权，之后无线 ADB 才认你。**

1. 准备 **USB-A 转 USB-A 数据线**（两端都是大方口）。**必须是数据线**，纯充电线不行。
2. 一头插电视的 **USB 2.0** 口，一头插电脑。
3. 跑 `adb devices`。列表里出现的是**序列号**（不是 `IP:5555`），同时电视弹出「允许 USB 调试吗？」→ 遥控器选「**始终允许**」。
4. 授权成功后，同一把 `adbkey` 就被记住了。拔线后可以 `adb connect <IP>:<端口>` 走无线——端口在电视的调试入口里看，**不是 5555**。

> ⚠️ USB-A 对 USB-A 不在 USB 规范的标准线缆之列（两端都是 host 时理论上存在电气风险）。多位用户实测可用，但请自行判断。
> ⚠️ 关机后无线 ADB 会失效，下次要用得再插一次线。

**先纠正一个常见误解**

> **"无障碍设置页被删了" ≠ "无障碍服务用不了"。** 页面只是 UI；真正让系统去绑定服务的是 `Secure` 设置里的 `enabled_accessibility_services` 这一项，而它**可以用 ADB 直接写**——Aerial 官方 `install.sh` 在所有机型上做的就是这件事（第 7 步）。
> 所以只要拿到 ADB，守护就能起来，**根本不需要那个被删掉的页面**。

**MT9655 上装 Aerial 的三条路**

**路线 B0（首选，风险低）**：有线拿到 ADB → 装 guard 版 → 用 ADB 把守护写进无障碍设置。

这条路不需要碰安装器、不需要 standard 版，也不改任何系统应用。命令就是第 5、7 步那套，只是用 guard 包：

```powershell
adb install -r aerial-guard.apk
adb shell pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell cmd notification allow_listener app.orchard.tv/app.orchard.tv.guard.RevivalListener
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW allow
adb shell am start -W -n app.orchard.tv/.MainActivity

# 看守护有没有被系统绑定（这一步是关键判据）
adb shell settings get secure enabled_accessibility_services
adb shell "dumpsys accessibility | grep -c -E 'label=(Aerial|app.orchard.tv)'"
```

第一条 `get` 里没有 `guard.HomeGuardService` 时，按 [第 7 步](#第-7-步启用桌面守护最关键的一步) 手动写进去。

> ⚠️ 诚实说明：这条链路的每一环都有官方文档或实测记录支撑，但**"guard 版 + 有线 ADB"这个组合在 MT9655 上还没有公开的成功记录**（issue #588 那位用的是 standard）。所以它是"逻辑成立、待你实测"的第一顺位，而不是保证。

**路线 B1（零风险，但没有主页键）**：装 **guard 版**，走多屏互动推装或 U 盘安装 → Aerial 当普通 App 用。
点图标进 Aerial，主页键回 TCL 灵控桌面。壁纸、屏保、手机整理、图标全部可用，**只是抢不到主页键**。
多屏互动推装的命令见 [第 4 步](#第-4-步下载并安装-apk) 的「TCL 专用通道」（雷鸟同源，一样适用）。

**路线 B2（高风险，只在 B0 拿不到 ADB 时才考虑）**：让系统真的把 standard 版认成默认桌面。

原理：TCL 的 `LauncherStrategy` 会在**安装阶段**拦掉声明了 HOME 的第三方包。把电视自带的「软件包安装程序」临时停用，这层校验就绕过去了；装完必须立刻启用，否则开不了机。

```powershell
# 1. 先找出安装器的真实包名（不同固件不一样）
adb shell pm list packages | Select-String -Pattern install
#    常见是 com.android.packageinstaller 或 com.google.android.packageinstaller

# 2. 停用（这一步开始电视处于"不能装应用"的状态，别断电）
adb shell pm disable-user --user 0 com.android.packageinstaller

# 3. 装 standard 版
adb install -r aerial-standard.apk

# 4. 立刻、马上重新启用（不许拖，不许先去看效果）
adb shell pm enable com.android.packageinstaller

# 5. 设为默认桌面
adb shell cmd role add-role-holder --user 0 android.app.role.HOME app.orchard.tv
```

> 🔴 三条红线：**不许在停用状态下关机或断电**；**装完立刻 `pm enable`**；**绝不碰 `com.tcl.cyberui`**（TCL 灵控桌面，停用它等于让电视开不了机）。
> 作者原话："停用「软件包安装程序」有变砖风险，务必装完立刻启用；Aerial 官方流程不建议这样做。"

**B2 能换来什么、换不来什么**（issue #588 实测，鹤6 Ultra 98S595C Ultra-A / Android 14）

| 项 | 结果 |
|---|---|
| 按主页键回 Aerial | ✅ |
| 待机唤醒后是 Aerial | ✅ |
| 系统默认桌面解析到 `app.orchard.tv` | ✅ |
| 航拍屏保、手机扫码、音量亮度 | ✅ |
| **断电/关机重开后是 Aerial** | ❌ 仍进 TCL 灵控桌面，要按一次主页键 |
| Aerial 控制中心里切信号源 | ❌ 用遥控器「设置」里的原版信号源，或回原桌面切 |

**雷鸟上不要做的事**

- 不要停用 `com.tcl.cyberui`。这是 TCL 灵控桌面，也是开机流程的一部分，停用后电视会卡在黑屏或开机画面。
- 不要指望"恢复出厂设置"能解决接管问题，它只会把你的账号和设置一起清掉。
- 如果你的诉求只是"没有广告和推荐流的干净桌面"，**B1 已经能满足 80%**——Aerial 就在那儿，当个 App 用也不难受。真正想开机直达，成本收益比其实不高。

**如果你决定放弃折腾**（很多雷鸟用户最后的选择）：买一个电视盒子（小米盒子、腾讯极光 6 等，支持表里是 ✅），把 Aerial 装在盒子上，电视只当显示器。盒子是完整 Android TV，没有这些固件级封锁，一次装好、开机直达。

---

### 6.8 只用 U 盘/多屏互动装（不碰 ADB），能用到什么程度

很多人的实际选择是：ADB 折腾不动，先手动装上再说。那到底差在哪，这里说清楚。

**先理解一件事**：Aerial 的核心价值是"开机第一眼就是它"。手动装完之后，你得到的是**第二个桌面**，不是"你的桌面"。

```
开机 → TCL 灵控桌面 → 在 App 列表里找到 Aerial → 点进去 → 才看到 Aerial
任何时候按主页键 → 直接弹回 TCL 灵控桌面
```

所以它的定位变成"一个需要主动打开的应用"，而不是"电视的界面"。这一点差异，比下面任何一条功能缺失都更影响体验。

**逐项对照**

| 功能 | 手动装（无 ADB） | 有 ADB（完整授权） |
|---|---|---|
| 磁贴 / 文件夹 / 隐藏 / 排序 / 单双排 / 时钟 / 焦点样式 / 按键音 | ✅ | ✅ |
| 壁纸：必应每日、14 张内置、手机传图、航拍动态 | ✅ | ✅ |
| **桌面内屏保**（空闲 5 分钟自动播航拍，4K / 杜比视界） | ✅ | ✅ |
| 手机扫码整理（局域网页面） | ✅ | ✅ |
| 天气、家长密码、音量 | ✅ | ✅ |
| 打开任意 App、HDMI 磁贴 | ✅ | ✅ |
| **按主页键回 Aerial** | ❌ | ✅ |
| **开机 / 待机唤醒进 Aerial** | ❌ | ✅ |
| **设为系统默认桌面** | ❌ | ✅ |
| 系统级屏保（电视自己的屏保时机） | ❌ | ✅ |
| 亮度调节（Aerial 里滑） | ❌ 需 `WRITE_SETTINGS` | ✅ |
| 应用内升级（控制中心「有新版本」） | ❌ 每次手动重装 | ✅ |
| 切换过渡的悬浮窗 | ❌ | ✅ |

**几个常见误解**

- **"屏保也用不了？"** 能用。桌面内屏保是 App 自己的空闲计时器（5 分钟），不需要任何权限。丢的是"电视系统屏保时机"那条路径。
- **"手动装是不是白装，以后还要重来？"** 不是。guard 版和 standard 版同包名同签名，日后拿到 ADB 直接 `adb install -r` 覆盖安装，所有设置和已经排好的图标都保留。
- **"为什么点了没反应？"** 亮度滑块、控制中心的「有新版本」、设置里的"设为默认桌面"，在无 ADB 时会静默失败或提示无权限。这是**预期行为**，不是 bug。

**一个值得去翻一下的补偿方案**

看看电视的 设置 → 通用 → **开关机设置** 里，有没有"开机自动进入指定应用 / 指定信号源"。有些 TCL/雷鸟固件提供这个选项——如果有，把它指到 Aerial，就等于用固件自己的能力实现了"开机直达"，能补上手动装最大的短板。（这一项不同固件差异很大，有没有要自己确认。）

**关于 `android-tv-developer-enabler` 这类"开发者选项开启器"**

不要对它抱期望。它**不是破解工具**，README 自己写着："It does not grant any permission you didn't already have — it only navigates to the standard `android.provider.Settings` activities"。9 个按钮的实际构成：

| 按钮 | 实际做什么 | 在雷鸟 MT9655 上的结果 |
|---|---|---|
| 1–5、8、9 | 向 `com.android.settings` 发标准 Intent | ❌ 该包在本机**不存在**，`ActivityNotFoundException` |
| 6 | `Settings.Global.putInt(ADB_ENABLED, 1)` | ❌ 需要 `WRITE_SECURE_SETTINGS`，而它的 Manifest **一个权限都没申请**，抛 `SecurityException` |
| 7 | `Runtime.exec("settings put global adb_enabled 1")` | ❌ App 是普通 UID，不是 shell，被 SELinux 拒绝 |

也就是说，它试图做的事，恰好是项目里已经实测失败过的那批做法。成本只有 12 KB + 5 分钟，想验证可以试一次，但别指望。

顺带一提这个项目的体量：7 star、1 fork、2026-05-14 单次提交后再无更新、0 个 issue。源码注释里还写着 "on Samsung TVs"。

---

### 6.9 支线：用第三方启动器打开隐藏设置（"P 开头"的那个）

网上流传一个说法："让 AI 给你一个能打开隐藏设置的启动器，推送进去就行，P 开头的。"

**"P 开头"指的是 Projectivy Launcher**（开源，作者 Spocky，仓库 `spocky/miproja1`，官网 `spocky.github.io/miproja1`）。这条建议**不是瞎猜**——它的官方功能列表里明确写着：

> **Engineering Menus Access**：自动检测并直达隐藏的工程菜单（**MediaTek**、AmLogic、Xiaomi、FengOS…）
> **Additional settings**：打开 amlogic/mediatek 的设置应用（如果存在）
> **Android settings**：打开 Android 原生设置

你的电视是 **MediaTek MT9655**，正好在它的自动检测范围内。这是这条建议里唯一真正有希望的点：**TCL 自己的设置里没有开发者选项，但 MTK 的工程菜单是另一套东西，可能被保留下来了。**

**但它有个前置难题**

Projectivy 自己**声明 HOME**（它就是个桌面），所以装它的时候会遇到和 Aerial standard 版一模一样的 `LauncherStrategy` 拦截。装不进去，后面都免谈。

**值得按这个顺序试（约 20 分钟）**

1. **先补一个可能漏掉的步骤**：设置 → 关于 → **本机信息**，遥控器按 **上、下、左、右**，如果出现「更多」页，先看有没有「**商场模式**」——TCL/雷鸟 有固件要求**先开商场模式，下面的 ADB 开关才可点**。这一步漏了，开关就会是灰的。
2. **下载 Projectivy**（官方 Release，4.72，11 MB）：
   `https://github.com/spocky/miproja1/releases/download/4.72/ProjectivyLauncher-4.72-c97-xda-release.apk`
3. **用你已经会的方式推装**：U 盘 / 应用商店的「应用管家」推送，都可以。
4. **进它的设置**：Projectivy 首页**右上角的齿轮图标**；如果齿轮被隐藏了，用遥控器依次按 **↑ → ↓ ← ↑ → ↓ ←** 也能进。
5. 在设置里找**这一组**（都是"跳到某个设备的设置页"，名字随版本/语言略有差异）：

   | 条目 | 实际打开 | 对雷鸟有没有用 |
   |---|---|---|
   | `Device settings` / 设备设置 | 电视自己的设置（`com.tcl.settings`） | ⭕ 能开，但和你按遥控器进设置是同一个，没新东西 |
   | `Additional settings` / 附加设置 | **amlogic / mediatek 的设置应用** | ✅ **只有这个值得试**——MT9655 是 MediaTek |
   | `Android settings` / Android 设置 | `com.android.settings` 原生设置 | ❌ 这个包在你机器上不存在 |
   | `Engineering menu` / 工程菜单 | **小米 MiTV / 风行 FengOs / 峰米 Fengmi** 的工程菜单 | ❌ 雷鸟不在这个列表里，别在这上面花时间 |

   > 注意这两项容易搞混：**"工程菜单"是小米系的**，你的希望不在它身上；**"附加设置"才是 MTK 那个**。

6. **这几个条目是"检测到才显示"的**（官方原话是 "if available"）。所以**菜单里根本没有这两项，本身就是答案**：Projectivy 没在你这台机器上找到 MTK 设置。
7. 如果 `Additional settings` 能点开，在里面找 **ADB / USB Debugging / Debug 开关**。

**为什么大概率找不到（有实测依据）**

项目在那台 55S595C Ultra 上用 T-Cast 的 `223>>GET` 拉过**完整可启动应用列表**，一共只有 **12 个**：

> 云视听小电视、天气预报、应用商店、多屏互动、影视仓、用户中心、机顶盒设置、`com.tcl.settings`、Aerial、VidHub、媒体中心、网易云音乐

**里面没有任何 MTK / MediaTek / 工厂菜单相关的 App。** TCL 把自家的设置裁剪成 9 个分类的同时，也没留 MTK 那套入口。所以 Projectivy 检测不到，"附加设置"就不会出现。

**自己再确认一遍的笨办法**：Projectivy 设置 → `Edit categories` / 编辑分类 → `Manage apps` / 管理应用，这里会列出它扫描到的**全部应用**（含系统应用）。翻一遍，找有没有 `MTK`、`MediaTek`、`Factory`、`工程`、`工厂` 字样。没有 → 这条支线确认走不通。

**如果第 4 步任何一个能打开，你就拿到了 ADB 入口**，接着走 [6.7 的路线 B0](#67-雷鸟ffalcon专章按上下左右没反应怎么办)——那才是真正解决问题的路。

**三个提醒**

- 装上后**不要**把 Projectivy 设成默认桌面，否则它会和 Aerial 抢 Home，排查时你会分不清是谁在接管。测完直接卸载。
- 第 5、6 步如果这几个条目要么根本没有、要么点开报错，说明这台固件确实把 MTK 设置也清了，这条支线到此为止，回 6.7 走有线。
- **不要**为了装它去停用「软件包安装程序」。那是 B2 的动作，风险和收益不匹配，别为了一个探路工具冒变砖风险。

**这条支线的期望值**：中等偏低。但它成本最低、可完全回退（卸载即可），所以在插线之前值得先花 20 分钟试一次。

**⚠️ 后续实测修正（2026-10-29，TCL Google TV 侧）**

上面这节写的是 MTK 机型找工程菜单的路子。实机（Realtek RT2885P）验证后发现 **Projectivy 在这台机器上真正的价值是另一个**：

- 它设置里的 `Android settings` 能发 `android.settings.ACCESSIBILITY_SETTINGS`，**绕过 TCL 从设置界面里删掉的无障碍入口**
- 这就是「保留 Projectivy 的唯一理由」——不是为了工程菜单，是为了无障碍页

既然已经做出了等价的自建工具（见 **8.6**），**Projectivy 可以删掉，省 46 MB**。

> 结论：这一节在 MTK 机型上仍有参考价值；但如果你只是想进无障碍/开发者选项，直接跳到 8.5 / 8.6，别绕 Projectivy。

---

### 6.10 实测记录：双系统机型（雷鸟 27U8A-S → Google TV 侧）——最好走的一条路

**2026-10-29 实机验证通过，全程 10 分钟，零风险动作。**

有一类 TCL / 雷鸟电视是**双系统**的：国行固件是雷鸟自家系统（灵控桌面），但里面**藏了一套完整的 Google TV**，可以手动切过去。切到 Google TV 那一侧之后，它和 6.7 描述的完全是两回事，走的是**最干净的那条 standard 路线**。

> 实测机型：**雷鸟 27U8A-S**（用户手动切到内置的 Google TV 系统后测得）
> `model=C3APRO` / `device=G10` / `product=M01` / `brand=TCL` / Android 14 / build `UKR1.250628.001`

**⚠️ 双系统机型必须知道的四件事**

1. **Aerial 只装在 Google TV 那一侧。** 两套系统的数据是分开的，切回雷鸟系统看不到它，那边要装得重新装一遍。
2. **切回雷鸟系统后，那边大概率就是 6.7 描述的封锁环境**（灵控桌面 + LauncherStrategy）。**千万不要在雷鸟系统里用「停用软件包安装程序」那套**——收益不如直接在 Google TV 侧用。
3. **哪边用，就在哪边装。** 如果只是想要干净桌面，一直待在 Google TV 侧就行，没必要来回切。
4. **系统 OTA 有风险。** 隐藏的 Google TV 属于非公开功能，官方更新可能覆盖或改变切换方式。**升级前先把自己切换系统的方法记下来**（工程菜单路径、按键序列），别只记在脑子里。

**先判断你是不是这一类**

```powershell
adb shell getprop ro.board.platform
adb shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"
```

| 判断项 | Google TV 侧（好走） | 雷鸟自家系统（难走，见 6.7） |
|---|---|---|
| `ro.board.platform` | `RT2885P` 等 Realtek 平台 | `mt9655` / `mt5879` |
| 默认桌面 | `com.google.android.apps.tv.launcherx` | `com.tcl.cyberui` |
| `com.android.tv.settings` | **存在** | 不存在 |

**四步装完**

```powershell
# 1. 装 standard 版（这个平台不会拦 HOME 声明）
adb install -r aerial-standard.apk

# 2. 设为默认桌面
adb shell cmd role add-role-holder --user 0 android.app.role.HOME app.orchard.tv

# 3. 授权
adb shell pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell pm grant app.orchard.tv android.permission.ACCESS_FINE_LOCATION
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW allow
adb shell appops set app.orchard.tv WRITE_SETTINGS allow
adb shell appops set app.orchard.tv REQUEST_INSTALL_PACKAGES allow
adb shell appops set app.orchard.tv GET_USAGE_STATS allow
adb shell cmd notification allow_listener app.orchard.tv/app.orchard.tv.guard.RevivalListener

# 4. 启动
adb shell am start -W -n app.orchard.tv/.MainActivity
```

**不需要** guard 版、**不需要**无障碍守护、**不需要**碰软件包安装程序。这些机器走的是系统级默认桌面，比守护抢按键稳得多。

**验收（两个判据，别用错）**

```powershell
adb shell cmd role get-role-holders --user 0 android.app.role.HOME     # → app.orchard.tv
adb shell input keyevent KEYCODE_HOME
adb shell "sleep 3; dumpsys window | grep mCurrentFocus"               # → app.orchard.tv/...MainActivity
```

> ⚠️ **别拿 `cmd package resolve-activity` 判断成败。** 实测这台机器上它一直返回 `launcherx`（因为 `launcherx` 的 `priority=2` 压过 Aerial 的 `priority=0`），但 **role 已经归 Aerial，按主页键实际进的就是 Aerial**。看到 launcherx 别慌，以 role holders 和实际按键结果为准。

**屏保与流畅度（可选）**

```powershell
adb shell settings get secure screensaver_enabled        # 是 1 才做下面两条
adb shell settings put secure screensaver_components app.orchard.tv/app.orchard.tv.dream.AerialDream
adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez own_screensaver true

adb shell cmd package compile -m speed -f app.orchard.tv  # 几十秒，出 Success
```

**这次踩到的三个环境坑**

1. **电视开代理会让无线调试显示假 IP。** 开着代理时配对页显示 `172.19.0.1`（代理的 tun 网卡），配对必然失败；关掉代理后才显示真实地址 `192.168.1.5`。**配对前先把电视上的代理 / VPN 关掉。**
2. **配对成功后不一定能直接 connect。** 配对端口和连接端口是两个（`40125` vs `43831`），连接端口每次开关无线调试都变。配对后 adb 可能通过 mDNS 自动连上（`adb-<serial>._adb-tls-connect._tcp`）；断了就用 30000–50000 段扫一遍端口再 `adb connect`。
3. **用脚本批量操作时，别把 `adb connect` 和后续命令分成两次进程调用。** 每起一个新的 PowerShell 进程，adb 服务可能被重启（屏幕上会刷 `daemon not running; starting now at tcp:5037`），刚建立的无线连接随即断开。**把 `connect` 和要执行的命令写在同一条命令里**，一次跑完。

**开机直达已实测确认**

断电重启后复验：`cmd role get-roleholders android.app.role.HOME` 仍返回 `app.orchard.tv`，且电视开机后**直接进入 Aerial**，没有先闪一下原桌面。role 授予是持久化的，重启不丢。

> 提示：重启后无线调试会关闭，但 `service.adb.tcp.port=5555` 常开，`adb connect <IP>:5555` 仍可直接连上（不需重新配对）。


**重启之后怎么再连上（重要）**

这台机器重启后 `service.adb.tcp.port` 仍然是 `5555`，**adbd 常驻在经典端口上，重启不会丢**：

```powershell
adb connect 192.168.1.5:5555     # 直接连，不需要配对码
```

所以**你根本不需要任何第三方"ADB 工具"App**。那些软件（ADB TV、甲壳虫之类）只是替你跳转到设置页的快捷方式，页面本身 Google TV 就有：

> **设置 → 系统 → 开发者选项 → 无线调试**
> 如果找不到「开发者选项」：设置 → 系统 → 关于 → 连按「**Android 版本**」7 次

**另外建议跑一次这条**，让 ADB 被系统关掉后 Aerial 自动重开：

```powershell
adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez keep_adb true
```

回包出现 `data="keep_adb=true ..."` 就是生效了。

> ⚠️ 5555 常开意味着同局域网内**任何一台电脑**都能向电视发起调试请求——但首次连接必须由你在电视上点「允许」，所以未授权过的机器连不上。介意的话可以在用完后把开发者选项关掉，需要时再打开。

**如果设置里连「开发者选项」这一栏都被摘掉了**

有的固件（如雷鸟 27U8A-S 的 Google TV 侧）会把「开发者选项」和「关于 → Android 版本」一起从设置界面上删掉，连按 7 次也没用。**但页面本身还在**，只是入口被摘了。两条路：

**路 A（推荐，不需要电脑）**：装一个 8 KB 的快捷方式 App，图标出现在 Aerial 的应用列表里，点一下直接进开发者选项页。

这个 App 就是本工作区的 `开发者选项-快捷入口.apk`（源码在 `tools/devshortcut/`）。装法：

```powershell
adb install -r 开发者选项-快捷入口.apk
```

它做的事只有一件——发一个 `android.settings.APPLICATION_DEVELOPMENT_SETTINGS` Intent，失败时再按组件名逐个尝试 `com.android.tv.settings/.system.development.DevelopmentActivity`。**不申请任何权限，不需要 ADB 在线**，所以哪怕哪天 ADB 全关了它照样能用，这就是留后路的意义。

装完在 Aerial 的应用列表里找名字叫 **「开发者选项」** 的图标（可能在末尾，长按 OK 可以移到第一排固定）。

**路 B（用电脑，一次性）**：

```powershell
adb shell am start -n com.android.tv.settings/.system.development.DevelopmentActivity
```

先确认这个 Activity 在你的机器上存在：

```powershell
adb shell "dumpsys package com.android.tv.settings | grep -i development"
```

有输出就能用。`com.android.tv.settings` 不存在的机型（雷鸟自家系统）没有这条路。

---

## 七、日常使用、升级与还原

### 升级

Aerial 更新很勤（v1.69 就是 2026-10-29 当天发的），但**不需要整套重来**，两条路选一条。

#### 先确认：自己什么版本、最新什么版本

```powershell
# 当前装的是哪个版本
adb shell "dumpsys package app.orchard.tv | grep versionName"

# 官方最新版（联网查 GitHub Release，不需要登录）
curl.exe -s https://api.github.com/repos/pengchujin/aerial-tv/releases/latest | findstr "tag_name"
```

`tag_name` 是 `v1.69` 这种格式，直接和 `versionName` 对比。一样就是最新版，什么都不用做。

#### 方式 A：应用内升级（首选）

**1.32 起支持**。Aerial 控制中心出现「**有新版本**」→ 点进去 → 「**下载并安装**」→ 在电视自己的系统安装界面确认。

- **设置全部保留**：桌面接管、屏保、快捷键都不用重配
- 依赖 `REQUEST_INSTALL_PACKAGES` 权限，装机时那批 `appops` 里已经授过
- 看不到「有新版本」通常只有两个原因：确实已是最新版，或者这台固件拦了 App 安装

#### 方式 B：ADB 覆盖安装（应用内升级失败时）

```powershell
# 1. 下载新版（standard 版，和首次安装用同一个）
curl.exe -fL -o aerial-standard.apk https://aerial.qust.me/download/aerial-standard.apk
# 官网下载失败时改用这个：
# curl.exe -fL -o aerial-standard.apk https://github.com/pengchujin/aerial-tv/releases/latest/download/aerial-standard.apk

# 2. 覆盖安装 —— -r 是全部关键，保留数据与设置
adb install -r aerial-standard.apk

# 3. 重新编译（新版本 dex 变了，这步别省，否则首启会明显变慢）
adb shell cmd package compile -m speed -f app.orchard.tv

# 4. 启动
adb shell am start -W -n app.orchard.tv/.MainActivity
```

**不带 `-r` 等于全新安装**，所有设置和 HOME 授权都会丢，别省这个参数。

> ⚠️ **官方文档写着「TCL 的 `adb install` 一定失败（`INSTALL_FAILED_VERIFICATION_FAILURE`），只能走多屏互动」——TCL 的 Google TV 侧是例外。** 本机实测 `adb install -r` 直接 `Success`，装 1.69 和自编小工具都是这条路。**只有在灵控（国行）系统侧装才需要走多屏互动。**

#### 更新后必须验收这 4 项

```powershell
adb shell "dumpsys package app.orchard.tv | grep versionName"        # ① 版本号变成新的了
adb shell cmd role get-role-holders android.app.role.HOME             # ② 还是 app.orchard.tv
adb shell settings get secure screensaver_components                  # ③ 屏保还在
adb shell input keyevent KEYCODE_HOME
adb shell "sleep 3; dumpsys window | grep mCurrentFocus"              # ④ 按主页键进的是 Aerial
```

- **② 掉了**：`adb shell cmd role add-role-holder --user 0 android.app.role.HOME app.orchard.tv`
- **③ 掉了**：重跑第 8 步的屏保设置
- **④ 对了才算更新成功**，前三项都通过但 ④ 不对，按下面的「常见问题」处理

#### 升级相关的几个常见问题

- **升级后按主页键又回到原桌面了**：先 `am force-stop app.orchard.tv`，等 3 秒，再 `am start -W -n app.orchard.tv/.MainActivity`。macOS / Linux 可以直接跑 `curl -fsSL https://aerial.qust.me/install.sh | TV=<IP> bash -s repair`。
- **它不会自动更新**：`installerPackageName` 是 `null`，说明侧载、没有商店来源，`auto_update_apps` 也管不到它。升级只能靠应用内提示或手动重装。
- **升级会不会把桌面守护弄丢**：不会。权限和通知监听器是系统级记录，覆盖安装不动它们。但如果从 standard 换成 guard 版（或反过来），会多/少一个无障碍服务，需要重新授权。

### 还原（随时可退回原桌面）

**不会丢任何东西**——不停用系统应用、不删文件，只是把改动过的开关拨回去。

macOS / Linux 一条命令：

```bash
curl -fsSL https://aerial.qust.me/install.sh | TV=192.168.1.20 bash -s restore
```

Windows 逐条：

```powershell
adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez keep_adb false --ez own_screensaver false
adb shell cmd notification disallow_listener app.orchard.tv/app.orchard.tv.guard.RevivalListener
adb shell pm revoke app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW default
adb shell appops set app.orchard.tv REQUEST_INSTALL_PACKAGES default
adb shell appops set app.orchard.tv APP_AUTO_START default
adb shell appops set app.orchard.tv APP_ASSOC_START default
adb shell am force-stop app.orchard.tv
adb shell cmd role remove-role-holder --user 0 android.app.role.HOME app.orchard.tv
```

**还有三件收尾必须做**（否则残留设置会一直指着 Aerial）：

1. 屏保写回过默认值：
   ```powershell
   adb shell settings get secure screensaver_components          # 若是 app.orchard.tv/... 就要改回
   adb shell settings get secure screensaver_default_component   # 取这个值
   adb shell settings put secure screensaver_components <默认值>
   ```
2. 从无障碍列表里**摘掉** Aerial（保留其他服务，注意别留下多余冒号）：
   ```powershell
   adb shell settings get secure enabled_accessibility_services
   ```
   摘完只剩它自己时：`adb shell settings delete secure enabled_accessibility_services`
3. 海信 VIDAA 还原：`adb shell settings get global aerial_vt_home_pkg` 有值 → 用它写回 `persist.vt.use_home_pkg`，再 `adb shell settings delete global aerial_vt_home_pkg`。

最后按主页键回原桌面；想彻底删掉：`adb uninstall app.orchard.tv`。

### 想反馈问题

```powershell
curl.exe -fsSL https://aerial.qust.me/tools/report.sh | TV=192.168.1.20 bash
```

（只读，不改电视，生成一份 `aerial-report-机型.md`。诊断信息不含账号、IP、WiFi 名和 App 列表。）没有 bash 的话，用手机扫个性化里的二维码，在页面底部「反馈问题」复制。提到 GitHub Issues 时**一定要带机型**：`adb shell getprop ro.product.model`。

---

## 八、附录

### 8.1 官方 manual 最短路径（6 条，风险自负）

README 里给熟练用户的版本，只保留最小集合。**它不设屏保、不保 ADB、不显式设默认桌面**，靠守护自然接管：

```powershell
adb connect 192.168.1.20:5555
adb install -r aerial-standard.apk
adb shell pm grant app.orchard.tv android.permission.WRITE_SECURE_SETTINGS
adb shell cmd notification allow_listener app.orchard.tv/app.orchard.tv.guard.RevivalListener
adb shell appops set app.orchard.tv SYSTEM_ALERT_WINDOW allow
adb shell am start -n app.orchard.tv/.MainActivity
```

第一次装**不建议**用这个。少的那几条权限，出问题时排查成本远高于一开始就全给。

### 8.2 macOS / Linux 一条命令

```bash
curl -fsSL https://aerial.qust.me/install.sh | KEEP_ADB=1 TV=192.168.1.20 bash
```

脚本会自动下载最新 APK、安装、授权、接管桌面和屏保、编译，最后打印结果。不想要"保持 ADB"就去掉 `KEEP_ADB=1`；提示固件拒绝装桌面类 App 时在末尾加 `-s guard`。

### 8.3 交给 AI Agent 装（官方推荐路线）

把下面这段整段发给能在**电脑上执行命令**的 Agent（WorkBuddy、豆包电脑版、千问电脑版、Trae、Claude Code、Codex）：

```text
请帮我把开源电视桌面 Aerial 装到我的电视上，并设为开机进入的桌面。
电视 IP：<电视IP>
保持 ADB 常开：要

安装说明在这里：https://aerial.qust.me/agent.md

请先完整读完安装说明再动手，然后在我的电脑上亲自运行命令，不要只把步骤告诉我。
需要我在电视上操作时提醒我。最后按说明里的「验收」，把这几条命令的原始输出原样贴给我：
- adb shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"
- adb shell settings get secure enabled_accessibility_services
- 按主页键后的 adb shell "dumpsys window | grep mCurrentFocus"
```

**注意**：手机版、网页版聊天只会把步骤告诉你，不会执行——必须是能跑命令的电脑端 Agent。

### 8.4 关键命令速查

| 目的 | 命令 |
|---|---|
| 看连上没有 | `adb devices` |
| 重连 | `adb disconnect; adb connect <IP>:5555` |
| 重置 adb 服务 | `adb kill-server; adb start-server` |
| 看装的版本 | `adb shell "dumpsys package app.orchard.tv \| grep versionName"` |
| **看 HOME 到底归谁**（最可靠） | `adb shell cmd role get-role-holders --user 0 android.app.role.HOME` |
| **把 Aerial 设为默认桌面** | `adb shell cmd role add-role-holder --user 0 android.app.role.HOME app.orchard.tv` |
| **退回原桌面** | `adb shell cmd role remove-role-holder --user 0 android.app.role.HOME app.orchard.tv` |
| 看系统"以为"的默认桌面 | `adb shell "cmd package resolve-activity --brief -a android.intent.action.MAIN -c android.intent.category.HOME"` |
| 看守护状态 | `adb shell settings get secure enabled_accessibility_services` |
| 看当前前台窗口 | `adb shell "dumpsys window \| grep mCurrentFocus"` |
| 模拟按主页键 | `adb shell input keyevent KEYCODE_HOME` |
| 直接打开开发者选项 | `adb shell am start -n com.android.tv.settings/.system.development.DevelopmentActivity` |
| 看 ADB 是否开启 / 端口 | `adb shell settings get global adb_enabled`；`adb shell getprop service.adb.tcp.port` |
| 电视型号 | `adb shell getprop ro.product.model` |
| 芯片平台 | `adb shell getprop ro.board.platform` |
| 系统版本 | `adb shell getprop ro.build.version.release` |
| 是否低内存机型 | `adb shell getprop ro.config.low_ram` |
| 设置 Aerial 屏保 | `adb shell settings put secure screensaver_components app.orchard.tv/app.orchard.tv.dream.AerialDream` |
| 编译加速（首启变快） | `adb shell cmd package compile -m speed -f app.orchard.tv` |
| 让 Aerial 在被系统杀掉后自动重启 | `adb shell am broadcast -a app.orchard.tv.CONFIG -n app.orchard.tv/.guard.ConfigReceiver --ez keep_adb true` |
| 强停后重启 Aerial | `adb shell am force-stop app.orchard.tv; adb shell am start -W -n app.orchard.tv/.MainActivity` |
| **打开被隐藏的无障碍设置** | `adb shell am start -a android.settings.ACCESSIBILITY_SETTINGS` |
| 页面内容核对（截图不可用时） | `adb shell uiautomator dump /data/local/tmp/ui.xml` → `adb pull` → grep `text="..."` |
| **卸载预装应用（仅当前用户，可还原）** | `adb shell pm uninstall --user 0 <包名>` |
| **还原被卸载的预装应用** | `adb shell cmd package install-existing <包名>` |
| 只停用不卸载 | `adb shell pm disable-user --user 0 <包名>`／恢复用 `pm enable <包名>` |
| 看谁在吃内存（按占用排序） | `adb shell "ps -A -o RSS,NAME --sort=-rss"` |
| 看真实内存余量 | `adb shell "grep -E 'MemAvailable\|SwapFree' /proc/meminfo"` |
| 看桌面实际能看到哪些应用 | `adb shell "cmd package query-activities --brief -a android.intent.action.MAIN -c android.intent.category.LEANBACK_LAUNCHER"` |
| 看包名对应的安装目录 | `adb shell "pm list packages -f"`（`/system`、`/product` = 预装，`/data/app` = 后装或已更新） |

> ⚠️ **`resolve-activity` 不能用来判断接管是否成功。** 部分机型（如 6.10 的 TCL Google TV）上它会一直返回原桌面（`launcherx` 的 `priority=2` 压过 Aerial 的 `0`），但 role 其实已经归 Aerial、按主页键进的就是 Aerial。**以 `cmd role get-role-holders` + 实际按键结果为准。**

### 8.5 自己编一个 TV 快捷方式 App（无 Gradle、不装 Android Studio）

**什么时候用得上**：电视把某个设置页藏了、第三方 App 的入口又没了，你想做一个"点一下就直达"的图标。
本工作区的 `开发者选项-快捷入口.apk`（8.4 KB）就是这么来的，源码在 `tools/devshortcut/`。

它的全部逻辑就是发一个 Intent：

```java
// 先按 action 发，失败再按组件名精确跳，全失败退到系统设置
startActivity(new Intent("android.settings.APPLICATION_DEVELOPMENT_SETTINGS"));
```

**不申请任何权限、不依赖 ADB**——这一点最关键：哪天 ADB 全关了、能进设置页的口子也没了，它照样能点开。

**需要的三样东西（本工作区 `tools/` 已全部备齐，可离线）**

| 组件 | 路径 | 用途 |
|---|---|---|
| JDK 11 | `C:\Program Files\Java\jdk-11` | `javac` / `keytool` / `java` |
| build-tools 34 | `tools/bt/android-14/` | `aapt2` / `aapt` / `zipalign` / `apksigner` / `lib/d8.jar` |
| android.jar | `tools/plat/android-34/android.jar` | 编译期平台 API（`platform-34-ext7_r03.zip`） |

**源码目录约定**

```
tools/devshortcut/
├─ AndroidManifest.xml
├─ src/app/orchard/devshortcut/MainActivity.java
├─ debug.keystore              # 签名密钥，只需生成一次
└─ build/                      # 中间产物
   ├─ classes/                 # .class
   ├─ dex/classes.dex
   ├─ base.apk → aligned.apk → devshortcut.apk
```

**七步构建（PowerShell，在 `tools/` 的上级目录执行）**

```powershell
$BT    = "tools\bt\android-14"
$AJ    = "tools\plat\android-34\android.jar"
$SRC   = "tools\devshortcut"
$OUT   = "$SRC\build"
$JAVAC = "C:\Program Files\Java\jdk-11\bin\javac.exe"

# 1. 编译 Java（Java 8 字节码，兼容老机型）
& $JAVAC -source 1.8 -target 1.8 -cp $AJ -d "$OUT\classes" `
    "$SRC\src\app\orchard\devshortcut\MainActivity.java"

# 2. 转成 dex（用 java -cp 调 d8.jar，比调 d8.bat 稳）
& java -cp "$BT\lib\d8.jar" com.android.tools.r8.D8 `
    --lib $AJ --min-api 21 --output "$OUT\dex" `
    "$OUT\classes\app\orchard\devshortcut\MainActivity.class"

# 3. 链接清单，产出未签名 APK
& "$BT\aapt2.exe" link -o "$OUT\base.apk" -I $AJ `
    --manifest "$SRC\AndroidManifest.xml" --min-sdk-version 21 --target-sdk-version 34

# 4. 把 dex 塞进 APK —— 最容易漏的一步
Push-Location "$OUT\dex"
& "$BT\aapt.exe" add "..\base.apk" classes.dex
Pop-Location

# 5. 四字节对齐
& "$BT\zipalign.exe" -f 4 "$OUT\base.apk" "$OUT\aligned.apk"

# 6. 生成调试签名（只需执行一次，已存在就跳过）
& "C:\Program Files\Java\jdk-11\bin\keytool.exe" -genkeypair `
    -keystore "$SRC\debug.keystore" -storepass android -keypass android `
    -alias androiddebugkey -keyalg RSA -keysize 2048 -validity 10000 `
    -dname "CN=Android Debug,O=Android,C=US"

# 7. 签名
& "$BT\apksigner.bat" sign --ks "$SRC\debug.keystore" `
    --ks-pass pass:android --key-pass pass:android --ks-key-alias androiddebugkey `
    --out "$OUT\devshortcut.apk" "$OUT\aligned.apk"
```

**四个坑**

1. **`d8.bat` / `apksigner.bat` 在 Git Bash 里容易出错**（路径转换问题）。d8 用 `java -cp lib/d8.jar com.android.tools.r8.D8` 代替；apksigner 在 PowerShell 里直接调 `.bat` 反而没问题。
2. **第 4 步别漏。** 只做 `aapt2 link` 不塞 `classes.dex`，装上去要么闪退，要么提示"解析包出现错误"。这是最容易卡住的地方。
3. **`platform-34_r03.zip` 那个下载地址是 404**，要用 `platform-34-ext7_r03.zip`（同样的 android.jar，多带扩展 API）。
4. **manifest 里必须同时声明两个 category**：

   ```xml
   <category android:name="android.intent.category.LAUNCHER" />
   <category android:name="android.intent.category.LEANBACK_LAUNCHER" />
   ```

   只写 `LAUNCHER` 的话，电视桌面（包括 Aerial）的应用列表里**看不到图标**；只写 `LEANBACK_LAUNCHER` 则手机端不显示。两个都写才通吃。

**装上去验证**

```powershell
adb install -r tools\devshortcut\build\devshortcut.apk
adb shell am start -n app.orchard.devshortcut/.MainActivity
adb shell "dumpsys window | grep mCurrentFocus"
```

最后一条应该看到目标页面（本例是 `com.android.tv.settings/.system.development.DevelopmentActivity`）。
装好之后在 Aerial 应用列表末尾找名字叫「**开发者选项**」的图标，长按 OK 可以固定到第一排。

---

### 8.6 第二个实例：无障碍入口 + 预装应用精简

**背景**：这台 TCL 把「无障碍」整栏也从设置界面删了（和当初删「开发者选项」同一个套路）。一度靠 Projectivy 桌面里的入口进去，但为了一个隐藏页面常年挂一个 46 MB 的第三方桌面不划算。

#### 8.6.1 先探测：Activity 到底还在不在

**不要猜组件名。** 三条命令定位，最后一条最关键：

```powershell
# 1. 看这个 action 最终路由到哪个组件
adb shell "cmd package resolve-activity -a android.settings.ACCESSIBILITY_SETTINGS"

# 2. 看还有谁声明处理它（排除 frameworkpackagestubs 这种空壳）
adb shell "pm query-activities --brief -a android.settings.ACCESSIBILITY_SETTINGS"

# 3. 从系统的使用记录里找"谁打开过它" —— 这条是铁证
adb shell "dumpsys package com.android.tv.settings" | Select-String -Pattern "accessib"
```

第 3 条的输出里会带历史记录，形如：

```
mActivityComponent=com.android.tv.settings/.oemlink.AccessibilitySettingsActivity
time="2026-10-29 03:18:06" type=ACTIVITY_RESUMED
```

**这行就是证据**：某个 App 在某个时刻确实打开过这个页面。之前用 Projectivy 进过一次，就会留下这条记录，说明这条路走得通。

#### 8.6.2 实测结论（TCL C3APRO / Realtek RT2885P / Android 14）

| 用途 | 组件 |
|---|---|
| **无障碍主页** | `com.android.tv.settings/.oemlink.AccessibilitySettingsActivity` ← 处理 `android.settings.ACCESSIBILITY_SETTINGS` |
| 无障碍快捷方式服务 | `.../.oemlink.AccessibilityServiceActivity` |
| 字幕 | `.../.oemlink.AccessibilityCaptionActivity` |
| 色彩校正 | `.../.accessibility.AccessibilityColorCorrectionActivity` |

`exported=true`、`priority=1` —— **UI 入口被删了，Activity 本身完好。**

页面实测内容（`uiautomator` 抓的控件文字）：无障碍功能 / 屏幕阅读器 / TalkBack / 文字转语音 / 显示（文字缩放、粗体文字、色彩校正）。**是真页面，不是引导页。**

#### 8.6.3 照 8.5 的模板改三处就行

产物：`tools/a11yshortcut/`，成品 `无障碍设置-快捷入口.apk`（8.6 KB）。

| 改哪里 | 改成 |
|---|---|
| 包名 | `app.orchard.a11yshortcut` |
| label / action | 「无障碍设置」/ `android.settings.ACCESSIBILITY_SETTINGS` |
| 候选组件数组 | 上表那四个 |

**keystore 直接复用 `tools/devshortcut/debug.keystore`**，不用重新生成。代码结构一字不改：先发 action → 再逐个试组件名 → 全失败退系统设置。

装好后在 Aerial 应用列表里会出现「无障碍设置」。

#### 8.6.4 验证：这台电视截不了图

**`screencap` 在这台机器上永远输出 0 字节**（`/sdcard`、`/data/local/tmp`、`exec-out` 三种方式都是空文件），厂商限制。

改用 `uiautomator` 抓控件树，比截图还准 —— 能看到每个控件的文字：

```powershell
adb shell uiautomator dump /data/local/tmp/ui.xml
adb pull /data/local/tmp/ui.xml .
# grep text="..." 就能看到页面上所有文字
```

---

#### 8.6.5 顺带：精简预装应用（`--user 0` 到底是什么）

**一句话**：`pm uninstall --user 0 <包名>` 是「**对 0 号用户卸载**」，不是把 APK 从系统里删掉。

Android 是多用户系统：

- **`user 0`** = 机主本人这个用户（电视上只有它）
- 预装 APK 在**只读分区**（`/product`、`/system`），物理上删不掉
- 这条命令做的是：在「user 0 的已安装记录」里标记为已卸载 → 它**不显示、不启动、不接收开机广播**，但 APK 文件还躺在原地

三种操作的差别：

| | `pm uninstall --user 0` | `pm disable-user --user 0` | 真删（`rm` APK） |
|---|---|---|---|
| 桌面图标 | 消失 | 消失 | 消失 |
| 后台进程 / 自启 | 不启动 | 不启动 | 不启动 |
| **释放磁盘** | **不释放** | 不释放 | 释放 |
| 恢复方式 | `cmd package install-existing <包名>` | `pm enable <包名>` | 只能重刷 |
| 风险 | 低 | 最低 | 高 |

**为什么要用它**：对一台 1.8 GB 内存、Swap 快见底的电视，删掉不用的内容应用能实打实减轻内存压力 —— **收益在内存和自启，不在磁盘**（这台 25 GB 只用了 2.6 GB，本来不缺空间）。

**实测收益（2026-10-29，逐批累加）**

| 阶段 | MemAvailable | SwapFree |
|---|---|---|
| 清理前 | 490 MB | 101 MB |
| 删 9 个 TCL 内容应用后 | 634 MB | 302 MB |
| 再删 TCL 应用商店后 | 597 MB | 265 MB |
| **停用 launcherx + 删 Play 游戏后** | **741 MB** | **383 MB** |

> 读数随使用波动（缓存填充、其他 App 启动），看趋势别看单点。**SwapFree 最诚实**：它是真内存不够时才会用的。
> 单项排名：launcherx 两个进程共 **141 MB**（最大）、TCL 应用商店 **120 MB**、Play 游戏 **56 MB**、语音助手 **69 MB**、点播 **36 MB**、推送 **35 MB**、账号 **31 MB**。

**停用冗余桌面（单项收益最大，但要满足前提）**

```
adb shell pm disable-user --user 0 com.google.android.apps.tv.launcherx
```

这是**停用**不是卸载，`pm enable` 一条命令就回来。

⚠️ **前提条件（缺一不可）**：

1. `cmd role get-role-holders android.app.role.HOME` 已经返回 `app.orchard.tv`
2. 按主页键实测进的是 Aerial

两条都满足才能停用。否则 TV 上会没有可用桌面，只能靠 ADB 救。

停用后必须复验：按 HOME → `dumpsys window | grep mCurrentFocus` 应仍是 `app.orchard.tv`，且 `logcat -b crash` 无新增。

**可以删（TCL 内容服务，Google TV 侧完全用不到）**

`com.tcl.vod`（点播）、`com.tcl.c2dm.client`（推送）、`com.tcl.usercenter`（账号会员）、`com.tcl.playskill`（游戏）、`com.tcl.tliveplay`（直播）、`com.tcl.ffeducation`（教育）、`com.tcl.tclmusic`（音乐）、`com.tcl.wholenetsearch`（全网搜索）、`com.tcl.appmarket2`（TCL 应用商店 —— 有 Play 商店就不需要它，**实测它常驻 120 MB，是所有 TCL 内容应用里最占内存的一个**）

**看你还用不用**

`com.tcl.MultiScreenInteraction_TV`（手机投屏）、`com.google.android.katniss`（语音助手，删了遥控器语音键失效）、`com.google.android.youtube.tvmusic`、`com.google.android.play.games`（Play 游戏，实测常驻 56 MB，电视上基本用不到）、`com.google.android.marvin.talkback`（无障碍朗读）、`com.spocky.projengmenu`（Projectivy，做完 8.6 的自建入口后就没必要留了）

**绝对不要删**

| 包名 | 原因 |
|---|---|
| `com.tcl.ttvs` | **双系统切换核心**（处理 `MODE_SWITCH_HOME` + 开机广播），删了可能切不回灵控系统 |
| `com.realtek.*` | 硬件抽象层（显示、电源、鼠标键盘拦截、Miracast） |
| `com.google.android.gms` | Play 服务，商店和大量应用依赖 |
| `com.google.android.apps.mediashell` | 投屏接收核心 |
| `com.tcl.ui_mediaCenter` | U 盘播放 |
| `com.tcl.common.shortcutmenu` | 遥控器快捷键 |
| `com.sohu.inputmethod.sogou.tv` | 唯一中文输入法，删了搜索框打不了字 |

**操作纪律（三条，别省）**

1. 一次删一批，删完立刻验：`pm list packages <包名>` 应为空、`dumpsys window | grep mCurrentFocus` 应仍是 `app.orchard.tv`、`logcat -b crash` 无新增
2. **先 `pm disable-user` 观察几天，确认没影响再改成 `uninstall --user 0`** —— 前者一条命令 `pm enable` 就回来了
3. 删完别急着重启。真要重启验证，先确认 ADB 常开（`getprop service.adb.tcp.port` 有值），万一开不了机还能连上去 `install-existing` 救回来

---

### 8.7 三条红线（写在最后）

1. **不要随便停用、卸载「桌面 / 框架 / 驱动」类系统应用。** 内容类预装应用（影视、教育、游戏、推送）可以删，红线清单见 **8.6.5**；**拿不准时先用 `pm disable-user`，别直接 uninstall**。
   原桌面（`launcherx`、灵控 `cyberui`）只有在 **HOME role 已归 Aerial、且按主页键实测能进** 之后才能停用——这两条是硬前提，缺一条都别动，否则电视就没有可用桌面了。见 8.6.5 的「停用冗余桌面」。
2. **不要在电视上"恢复出厂设置"** 来"清理干净"。这不是清理，是把所有账号和设置一起清掉。
3. **`adb reboot` 之前先确认电视上没有正在进行的录制/下载**，也别在系统更新过程中重启。

---

## 修订记录

| 日期 | 内容 |
|---|---|
| 2026-10-26 | 依据 Aerial v1.52 官方文档（`site/agent.md`、README、`docs/ARCHITECTURE.md`）整理初版，命令未做改写 |
| 2026-10-29 | 增补 6.7 雷鸟专章、6.8 U 盘安装边界、6.9 第三方启动器路线 |
| 2026-10-29 | 新增 **6.10 双系统机型实测**（雷鸟 27U8A-S → Google TV 侧，v1.69 全程验证）；补「开发者选项被摘」的两条路；补开机直达复验结论；8.4 速查表补 role / 屏保 / 编译命令；新增 **8.5 自编译 TV 快捷方式 App** |
| 2026-10-29 | 新增 **8.6**：无障碍入口探测方法（实测 `oemlink.AccessibilitySettingsActivity`）+ `pm uninstall --user 0` 的真实语义与预装应用精简清单（实测数据）；6.9 补「TCL Google TV 侧 Projectivy 的真实用途」修正；8.4 补无障碍 / 卸载 / 内存排查命令；原 8.6 三条红线顺延为 8.7 并改写第 1 条 |
| 2026-10-29 | **第七章「升级」重写**：版本自查、应用内升级、`adb install -r` 完整流程、更新后 4 项验收、3 个常见问题；补「TCL 的 adb install 会失败——但 Google TV 侧是例外」这一实测修正；8.6.5 新增「停用冗余桌面」（launcherx，含两条硬前提）与四阶段收益表 |

> 上游更新较快，执行前可对照 https://aerial.qust.me/agent.md 确认是否变化。
> 本机实测环境：TCL C3APRO（雷鸟 27U8A-S 的 Google TV 侧）/ Realtek RT2885P / Android 14 (SDK 34)，Aerial v1.69。
