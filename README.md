> **本仓库为社区修改版，不是 PCL 官方仓库。**
> 基于官方 2.99.0.0 源码修改，恢复了 MC 下载功能，并做了少量安全加固，非官方发布，与 PCL 作者及官方团队无关。
> 官方项目请访问 https://github.com/Meloong-Git/PCL
>
> **注意：本项目的代码排查、修改与版本更新均由 AI 完成。相关改动虽已经过人工复核，但受限于个人技术水平，仍可能存在未预见或未发现的问题，请警惕并自行评估风险后再使用。**

# PCL 社区修改版

[![Stars](https://img.shields.io/github/stars/Meloong-Git/PCL?style=flat&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZlcnNpb249IjEiIHdpZHRoPSIxNiIgaGVpZ2h0PSIxNiI+PHBhdGggZD0iTTggLjI1YS43NS43NSAwIDAgMSAuNjczLjQxOGwxLjg4MiAzLjgxNSA0LjIxLjYxMmEuNzUuNzUgMCAwIDEgLjQxNiAxLjI3OWwtMy4wNDYgMi45Ny43MTkgNC4xOTJhLjc1MS43NTEgMCAwIDEtMS4wODguNzkxTDggMTIuMzQ3bC0zLjc2NiAxLjk4YS43NS43NSAwIDAgMS0xLjA4OC0uNzlsLjcyLTQuMTk0TC44MTggNi4zNzRhLjc1Ljc1IDAgMCAxIC40MTYtMS4yOGw0LjIxLS42MTFMNy4zMjcuNjY4QS43NS43NSAwIDAgMSA4IC4yNVoiIGZpbGw9IiNlYWM1NGYiLz48L3N2Zz4=&logoSize=auto&label=Stars&labelColor=444444&color=eac54f)](https://github.com/Meloong-Git/PCL/)
[![Issues](https://img.shields.io/github/issues/Meloong-Git/PCL?style=flat&label=Issues&labelColor=444444&color=1F883D)](https://github.com/Meloong-Git/PCL/issues)
[![哔哩哔哩](https://img.shields.io/badge/动态-BiliBili-00A4DB?style=flat&labelColor=444444&logoSize=auto)](https://space.bilibili.com/11343203/dynamic)
[![爱发电](https://img.shields.io/badge/赞助-爱发电-946ce6?style=flat&labelColor=444444&logoSize=auto)](https://meloong.com/afd/a/LTCat)

本仓库是 [Plain Craft Launcher](https://github.com/Meloong-Git/PCL) 的社区修改版，基于官方 2.99.0.0（提交 [`6b9a50d`](https://github.com/Meloong-Git/PCL/commit/6b9a50df911c9d68b69b24c979efba477fb14b67)）构建。

## 这个分支做了什么

| 项目 | 官方 2.99.0.0 | 本分支 |
|---|---|---|
| Minecraft 下载 | 已移除 | **已恢复**（下载页 / 整合包安装 / 版本修改页 / 拖拽安装） |
| 联机（EasyTier） | 强制隐藏、未启用 | 与官方一致，**未启用** |
| 启动弹窗 | 开源版说明 | 增加社区修改版声明与免责说明，标注上游提交 |
| 程序内名称 | Plain Craft Launcher 2 | **PCL 社区修改版**（窗口标题、关于页、版本号） |
| 「执行命令」确认 | 可勾选「不再询问」永久跳过 | **每次强制确认**，且禁止被自定义事件改写 |

除上述改动外，UI、逻辑、依赖与上游保持一致；未加入任何后门、远程控制、静默下载或额外的数据上报。

## 下载

- 发布页：[Releases](https://github.com/hmaoraze/PCL/releases)
- 启动器为**免安装单文件**，下载后双击即可运行；需要 .NET Framework 4.8（Windows 10 1903+ / Windows 11 已自带）
- 本程序未做代码签名，首次运行可能被 SmartScreen 拦截，选择「更多信息 → 仍要运行」；文件大小与 SHA-256 以对应 Release 的说明为准，下载后可自行校验：

```bat
certutil -hashfile "PCL-2.99.0.0-community.1.exe" SHA256
```

## 关于「公共通道 / 联机」的说明

针对社区中流传的「PCL 留有后门」说法，本分支的核查结论如下（完整报告见 [`docs/security-review-shared-channel-and-link.md`](docs/security-review-shared-channel-and-link.md)）：

- **联机功能未启用**。上游 2.99.0.0 已通过 `Settings.Get` 强制隐藏联机页与联机设置，且开源版的联机配置源（`ServerConfig`）为空实现，EasyTier 相关代码不会执行，因此不存在「加入房间后被他人控制」的风险。
- 上游的**自定义主页**支持通过联网 XAML 定义按钮事件，其中包含「执行命令」。本分支已加固：执行命令 / 打开文件**每次都必须手动确认**，不能被「不再询问」永久跳过，也不能通过自定义事件改写该开关。
- 仓库内不存在后门、远程控制或隐藏上报逻辑；所有网络请求均可在 `Modules/Base/ModNet.vb` 及各模块的 `NetRequest*` 调用处查证。

## 从源码构建

环境要求：Windows 10/11、.NET SDK 9 或更高（提供 VB 编译器）、.NET Framework 4.8 引用程序集。

- 使用 Visual Studio：打开 `Plain Craft Launcher 2.sln`，需要「.NET 桌面开发」工作负载与 .NET Framework 4.8 目标包，选择 Release 生成。
- 使用命令行：仓库根目录的 `build-release.bat` 会先编译 `PCLCS`，再编译 VB/WPF 主工程，产物输出到 `dist\`。
  > 注意：该脚本中的 .NET SDK 与引用程序集路径为作者本机路径，克隆后需按自己的环境修改。

## 上游项目

以下为上游仓库的原始介绍：

包含 PCL 的大部分源代码，包括 UI 库、动画模块、下载模块、Minecraft 启动模块等！<br/>
代码并不是实时更新的，它会在每次 PCL 发布更新时同步一次。<br/>
非常感谢社区的大力支持！

### 相关资源

- [PCL 下载](https://meloong.com/pcl)：下载可免费使用的正式版 PCL！
- [PCL 功能投票](https://github.com/Meloong-Git/PCL/discussions/2)：来参加投票吧，开发者会优先处理票数较高的项目！
- [帮助文档库](https://github.com/LTCatt/PCL2Help)：PCL 内置帮助文档的存储库

## 许可与署名

- 原项目：**Plain Craft Launcher（PCL）**，作者 **龙腾猫跃（LTCat）**
- 项目主页：https://github.com/Meloong-Git/PCL
- 赞助作者：https://meloong.com/afd/a/LTCat
- 本修改版遵循仓库内的《PCL 分发有限许可》与《PCL 存储库合理使用指南》，源码公开，关于页面保留作者署名与赞助入口。
- 免责声明：本修改版未加入病毒或恶意代码，但基于上游开源代码及其依赖构建，无法完全排除上游被投毒的可能；若因此产生任何问题，本修改版不承担相关责任。
