# PCL 开源版安全审查：所谓「公共通道后门」与「联机漏洞」

> 审查对象：本仓库 `main` 分支（基于上游提交 `6b9a50d`，PCL 2.99.0.0 正式版）
> 审查日期：2026-10-25
> 结论：**视频里的两条技术指控，一条是真实存在的设计缺陷（已修复），一条在开源版不成立。**

---

## 结论速览

| 视频中的说法 | 审查结论 | 依据 |
|---|---|---|
| 「公共通道可以在用户电脑上远程执行命令」 | **部分成立**：机制真实存在，但不是隐蔽后门，需要用户启用联网主页并点击、并确认 | 见 §1 |
| 「联机节点 IP 暴露，可被改造成反向 VPN / 后门」 | **在开源版不成立**：联机入口被强制隐藏，且即便启用也有端口白名单 | 见 §2 |
| 「这个漏洞足以让 GitHub 封禁作者账号」 | **技术上不成立**：GitHub 封号针对恶意代码、滥用、侵权，不是「带确认框的用户自定义脚本功能」 | 见 §3 |

---

## 1. 指控一：公共通道可远程执行命令

### 1.1 所谓「公共通道」是什么

开源版中并不存在一个隐藏的远程控制信道。全仓库里唯一的「远端内容 → 本机执行」路径，是**联网自定义主页**：

```
设置 → 主页 → 联网获取  (UiCustomType = 2, UiCustomNet)
    ↓
PageLaunchRight.RefreshReal → NetRequestByClientRetry 下载 XAML
    ↓  （缓存到 %Temp%\PCL\Cache\Custom.xaml）
PageLaunchRight.LoadContent → 剥离 xmlns，只保留 5 个命名空间
    ↓
ModBase.GetObjectFromXML → 黑名单校验 → XamlReader.Load
    ↓
自定义事件 CustomEventService.EventType = 执行命令
    ↓
ProcessStartInfo(FileName = cmd.exe, Arguments = /c ...) → StartProcess
```

对应代码：

| 环节 | 位置 |
|---|---|
| 联网下载主页 | `Pages/PageLaunch/PageLaunchRight.xaml.vb:191` |
| 命名空间剥离（只留 xaml/local/core/corewpf/sys） | `PageLaunchRight.xaml.vb:249-253` |
| XAML 加载与黑名单 | `Modules/Base/ModBase.vb:1114-1147` |
| 事件类型定义（含 `执行命令`） | `Modules/ModEvent.vb:148-174` |
| 执行命令实现 | `Modules/ModEvent.vb:197-220` |
| 确认弹窗 | `Modules/ModEvent.vb:434` |

### 1.2 现有防护（三道，不是没有）

1. **命名空间白名单**：加载前把内容里所有 `xmlns` 删掉，再套上固定声明，只允许 `PCL`、`MeloongCore`、`MeloongCore.Wpf`、`mscorlib`（`sys`）和 XAML 本身。**没法直接 new 任意 .NET 类型**，`System.Diagnostics.Process` 这类根本引用不进来。
2. **类型黑名单**：`WebBrowser / Frame / MediaElement / ObjectDataProvider / XamlReader / Window / XmlDataProvider / SettingService`，以及成员 `Code`、`FactoryMethod`、`Static`（杜绝 `x:Code`、`ObjectDataProvider` 调静态方法这类经典 RCE 手法）。
3. **交互 + 确认**：所有自定义事件都绑在点击/勾选/回车这类用户操作上（`MyButton.xaml.vb:149`、`MyCard.vb:297` 等），没有任何 `Loaded` 自动触发；执行前弹「执行确认」。

> 这也解释了为什么视频里另一位的结论是「设计缺陷，未发现恶意行为」——**代码里确实没有后门，只是提供了一个可被滥用的脚本能力**。

### 1.3 真实存在的问题（本次审查发现）

| # | 问题 | 严重度 |
|---|---|---|
| A | 「执行确认」提供了「继续且今后不再要求确认」，一旦选中，写死 `HintCustomCommand = True`，**此后所有执行命令永久静默** | 中 |
| B | `写入设置` / `修改设置` 事件可以写任意非加密设置项，包括 `HintCustomCommand` 本身 → **恶意主页可以自己把确认框关掉** | 中 |
| C | 黑名单是「黑名单」而非白名单，`local:` 命名空间下 PCL 自身的控件属性不受限制（正是利用点） | 低（需配合 A/B） |

### 1.4 已修复内容

1. `ModEvent.vb:434` 的 `EventSafetyConfirm` 增加 `AllowPersistentSkip` 参数；`执行命令 / 打开文件` 传入 `False`：
   - 不再提供「不再要求确认」选项
   - **也不再被 `HintCustomCommand` 跳过** —— 每次都必须手动确认
2. `Settings.vb:301` 的 `SetSafe` 禁止写入 `HintCustomCommand`，堵死「恶意主页自己关确认框」的链路。

### 1.5 残余风险（未改动，建议后续处理）

- 联网主页仍可触发 `下载文件`、`加入房间` 等事件（有确认，但可一次性跳过）。若要更严格，应把「来源是网络」这一信息透传到事件层，对远程来源完全禁用 `执行命令 / 打开文件`。
- XAML 黑名单未来仍可能被新类型绕过，长期应改成**控件白名单**（只允许 PCL 自己的控件 + 基础面板）。

---

## 2. 指控二：联机节点 IP 暴露、可改造成反向 VPN

### 2.1 开源版的联机是「停用」状态，不是「隐藏入口」那么简单

`Pages/PageSetup/Settings.vb:319-325` 在读设置时强制覆盖：

```vb
Static ForcedSettings As New Dictionary(Of String, Object) From {
    {"UiHiddenPageLink", True},
    {"UiHiddenSetupLink", True}
}
```

也就是说，**不管用户配置文件里写什么，联机页面与联机设置都被强制标记为隐藏**；`FormMain.xaml.vb:970` 的剪贴板邀请码入口也被 `'TODO: 联机复活赛` + 立即 `Return` 短路。

更要紧的是：开源版的 `ServerConfig` **全仓库没有任何赋值点**（`ModSecret.vb` 里 `ServerLoader` 是空实现）。而联机流程第一步就是 `ServerConfig("Link")("EasyTierVersion")`（`PageLinkMain.xaml.vb:469`），空引用必然抛异常 —— **联机代码在开源版根本跑不起来**。

### 2.2 即便启用，也不是「全端口暴露」

`PageLinkMain.xaml.vb:524-537` 的 EasyTier 启动参数：

| 角色 | 关键参数 | 效果 |
|---|---|---|
| 房主 | `--tcp-whitelist={MC端口} --udp-whitelist={MC端口}` | 只放行 Minecraft 端口 |
| 加入者 | `-d --tcp-whitelist=0 --udp-whitelist=0` + 把 MC 端口 `--port-forward` 到 `127.0.0.1` | 对虚拟局域网内其他成员**完全不可达**，只能反向连房主 |

也就是说，所谓「进房间后别人能远程执行命令」在实现层面被端口白名单挡住了 —— 除非攻击者能攻破 Minecraft 服务端本身。

### 2.3 结论

「联机漏洞」这条在**开源版**不成立：入口强制隐藏 + 配置源为空 + 代码本身有端口白名单。
至于「节点 IP 暴露」，那是任何 P2P 虚拟局域网（ZeroTier / Hamachi / Radmin）的固有属性 —— **节点知道自己连了谁**，不等于能把你的机器变成肉鸡。

---

## 3. 指控三：足以让 GitHub 封禁作者账号

GitHub 的账号处置针对的是：恶意软件、滥用平台、侵权（DMCA）、泄露密钥、诈骗等。一个**默认关闭、需要用户主动启用、执行前有确认框**的 UI 脚本能力，不构成上述任何一条。

真正的封号风险来自另外两件事，与代码无关：
- 仓库里提交了密钥 / Token（本仓库无）
- 分发盗版或规避授权的二进制

---

## 4. 给本 fork 的说明（可直接粘贴到 README）

```markdown
## 关于「公共通道 / 联机」的说明

- 本分支**未启用联机功能**：上游 2.99.0.0 已将联机入口强制隐藏，且开源版的联机配置源为空，
  EasyTier 相关代码不会执行，因此不存在「加入房间后被他人控制」的风险。
- 上游的「自定义主页」支持通过联网 XAML 定义按钮事件，其中包含「执行命令」类型。
  本分支已做加固：执行命令/打开文件**每次都必须手动确认**，不能被「不再询问」永久跳过，
  也不能通过自定义事件改写该开关。
- 本分支未加入任何后门、远程控制、静默下载或数据上报逻辑；所有网络请求均可在
  `Modules/Base/ModNet.vb` 与各模块的 `NetRequest*` 调用处查证。
```

---

## 5. 如何自行复核

```bash
# 1. 确认联机被强制隐藏
grep -n "UiHiddenPageLink" "Plain Craft Launcher 2/Pages/PageSetup/Settings.vb"

# 2. 确认开源版没有服务端配置来源
grep -rn "ServerConfig = " "Plain Craft Launcher 2"          # 应无输出
grep -n "PCL 配置更新" "Plain Craft Launcher 2/Modules/ModSecret.vb"

# 3. 查看 XAML 黑名单
sed -n '1114,1147p' "Plain Craft Launcher 2/Modules/Base/ModBase.vb"

# 4. 查看执行命令的确认逻辑
sed -n '197,220p' "Plain Craft Launcher 2/Modules/ModEvent.vb"
```

---

## 6. 一句话总结

**没有找到后门。** 找到的是一处「用户自定义脚本 + 可被永久关闭的确认框」的设计缺陷，已修复；
联机相关的指控在开源版不成立，因为这块功能在 2.99.0.0 里已经被强制停用了。
