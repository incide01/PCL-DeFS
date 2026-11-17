# PCL-DeFS 插件开发指南

本文适用于 **PCL-DeFS 2.15.0、插件 API 1**。插件使用 VB.NET 或 C# 编写，目标框架为 **.NET Framework 4.8**；文中的示例使用 VB.NET。

## 1. 插件包是什么

PCL-DeFS 插件包以 `.jar` 为扩展名，但它在这里只是一个 **ZIP/JAR 容器**，不是 Java 插件。包内承载插件清单、.NET 程序集和资源，宿主通过 .NET 反射加载入口类型。

> **重要：这是 PCL-DeFS 启动器插件，不是 Minecraft 模组，也不是 Bukkit/Spigot 等服务端插件。** 插件扩展的是启动器本体的界面与能力，和 Minecraft 游戏内容无关；Minecraft 模组请通过启动器「下载」页管理，不要把模组 JAR 放进 `PCL/Plugins/`。

- **不需要安装或启动 JVM**，插件 DLL 也不是 Java 字节码。
- 插件项目必须兼容 **.NET Framework 4.8**，不能直接把 .NET 6/7/8、.NET Standard 不兼容依赖或 Java 类库当作入口程序集。
- 一个插件包至少包含 `META-INF/MANIFEST.MF`、根目录 `plugin.json` 和清单指定的入口 DLL。
- 路径和文件名应与示例保持一致，尤其是 `META-INF/MANIFEST.MF` 与 `plugin.json` 的位置。

典型包结构：

```text
Contoso.Hello.jar
├─ META-INF/
│  └─ MANIFEST.MF
├─ plugin.json
├─ Contoso.Hello.dll          # 入口程序集
├─ Contoso.Helper.dll         # 可选：插件私有依赖，建议放在包根目录
└─ Assets/
   └─ Example.png             # 可选：插件资源
```

`MANIFEST.MF` 可使用以下内容。当前运行时要求该条目存在，但不从其中读取入口信息；入口仍以 `plugin.json` 为准：

```text
Manifest-Version: 1.0
Created-By: PCL-DeFS Plugin Packager
```

## 2. plugin.json 完整说明

### 2.1 示例

```json
{
  "id": "Contoso.Hello",
  "name": "示例插件",
  "version": "1.0.0",
  "description": "在设置页显示一段插件文本。",
  "author": "Contoso",
  "apiVersion": 1,
  "minHostVersion": "2.15.0",
  "entryAssembly": "Contoso.Hello.dll",
  "entryType": "Contoso.Hello.PluginEntry",
  "autoEnable": true,
  "dependencies": [
    {
      "id": "PCL.DeFS.Personalization",
      "minVersion": "1.0.0"
    }
  ]
}
```

### 2.2 字段表

| 字段 | 类型 | 必需性 / 默认值 | 说明 |
| --- | --- | --- | --- |
| `id` | 字符串 | **必需** | 插件唯一标识，不区分大小写。只能包含字母、数字、点 `.`、连字符 `-`、下划线 `_`；建议使用反向域名风格。运行目录、依赖和启用状态都以此关联。 |
| `name` | 字符串 | 可选，默认空字符串 | 面向用户的显示名称，建议始终填写。 |
| `version` | 字符串 | **必需** | 插件版本，必须能由 .NET `System.Version` 解析，推荐 `主版本.次版本.修订号`，如 `1.0.0`。每次更新应提高版本。 |
| `description` | 字符串 | 可选，默认空字符串 | 功能简介。 |
| `author` | 字符串 | 可选，默认空字符串 | 作者或维护者。 |
| `apiVersion` | 整数 | 可选，默认宿主 API 版本 | 插件面向的 API 主版本；当前必须填写或解析为 **1**。与宿主 `PCL.PluginApi.ApiVersion` 不一致时拒绝安装。建议显式填写。 |
| `minHostVersion` | 字符串 | 可选，默认不限制 | 所需的最低宿主版本，如 `2.15.0`。宿主 `VersionNumber` 低于该值时拒绝安装。 |
| `entryAssembly` | 字符串 | **必需** | 入口 DLL 相对于插件包根目录的路径，例如 `Contoso.Hello.dll`。不能为空、不能是绝对路径，也不能包含 `..`。 |
| `entryType` | 字符串 | **必需** | 入口类型的完整名称，必须包含项目根命名空间，例如 `Contoso.Hello.PluginEntry`。该类型须实现 `PCL.IPclPlugin`，并提供可访问的无参数构造函数。 |
| `autoEnable` | 布尔值 | 可选，默认 `false` | **首次安装**时是否自动启用。已有 `plugins.json` 启用选择优先，修改后续包的此字段不会覆盖用户选择。 |
| `dependencies` | 对象数组 | 可选，默认空数组 | 插件依赖列表。每项包含下表的 `id` 和 `minVersion`。 |

依赖项字段：

| 字段 | 类型 | 必需性 / 默认值 | 说明 |
| --- | --- | --- | --- |
| `id` | 字符串 | **必需** | 被依赖插件的 `id`，不能为空。 |
| `minVersion` | 字符串 | 可选，默认不限制版本 | 被依赖插件的最低版本。已安装版本低于此值时，当前插件进入 `MissingDependency`。 |

清单读取还会校验包结构和入口路径。无效 JSON、无效版本、缺少必要条目、非法 `id` 或不兼容 API 都会导致安装失败，具体原因见宿主日志。

## 3. 运行目录与持久化

以下目录均相对于启动器所在目录。插件根目录为 `PCL/Plugins/`：

```text
PCL/
└─ Plugins/
   ├─ Contoso.Hello.jar
   ├─ plugins.json
   ├─ .runtime/
   │  └─ Contoso.Hello/
   │     └─ 1.0.0-包SHA256前12位/
   │        ├─ plugin.json
   │        ├─ Contoso.Hello.dll
   │        └─ ...
   └─ .data/
      └─ Contoso.Hello/
         └─ 插件自己的持久化文件
```

- **`PCL/Plugins/`**：只扫描此目录顶层的 `*.jar`，不会递归扫描子目录中的插件包。普通用户不需要手工操作文件：启动器标题栏的「插件」页与「启动」「下载」「设置」并列，可直接安装 `.jar`、启用/停用、卸载、重新扫描和打开插件目录；该页管理的是 PCL-DeFS 启动器插件，不是 Minecraft 模组。
- **`plugins.json`**：保存各插件的启用选择，格式版本当前为 1。例如：

  ```json
  {
    "schemaVersion": 1,
    "enabled": {
      "Contoso.Hello": true
    }
  }
  ```

  该选择优先于后续包的 `autoEnable`。运行时不会监听此文件的热修改；如需手工调整，应先完全退出启动器，再编辑并重新启动。
- **`.runtime/{id}/{version-hash}/`**：宿主管理的解包目录。目录名由插件版本和包 SHA-256 前 12 位组成。不要在此目录写入需要长期保留的数据，也不要手工替换其中的 DLL。
- **`.data/{id}/`**：通过 `IPluginContext.DataDirectory` 获得的插件数据目录，宿主创建后可读写。配置、缓存或资源应由插件自行组织；敏感数据不应明文保存。
- `IPluginContext.PluginDirectory` 指向当前 **解包后的运行目录**，不是 JAR 所在目录。
- 卸载插件不会自动删除 `.data/{id}`，热替换留下的旧解包目录也不应由正在运行的插件擅自清理。确需彻底删除数据时，应先退出启动器并备份后手工处理。

包 SHA-256 在这里用于识别包内容变化和生成隔离目录，**不是发布者签名，也不证明插件可信**。

## 4. API 1 参考

插件引用 `PCL-DeFS.exe` 后，可使用 `PCL` 根命名空间中的公共契约。

### 4.1 API 版本

```vb
PCL.PluginApi.ApiVersion = 1
```

`ApiVersion` 是常量。插件与宿主仅在 API 主版本一致时兼容。

### 4.2 IPclPlugin

```vb
Public Interface IPclPlugin
    Sub OnLoad(Context As IPluginContext)
    Sub OnEnable()
    Sub OnDisable()
    Sub OnUnload()
End Interface
```

这是入口类型必须实现的标准生命周期，详见第 6 节。

### 4.3 IPluginContext

| 成员 | 用途 |
| --- | --- |
| `Manifest As PluginManifest` | 当前插件的只读清单信息。 |
| `HostVersion As Version` | 当前宿主版本。 |
| `PluginDirectory As String` | 当前版本解包后的运行目录。 |
| `DataDirectory As String` | 当前插件持久化数据目录 `.data/{id}`。 |
| `GetSetting(Of T)(Key As String) As T` | 读取宿主设置。 |
| `SetSetting(Key As String, Value As Object)` | 写入宿主设置。 |
| `RegisterPageExtension(ExtensionPoint, Factory)` | 注册页面扩展工厂。 |
| `RegisterBackgroundService(Service)` | 注册背景服务。 |
| `RegisterVisualEffectService(Service)` | 注册视觉效果服务。 |
| `ClearExtensions()` | 撤销当前插件注册的扩展和服务。宿主在停用、卸载时也会自动调用。 |
| `LogInfo(Message)` | 写入带插件 ID 的普通日志。 |
| `LogError(Message, Optional Exception)` | 写入带插件 ID 的错误日志。 |
| `SettingChanged(Key)` | 设置变化事件；仅向处于启用状态的插件分发。 |

设置使用宿主共享键空间，键名应带插件 ID 前缀，例如 `Contoso.Hello.Enabled`，避免与宿主或其他插件冲突。不要记录密码、令牌、个人隐私或其他秘密。

### 4.4 IPluginPageExtension

```vb
Function CreateContent() As UIElement
Sub Refresh()
Sub Detach()
```

- `CreateContent` 创建要注入宿主的 WPF 内容。
- 页面需要刷新时宿主调用 `Refresh`。
- 页面卸载、插件停用或卸载时调用 `Detach`；应在此释放事件订阅、计时器和内容引用。

当前公开页面扩展点为：

| 扩展点 | 注入位置 |
| --- | --- |
| `Setup.UI` | 设置页的个性化界面扩展区域。 |

注册方式：

```vb
Context.RegisterPageExtension("Setup.UI", Function() New HelloExtension())
```

### 4.5 IPluginBackgroundService

```vb
Sub Refresh(IsHint As Boolean, ApplyBackground As Boolean)
Sub StopBackground()
```

用于提供图片、视频等背景。`Refresh` 参数分别表示是否显示提示、是否实际应用背景；`StopBackground` 应停止播放并释放相关资源。当前宿主使用最后注册的背景服务；没有任何插件背景服务时回退到内置机械动力背景，因此卸载背景插件后不应留下失效的媒体引用。

### 4.6 IPluginVisualEffectService

```vb
Sub RefreshAll()
Sub ApplyToTree(Root As DependencyObject)
Sub ResetAll()
```

用于液态玻璃、悬浮等全局视觉效果。`RefreshAll` 刷新整个界面，`ApplyToTree` 处理指定视觉树，`ResetAll` 必须能撤销插件施加的效果。插件被停用或卸载时，宿主撤销注册前会调用 `ResetAll`。

### 4.7 主窗口与主题门面

`PCL.ModPlugins` 还提供以下公开方法，让插件无需直接访问宿主内部控件：

| 方法 | 用途 |
| --- | --- |
| `ApplyBackgroundImage(Address As String)` | 应用图片背景；支持文件路径和宿主内置资源的 Pack URI。 |
| `ApplyBackgroundVideo(Address As String)` | 应用并播放视频背景。 |
| `ClearBackgroundMedia()` | 停止并清除当前视频背景。 |
| `SetSidebarGlass(Enabled As Boolean)` | 切换侧栏液态玻璃画刷。 |
| `RefreshCustomTheme()` | 按当前宿主设置重新应用自定义主题色。 |
| `RefreshBackground(IsHint, ApplyBackground)` | 调用当前已注册的背景服务；没有服务时回退默认背景。 |
| `RefreshVisualEffects()` / `ApplyVisualEffects(Root)` | 刷新全部视觉效果，或应用到指定视觉树。 |

这些方法同样遵守 UI 线程约定。主窗口尚未加载时，背景和侧栏方法会安全返回，插件应在宿主刷新回调中应用效果，而不是在后台线程直接操作 WPF 控件。

## 5. 线程约定

**所有插件生命周期方法、扩展创建与刷新、服务回调以及 `PluginRuntime` 的公开生命周期操作均按 UI 线程约定使用。**

- 在 `OnLoad`、`OnEnable`、`OnDisable`、`OnUnload` 中可以直接执行轻量 UI 操作，但不要进行长时间下载、解压或计算，以免阻塞启动器。
- 插件自行创建的后台线程、任务或定时器回调**不会自动切换到 UI 线程**。访问 WPF 元素、注册扩展或调用要求 UI 线程的宿主能力前，应通过插件保存的 WPF `Dispatcher` 切回 UI 线程。
- 后台任务必须支持停止；在 `OnDisable` 或 `OnUnload` 中取消任务、停止计时器并退订事件，避免回调已释放对象。
- 文件监视器只负责发出重扫请求，实际重扫会切回宿主 UI 线程，插件不应直接调用监视器内部实现。

## 6. 生命周期、状态与依赖顺序

### 6.1 生命周期

| 方法 | 调用时机 | 插件应执行的工作 |
| --- | --- | --- |
| `OnLoad(Context)` | 入口程序集加载、入口实例创建之后 | 保存上下文，读取清单和目录，注册必要事件。不要假设依赖已经启用。 |
| `OnEnable()` | 所有依赖均已启用，且 `OnLoad` 成功之后 | 注册页面、背景、视觉效果，启动工作。失败时应抛出异常并尽量清理。 |
| `OnDisable()` | 手动停用、依赖链停用、热更新或卸载之前 | 停止工作，撤销事件、计时器和任务，释放扩展资源。宿主随后会清除该插件的扩展注册。 |
| `OnUnload()` | 热覆盖、热删除或卸载、从运行记录移除之前 | 释放插件实例仍持有的剩余资源。不要在此访问已撤销的 UI 扩展。 |

首次启用顺序为：**递归启用依赖 → 加载入口程序集并创建实例 → `OnLoad` → `OnEnable` → 状态变为 `Enabled`**。

启用失败时，运行时会尝试调用 `OnDisable` 和清除扩展，记录错误并进入 `Failed`。正常进程关闭会停用启用的插件；不要依赖进程退出时一定执行 `OnUnload`，重要状态应在 `OnDisable` 中保存。

### 6.2 插件状态

`PCL.PluginState` 包含：

| 状态 | 含义 |
| --- | --- |
| `Installed` | 已发现并安装，尚未加载实例。 |
| `Loading` | 正在加载程序集、创建实例或执行 `OnLoad`。 |
| `Loaded` | `OnLoad` 已成功，但尚未完成启用。 |
| `Enabled` | `OnEnable` 成功，扩展和服务可以工作。 |
| `Disabled` | 实例已加载，但插件已停用。 |
| `MissingDependency` | 依赖缺失、未启用或版本过低；`LastError` 给出原因。 |
| `Failed` | 加载或启用失败；查看 `LastError` 与宿主日志。 |

状态变化通过 `PluginRuntime.PluginStateChanged` 通知。`LastError` 是诊断信息，不应被当作稳定的机器可解析协议。

### 6.3 依赖拓扑与启停顺序

- 依赖关系构成有向图，插件必须先启用自己的全部依赖。
- `PluginRuntime.ResolveOrder` 返回**依赖在前、依赖方在后**的拓扑结果；缺失依赖、版本过低或循环依赖会抛出异常。
- 启用插件时，运行时递归启用依赖，然后加载并启用当前插件。例如 `A → B → C`（A 依赖 B，B 依赖 C）的启用顺序为 `C、B、A`。
- 停用某个被依赖插件时，运行时会先递归停用仍启用且依赖它的插件，再停用目标插件。例如上述链条停用 C 时，顺序为 `A、B、C`。
- 循环依赖无法启用，应在清单设计阶段消除。依赖版本检查针对已安装插件的 `version`，不是程序集文件版本。
- 关闭宿主时会停止监听并停用启用的插件；插件自身仍应保证 `OnDisable` 幂等，以应对重复清理路径。

## 7. 安装、热插拔与卸载

### 7.1 文件操作方式

| 场景 | 操作与结果 |
| --- | --- |
| 启动时安装 | 将 `.jar` 放入 `PCL/Plugins/` 后启动。宿主扫描、校验并解包，首次安装按 `autoEnable` 决定是否启用。 |
| 热新增 | 启动器运行时，将完整插件包放入根目录。监视器在文件变化后约 600 ms 请求 UI 线程重扫。建议先复制为 `.tmp` 等同目录临时文件，完成后再改名为 `.jar`，避免宿主读到未写完的包。 |
| 热覆盖 | 用同 ID、内容有变化的新包覆盖旧 JAR。重扫后旧实例先逻辑停用、卸载，再从新的 `版本-哈希` 目录建立记录。原启用选择保留，首次安装才使用 `autoEnable`。建议同时提升插件版本和程序集版本。 |
| 热删除 | 删除根目录中的 JAR，重扫后对应实例逻辑停用、卸载并移出运行记录。启用选择会保留，之后重新放入同 ID 插件时可沿用。 |
| 启动器退出后删除 | 完全退出后删除 JAR，下次启动不再发现该插件。`.data` 不会因此自动删除。 |

相同 ID 且包签名相同的记录在重扫时会跳过。若同一目录中存在多个相同 ID 的 JAR，宿主按文件名不区分大小写排序，仅使用排序靠前的第一个有效包并忽略后续重复项；重复扫描不会在包之间来回切换。若拷贝后未触发事件，可在确认文件完整后重命名触发重扫，或重启启动器。热覆盖涉及程序集驻留限制，务必阅读第 8 节。

### 7.2 宿主与管理器可调用的方法

以下方法位于 `PluginRuntime`，**调用方必须遵守 UI 线程约定**：

| 方法 | 行为 |
| --- | --- |
| `InstallPackage(PackagePath)` | 校验并安装或覆盖包；包不在插件根目录时会先复制进去。沿用已有启用选择，首次安装读取 `autoEnable`，需要时立即启用。 |
| `Scan()` | 扫描新增、覆盖、删除的顶层 JAR，重放应启用插件的依赖顺序并保存状态。 |
| `EnablePlugin(Id)` | 按依赖顺序加载并启用插件，持久化启用选择。 |
| `DisablePlugin(Id)` | 先停用依赖方，再停用目标并持久化选择。 |
| `UninstallPlugin(Id)` | 逻辑停用和卸载、删除 JAR、移除运行记录及启用选择，然后保存配置；不会自动删除 `.data/{id}`。 |
| `Shutdown()` | 停止文件监听并停用启用的插件。宿主退出时调用，插件通常不应主动调用。 |

`ResolveOrder(Manifests)` 可用于管理器预先检查拓扑；`Plugins` 可读取当前运行记录。普通插件不应通过运行时管理接口随意停用其他插件，管理操作应由可信的宿主功能执行。

## 8. .NET Framework 卸载与热替换限制

这一点必须在插件设计和发布说明中明确：

> PCL-DeFS 基于 .NET Framework 默认 `AppDomain`。默认 `AppDomain` 不能单独物理卸载已经加载的程序集。插件的“卸载”和“热替换”是**逻辑卸载与逻辑替换**，不是 CLR 层面的程序集物理卸载。

运行时通过 `Assembly.Load(File.ReadAllBytes(...))` 加载入口及依赖，虽然这样能减少 DLL 文件占用，但不等于创建了可卸载的独立 `AppDomain` 或可收集程序集上下文。

- `OnUnload` 后，宿主会清除实例、上下文和扩展引用，但**旧程序集映像及其元数据仍驻留在默认 AppDomain 中，随整个启动器进程退出才释放**。
- 热覆盖会创建新的运行记录和解包目录，但旧代码不会从进程中删除；同名依赖解析还可能复用已加载程序集。
- 多次热更新可能增加内存占用，并引入程序集身份、静态状态或类型冲突。发布更新时应提高插件版本与程序集版本，避免携带互相冲突的依赖。
- 对生产环境的正式升级、入口类型或依赖发生较大变化、热更新后仍表现旧行为等情况，**应完全重启启动器**，以获得干净进程。不要把热替换描述为物理卸载或保证所有旧代码立即消失。

## 9. 最小 VB.NET 插件

### 9.1 创建项目

1. 在 Visual Studio 2022 中创建 **VB.NET 类库（.NET Framework）**，目标框架选择 **.NET Framework 4.8**。
2. 将程序集名和根命名空间均设为 `Contoso.Hello`；程序集版本可设为 `1.0.0.0`。
3. 添加对发布版 `PCL-DeFS.exe` 的引用，并在引用属性中设置 **复制本地（Copy Local）= False**。插件运行时使用宿主进程中的 API，不能把宿主 EXE 打进插件包。
4. 按需添加 WPF 引用：`PresentationFramework`、`PresentationCore`、`WindowsBase`、`System.Xaml`。
5. 私有第三方依赖必须兼容 .NET Framework 4.8，并随包放在 JAR 根目录；不要重复打包 `PCL-DeFS.exe` 或宿主自带程序集。

项目文件中的宿主引用通常类似：

```xml
<Reference Include="PCL-DeFS">
  <HintPath>..\References\PCL-DeFS.exe</HintPath>
  <Private>False</Private>
</Reference>
```

### 9.2 入口代码

在项目根命名空间为 `Contoso.Hello` 的前提下，以下入口类型全名为 `Contoso.Hello.PluginEntry`：

```vb
Imports System.Windows
Imports System.Windows.Controls
Imports PCL

Public NotInheritable Class PluginEntry
    Implements IPclPlugin

    Private Context As IPluginContext

    Public Sub OnLoad(Context As IPluginContext) Implements IPclPlugin.OnLoad
        Me.Context = Context
        Me.Context.LogInfo("示例插件已加载")
    End Sub

    Public Sub OnEnable() Implements IPclPlugin.OnEnable
        Context.RegisterPageExtension("Setup.UI", Function() New HelloExtension())
        Context.LogInfo("示例插件已启用")
    End Sub

    Public Sub OnDisable() Implements IPclPlugin.OnDisable
        Context.ClearExtensions()
        Context.LogInfo("示例插件已停用")
    End Sub

    Public Sub OnUnload() Implements IPclPlugin.OnUnload
        Context = Nothing
    End Sub
End Class

Public NotInheritable Class HelloExtension
    Implements IPluginPageExtension

    Private Content As TextBlock

    Public Function CreateContent() As UIElement Implements IPluginPageExtension.CreateContent
        Content = New TextBlock With {
            .Text = "Hello，来自 Contoso.Hello！",
            .TextWrapping = TextWrapping.Wrap
        }
        Return Content
    End Function

    Public Sub Refresh() Implements IPluginPageExtension.Refresh
    End Sub

    Public Sub Detach() Implements IPluginPageExtension.Detach
        Content = Nothing
    End Sub
End Class
```

对应的 `plugin.json` 使用第 2 节示例，但此最小插件实际上不依赖个性化插件，因此应将 `dependencies` 改为：

```json
"dependencies": []
```

编译 Release 后，确认输出中存在 `Contoso.Hello.dll`，再按下一节组包。

## 10. 打包与部署

1. 新建临时组包目录，将编译好的入口 DLL、必要的私有依赖和资源放入其中。
2. 在组包目录根层写入 `plugin.json`，核对 `entryAssembly` 与实际 DLL 文件名、`entryType` 与编译后的完整类型名一致。
3. 创建 `META-INF/MANIFEST.MF`。最终应压缩**组包目录中的内容**，不要多包一层同名文件夹。
4. 使用任意 ZIP 工具生成压缩包，再把扩展名改为 `.jar`。例如 PowerShell：

   ```powershell
   Compress-Archive -Path .\package\* -DestinationPath .\Contoso.Hello.zip
   Rename-Item -Path .\Contoso.Hello.zip -NewName .\Contoso.Hello.jar
   ```

5. 用 ZIP 工具重新打开成品，确认 `plugin.json` 和 `META-INF/MANIFEST.MF` 位于规定位置，入口 DLL 能按清单路径找到。
6. 备份原有同 ID 插件及需要保留的数据，将 JAR 复制到 `PCL/Plugins/`。运行中复制会触发热安装；首次安装且 `autoEnable` 为 `true` 时自动启用。
7. 检查插件状态和宿主日志。正式更新或出现异常时，完全重启启动器后再验证，以排除默认 AppDomain 中旧程序集的影响。

## 11. 排错

| 现象 / 错误 | 检查项 |
| --- | --- |
| 提示缺少 `plugin.json` 或 `META-INF/MANIFEST.MF` | 两者是否实际位于包内规定路径；是否多压缩了一层目录；文件名大小写和扩展名是否正确。 |
| JSON 或版本解析失败 | 使用 JSON 校验工具；`version`、`minHostVersion`、`minVersion` 必须符合 `System.Version` 格式。 |
| API 不兼容 | `apiVersion` 必须为 1；旧 API 插件需要迁移后重新编译。 |
| 宿主版本过低 | 升级宿主，或在确认未使用新能力后调整 `minHostVersion`；不要通过虚报宿主版本绕过兼容问题。 |
| 入口程序集不存在 | `entryAssembly` 是否为包内相对路径；DLL 是否已包含；路径是否含 `..` 或错误目录。 |
| 找不到入口类型 / 未实现 `IPclPlugin` | 核对 VB 项目根命名空间后的完整类型名；类应为 Public、具有无参数构造函数，并实现 `PCL.IPclPlugin`。 |
| DLL 或依赖加载失败 | 确认目标框架为 .NET Framework 4.8、依赖位于包根目录且位数兼容；不要混用仅适用于 .NET Core/.NET 5+ 的程序集。必要时重启排除旧程序集冲突。 |
| 状态为 `MissingDependency` | 按 `LastError` 安装并启用依赖，检查 `minVersion` 和循环依赖。 |
| 状态为 `Failed` | 查看 `LastError` 与宿主日志；重点检查构造函数、`OnLoad`、`OnEnable` 中的异常。 |
| 设置了 `autoEnable` 却未启用 | `plugins.json` 中已有选择优先；完全退出后检查该 ID 的布尔值，并确认插件不是 `MissingDependency` 或 `Failed`。 |
| 页面没有扩展内容 | 确认插件已启用、在 `OnEnable` 注册 `Setup.UI`、工厂能创建 `IPluginPageExtension`，并检查 `CreateContent` 是否返回有效 WPF 元素。 |
| 热更新未识别 | 确认是根目录顶层 `.jar`、文件已写完且内容哈希改变；等待防抖重扫，或重启启动器。 |
| 热更新后仍像旧版本 | 这是默认 AppDomain 程序集驻留或同名依赖复用的典型风险。提升版本并完全重启；不要把逻辑卸载当成物理卸载。 |

插件应通过 `Context.LogInfo`、`Context.LogError` 记录可定位的信息，但不要输出秘密或完整隐私路径。发布前至少验证首次安装、手动停用、依赖缺失、热覆盖、热删除、完全卸载和重启恢复。

## 12. 安全建议

- 插件 DLL 与宿主在**同一进程、同一默认 AppDomain、同等信任级别**运行，没有安全沙箱。只安装来源可信且经过验证的插件；一个恶意插件可以访问用户文件、网络和宿主能力。
- 发布者宜提供可信下载渠道、版本记录和独立 SHA-256；用户应核对来源与哈希。运行时目录名中的哈希仅用于变更识别，不能替代发布者签名。
- 审查第三方依赖及其许可证，保持依赖版本一致，避免把宿主程序集、调试符号、源码、个人配置或签名私钥打进包中。
- 不要在 `plugin.json`、宿主设置、日志或普通文本数据中存放令牌、密码等秘密。确需保存敏感信息时使用操作系统提供的保护机制，并限制数据目录访问权限。
- 对下载内容、插件数据和用户输入进行校验；网络操作使用 TLS、合理超时与大小限制。组包前检查异常路径、超大文件和压缩炸弹，避免消耗过多磁盘或内存。
- 卸载前备份 `.data/{id}` 中仍需保留的数据；清理 `.runtime` 或旧版本目录应在启动器完全退出后进行，避免破坏当前实例。
- 插件应做到停用即停止后台工作、撤销 UI 与事件副作用，卸载即释放剩余资源；这既是稳定性要求，也能降低数据被破坏的风险。
