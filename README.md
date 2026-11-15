# PCL-DeFS

基于 **Plain Craft Launcher 2（PCL2）2.14.0.0** 深度定制的 Minecraft 启动器，由 **龙腾猫跃、incide_01** 制作。
采用黄铜主题与机械动力（Create）齿轮背景，内置 FlyServer 整合包一键下载安装、启动器全自动更新、版本按键修改与存档备份等增强功能。

- 当前版本：**2.14.2**
- 技术栈：VB.NET + WPF，.NET Framework 4.8
- 主程序：`PCL-DeFS.exe`

## 主要特性

### 界面与个性化
- 黄铜配色主题，机械动力齿轮动态背景。
- 个性化页提供**液态玻璃**开关：开启后启动器切换为液态玻璃样式，关闭恢复原有样式。
- 个性化页提供**悬浮效果**开关：开启后按钮具有悬浮感，关闭恢复原有样式。
- 背景设置支持**背景预览**，可通过原有按钮添加自定义背景并在预览中选择；内置「兽耳地铁」视频背景。
- 「更多」页精简了百宝箱与投票入口，新增「关于」页（显示版本号、支持检查更新）。

### FlyServer 整合包
- 内置整合包仓库，下载页可直接浏览 FlyServer 整合包版本列表并一键下载安装。
- 支持整合包更新提醒。

### 启动器自动更新
- 启动时自动比对内置更新仓库中的启动器版本，发现新版本后自动下载，提示重启即可完成更新。
- 「设置 → 其他 → 启动器更新」可切换更新方式：
  - 有新版本时自动下载（默认）
  - 有新版本时显示提示
  - 仅在有重要更新时显示提示
  - 关闭更新检查
- 同页提供「检查更新」按钮手动检查；「更多 → 关于」页也可检查更新。
- 更新过程不在界面、提示或日志中显示仓库地址。

### 版本设置增强
- **按键设置**：直接读取当前版本的 `options.txt`，以中文名称列出全部按键绑定；点击「修改」后按下新按键（支持键盘与鼠标左/右/中键）即保存，Esc 取消。若该版本尚无按键记录，则展示一套原版默认按键。
- **备份**：可将「仅存档文件夹（saves）」或「整个游戏文件夹」一键打包为 ZIP；恢复时选择备份 ZIP，确认后解压覆盖回原位置。

## 版本号规则

- 每次更新版本号 **+0.0.1**。
- 第三位（修订号）到 10 时向第二位进位，例如 `2.14.9 → 2.15.0`。
- 版本号统一维护在 `Modules/Base/ModBase.vb` 的 `VersionNumber` 常量，并同步到程序集版本；「关于」页直接显示该版本号。

## 构建与打包

1. 使用 Visual Studio 2022（含 .NET 桌面开发工作负载）打开 `PCL-DeFS/PCL-DeFS/Plain Craft Launcher 2.sln`。
2. 以 **Release / AnyCPU** 编译，输出位于 `PCL-DeFS/PCL-DeFS/Plain Craft Launcher 2/bin/PCL-DeFS.exe`。
3. 打包发布时，将以下 7 个文件放入 `PCL-DeFS-Release-{版本号}` 文件夹，并压缩为 `PCL-DeFS-{版本号}.zip`（不带运行时生成的 `PCL` 目录）：
   - `PCL-DeFS.exe`
   - `PCL-DeFS.exe.config`
   - `PCL-DeFS.xml`
   - `MeloongCore.dll`
   - `MeloongCore.Wpf.dll`
   - `PCLCS.dll`
   - `Microsoft.Win32.Registry.dll`

## 目录结构

```
PCL-DeFS/                     工作区根目录
├─ PCL-DeFS/                  源码仓库
│  └─ PCL-DeFS/
│     ├─ Plain Craft Launcher 2/   启动器主项目（VB.NET / WPF）
│     ├─ MeloongCore/              基础库
│     └─ PCLCS/                    C# 组件
├─ PCL-DeFS-Release-2.14.2/   当前发布文件（7 个）
├─ PCL-DeFS-2.14.2.zip        当前发布压缩包
└─ 宣传素材/                   宣传与验证素材
```

## 鸣谢

- 原版启动器：**Plain Craft Launcher 2**，作者 **龙腾猫跃**。
- 本定制版（PCL-DeFS）修改与维护：**incide_01**。

### 相关资源
- [PCL-DeFS 下载](https://wwasi.lanzoub.com/b01i60kwfe)：下载可免费使用的正式版 PCL 密码:defs
