# WPF 从基础到真实产品架构与演进指南

> 面向希望从“能写界面”成长为能够设计、交付和长期维护 Windows 桌面产品的 .NET 开发者。
>
> 本文按 **基础机制 -> 工程能力 -> 产品架构 -> 运行与交付 -> 技术选型** 的路径展开。设计 WPF 产品时，应始终回答四个问题：状态由谁拥有、依赖如何替换、耗时操作在哪里执行、产品如何安全地升级和诊断。

## 目录

1. [WPF 的定位与适用边界](#一wpf-的定位与适用边界)
2. [核心基础：XAML、依赖属性与布局](#二核心基础xaml依赖属性与布局)
3. [数据绑定、命令与 MVVM](#三数据绑定命令与-mvvm)
4. [资源、样式、模板与主题](#四资源样式模板与主题)
5. [线程、异步与 UI 响应](#五线程异步与-ui-响应)
6. [进阶机制与可复用控件](#六进阶机制与可复用控件)
7. [真实产品的推荐架构](#七真实产品的推荐架构)
8. [产品案例：设备运维工作台](#八产品案例设备运维工作台)
9. [数据、网络、配置与安全](#九数据网络配置与安全)
10. [性能、可访问性与本地化](#十性能可访问性与本地化)
11. [测试、诊断与可观测性](#十一测试诊断与可观测性)
12. [部署、升级与运维](#十二部署升级与运维)
13. [常见反模式与改进路径](#十三常见反模式与改进路径)
14. [WPF 的未来发展趋势](#十四wpf-的未来发展趋势)
15. [替代方案与选型矩阵](#十五替代方案与选型矩阵)
16. [落地路线图与面试自检](#十六落地路线图与面试自检)

---

## 一、WPF 的定位与适用边界

WPF（Windows Presentation Foundation）是运行在 Windows 上的 .NET 桌面 UI 框架。它以 XAML 描述界面，以 .NET 对象模型组织行为，并通过保留模式的可视树、数据绑定、样式模板和硬件加速渲染构建富客户端应用。

### 1. WPF 的优势

| 能力 | 产品价值 | 典型场景 |
| --- | --- | --- |
| 成熟的数据绑定和 MVVM | 将复杂业务状态与视图解耦，便于测试和多人协作 | 交易、医疗、工业控制、运营后台 |
| 强大的控件模板 | 在不复制业务代码的前提下深度定制外观和交互 | 品牌化客户端、专业工具软件 |
| Windows 集成 | 可使用 COM、Win32、打印、证书、企业身份、文件系统和硬件 SDK | 扫码枪、打印机、仪器、Office 集成 |
| 长生命周期与兼容性 | 适合需要多年维护的企业软件 | 制造执行、金融内控、政企桌面系统 |
| .NET 生态 | 可复用 ASP.NET Core、DI、日志、配置和测试体系 | 前后端共享领域模型和基础设施 |

### 2. WPF 的边界

- WPF 仅支持 Windows，不是跨平台桌面框架。
- WPF 的默认视觉语言偏传统，需要自行构建设计系统或引入经过评估的控件库。
- 浏览器式动态布局、移动端优先体验和跨端一致性并不是其优势。
- XAML 数据绑定大多在运行时解析；错误路径、拼写和数据上下文错误若无测试和诊断，可能延后暴露。
- UI 线程模型严格。任何阻塞 UI 线程的 I/O、锁等待或大规模集合更新都会直接影响用户体验。

### 3. 何时选择 WPF

选择 WPF，通常意味着以下条件至少有两到三项成立：

1. 产品明确只运行于 Windows，且需要深度调用本地能力、硬件或企业环境。
2. 界面复杂、数据密度高、长期维护成本比首发速度更重要。
3. 团队已有 C#/.NET 能力，并愿意遵守 MVVM、分层和自动化测试纪律。
4. 产品需要离线可用、局域网部署、受控升级或内网身份集成。

不宜仅因“Windows 客户端”就默认选 WPF。若产品必须覆盖 macOS/Linux，应优先评估 Avalonia 或其他跨平台路线；若需要与现代 Windows 设计语言和新系统能力深度同步，应评估 WinUI 3；若本质是信息展示和流程表单，Web 技术也可能更经济。

---

## 二、核心基础：XAML、依赖属性与布局

### 1. XAML 与代码隐藏的职责

XAML 是声明 UI 对象图的语言，不是“只能放样式的 HTML”。它能创建对象、设置属性、声明绑定、资源、事件和模板。代码隐藏（`*.xaml.cs`）不是禁区，但应保持轻量：

- 允许：纯视觉交互、焦点控制、动画启动、与控件生命周期强绑定且无法抽象的逻辑。
- 不应承担：业务规则、HTTP 调用、数据库访问、跨页面共享状态、权限判断。
- 一个实用准则：若逻辑需要单元测试，或可能被另一视图复用，就应优先进入 ViewModel、应用服务或领域层。

```xml
<Window x:Class="Operations.Client.Views.DeviceListView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="设备列表" Height="720" Width="1100">
    <Grid Margin="24">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto" />
            <RowDefinition Height="*" />
        </Grid.RowDefinitions>

        <Button Content="刷新"
                Command="{Binding RefreshCommand}"
                IsEnabled="{Binding CanRefresh}" />

        <DataGrid Grid.Row="1"
                  Margin="0,16,0,0"
                  ItemsSource="{Binding Devices}"
                  SelectedItem="{Binding SelectedDevice, Mode=TwoWay}"
                  IsReadOnly="True" />
    </Grid>
</Window>
```

### 2. 依赖属性：WPF 可组合性的基础

依赖属性（Dependency Property，DP）由 WPF 属性系统托管，而不是普通 CLR 自动属性。它支持默认值、样式、模板、动画、继承、绑定和变更通知等能力。

属性的最终值由多种来源按优先级决定。常见从高到低的来源可理解为：动画/强制值、控件本地值、模板触发器、样式触发器、样式 Setter、默认样式、元数据默认值。因此，“我在样式里设置了颜色但没有生效”通常不是渲染错误，而是更高优先级的本地值或触发器覆盖了它。

```csharp
public sealed class StatusBadge : Control
{
    public static readonly DependencyProperty StatusProperty =
        DependencyProperty.Register(
            nameof(Status),
            typeof(DeviceStatus),
            typeof(StatusBadge),
            new PropertyMetadata(DeviceStatus.Unknown));

    public DeviceStatus Status
    {
        get => (DeviceStatus)GetValue(StatusProperty);
        set => SetValue(StatusProperty, value);
    }

    static StatusBadge()
    {
        DefaultStyleKeyProperty.OverrideMetadata(
            typeof(StatusBadge),
            new FrameworkPropertyMetadata(typeof(StatusBadge)));
    }
}
```

使用 DP 的判断标准：只有当属性需要绑定、样式化、动画化、模板化或被父级继承时才考虑它。普通内部状态优先使用普通 CLR 属性，避免将属性系统变成通用状态容器。

### 3. 路由事件与命令

WPF 事件会沿可视树或逻辑树路由：

- 隧道路由（如 `PreviewMouseDown`）从根向目标元素传递，适合拦截或预处理。
- 冒泡路由（如 `MouseDown`）从目标元素向根传递，适合由容器统一处理子元素交互。
- 直接事件只通知事件源。

业务动作优先使用 `ICommand`，而非在每个按钮上写 Click 事件。命令可以绑定到 ViewModel，并基于 `CanExecute` 自动管理按钮是否可用；也可通过 `CommandParameter` 接收当前行对象。

### 4. 布局面板选择

| 面板 | 使用建议 | 常见误用 |
| --- | --- | --- |
| `Grid` | 表单、主页面、需要明确行列关系的区域 | 用大量嵌套 Grid 模拟绝对定位 |
| `StackPanel` | 单方向、少量、尺寸简单的元素 | 放大量可滚动项，导致无法有效虚拟化 |
| `DockPanel` | 工具栏、状态栏、左右停靠区域 | 复杂响应式页面的通用替代品 |
| `WrapPanel` | 标签、按钮组、尺寸不固定的卡片 | 展示上千条数据 |
| `Canvas` | 图编辑器、拓扑图、设计器等绝对坐标场景 | 普通业务表单布局 |
| `VirtualizingStackPanel` | 大列表的默认虚拟化宿主 | 被外层 `ScrollViewer` 包裹后丢失虚拟化 |

布局应描述关系，而非像素。固定宽高只适用于图标、工具栏按钮、图表画布或明确受控的区域；业务表单应使用 `Auto`、`*`、`MinWidth`、`MaxWidth` 和对齐属性，兼容 DPI 缩放、语言扩展和辅助功能字体。

---

## 三、数据绑定、命令与 MVVM

### 1. 数据绑定的四个关键点

1. **数据上下文**：`DataContext` 是绑定的默认源，通常由 View 注入对应的 ViewModel。
2. **变更通知**：属性变化需要 `INotifyPropertyChanged`，集合增删需要 `ObservableCollection<T>` 或能发出集合变更通知的集合。
3. **绑定方向**：显示字段通常是 `OneWay`，输入字段通常是 `TwoWay`，只初始化一次时可选 `OneTime`。
4. **更新时机**：`TextBox.Text` 默认失焦才回写源；实时校验时显式设置 `UpdateSourceTrigger=PropertyChanged`。

```csharp
public sealed partial class DeviceEditorViewModel : ObservableObject
{
    [ObservableProperty]
    private string deviceName = string.Empty;

    [ObservableProperty]
    private string endpoint = string.Empty;

    public bool CanSave =>
        !string.IsNullOrWhiteSpace(DeviceName) &&
        Uri.TryCreate(Endpoint, UriKind.Absolute, out _);

    partial void OnDeviceNameChanged(string value) => SaveCommand.NotifyCanExecuteChanged();

    partial void OnEndpointChanged(string value) => SaveCommand.NotifyCanExecuteChanged();

    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync(CancellationToken cancellationToken)
    {
        await Task.CompletedTask;
    }
}
```

上例使用 CommunityToolkit.Mvvm 减少 `INotifyPropertyChanged` 和命令样板代码。它不是 WPF 必需依赖；若团队不采用源生成器，也可以手写通知逻辑或使用现有 MVVM 基础库。关键不是库名，而是 ViewModel 能独立于 View 被构造和测试。

### 2. MVVM 的职责划分

| 层 | 可以做什么 | 不应做什么 |
| --- | --- | --- |
| View | 展示、视觉状态、控件模板、有限的视觉行为 | 承担领域规则或直接访问数据库 |
| ViewModel | 暴露可绑定状态、协调命令、输入校验、导航状态 | 直接依赖 `Window`、`MessageBox`、`DataGrid` 等具体 UI 对象 |
| Model/DTO | 数据结构、传输契约 | 夹带视图行为和控件状态 |
| Application Service | 用例编排、事务边界、权限和调用端口 | 依赖 WPF 类型 |
| Domain | 业务规则、实体和值对象 | 依赖 EF Core、HTTP 或 UI |
| Infrastructure | HTTP、数据库、文件、设备 SDK 的具体实现 | 向上泄露第三方实现细节 |

### 3. 验证输入而不是只禁用提交按钮

禁用提交按钮只能改善操作体验，不能取代校验。校验至少应分为三层：

- 字段格式校验：必填、长度、数值和 URI 格式，可由 ViewModel 即时提供。
- 用例校验：设备名称是否重复、当前操作者是否有权限，由应用服务执行。
- 领域规则：例如“运行中的设备不能直接删除”，由领域模型或领域服务保证。

对于可编辑表单，可使用 `INotifyDataErrorInfo` 将字段错误与业务校验错误投影到界面。后端或领域层仍须重复关键校验，不能信任客户端状态。

### 4. 避免绑定常见陷阱

- 不要将 ViewModel 设置为 `DataContext = this` 的窗口代码和 DI 创建的 ViewModel 混用。选择一种组合根策略并保持一致。
- 不要在属性 getter 内部执行网络调用、数据库查询或改变状态；绑定引擎可多次读取 getter。
- 不要在 ViewModel 中构造 `new HttpClient()`、`new Window()` 或读取静态全局单例，这会破坏测试性和生命周期管理。
- 打开绑定诊断：开发环境可为关键绑定设置 `PresentationTraceSources.TraceLevel=High`，并在输出窗口定位路径错误。

---

## 四、资源、样式、模板与主题

### 1. 四种外观复用层级

| 机制 | 解决的问题 | 示例 |
| --- | --- | --- |
| 资源 `ResourceDictionary` | 复用颜色、尺寸、画笔和转换器 | `BrandAccentBrush` |
| `Style` | 复用一组属性和值 | 所有表单标签的间距和字体 |
| `ControlTemplate` | 重定义已有控件的视觉树 | 自定义 `Button`、`ToggleButton` 外观 |
| 自定义控件 | 复用行为、状态和默认模板 | 状态徽标、时间范围选择器 |

`DataTemplate` 用于“数据对象应该如何显示”，`ControlTemplate` 用于“控件自身由哪些视觉元素构成”。把两者混淆会造成模板难以维护。

### 2. 设计系统的最小组成

真实产品不应在每个页面硬编码 `#2F80ED`、`Margin="8"` 或字体大小。至少建立：

- 语义色：文本、次要文本、边框、表面、危险、警告、成功、焦点。
- 尺寸令牌：间距、圆角、行高、控件高度、图标尺寸。
- 字体层级：页面标题、区块标题、正文、辅助文本、等宽数据文本。
- 基础控件样式：按钮、输入框、下拉框、表格、菜单、对话框、错误状态。
- 明确的浅色/深色资源字典，而不是在业务 XAML 内写大量触发器。

```xml
<ResourceDictionary xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
                    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Color x:Key="Color.Surface">#FFF8FAFC</Color>
    <Color x:Key="Color.Text.Primary">#FF172033</Color>
    <Color x:Key="Color.Action.Primary">#FF006A6A</Color>

    <SolidColorBrush x:Key="Brush.Surface" Color="{StaticResource Color.Surface}" />
    <SolidColorBrush x:Key="Brush.Text.Primary" Color="{StaticResource Color.Text.Primary}" />
    <SolidColorBrush x:Key="Brush.Action.Primary" Color="{StaticResource Color.Action.Primary}" />

    <x:Double x:Key="Space.2">8</x:Double>
    <x:Double x:Key="Space.4">16</x:Double>
</ResourceDictionary>
```

生产级主题切换应替换主题字典中的资源对象，并确保业务视图引用语义资源。对于需要运行时更新的颜色或画笔使用 `DynamicResource`；不会切换的静态资源使用 `StaticResource`，降低查找和更新开销。

---

## 五、线程、异步与 UI 响应

### 1. Dispatcher 与 UI 线程规则

WPF 控件具有线程亲和性：创建控件的 UI 线程必须访问它。后台线程不能直接修改 `ObservableCollection`、控件属性或依赖属性。

正确模型是：后台线程执行 I/O 或 CPU 密集工作，完成后回到 UI 上下文提交一个小的、原子化的界面状态更新。不要为了绕过异常而在任何地方随意调用 `Dispatcher.Invoke`；先明确“数据归谁更新、在哪个线程更新”。

```csharp
public sealed partial class DeviceListViewModel : ObservableObject
{
    private readonly IDeviceQueryService deviceQueryService;

    public ObservableCollection<DeviceRowViewModel> Devices { get; } = [];

    public DeviceListViewModel(IDeviceQueryService deviceQueryService)
    {
        this.deviceQueryService = deviceQueryService;
    }

    [RelayCommand]
    private async Task RefreshAsync(CancellationToken cancellationToken)
    {
        var devices = await deviceQueryService.GetDevicesAsync(cancellationToken);

        Devices.Clear();
        foreach (var device in devices)
        {
            Devices.Add(DeviceRowViewModel.From(device));
        }
    }
}
```

若命令从 UI 线程开始且下游异步 API 正确使用 `await`，默认会回到 WPF 同步上下文，因此上例可直接更新集合。若主动使用 `ConfigureAwait(false)`、事件来自后台 SDK，或运行于后台服务，则需要在 UI 边界显式调度更新。

### 2. 必须避免的异步错误

| 错误 | 后果 | 改进 |
| --- | --- | --- |
| `.Result` / `.Wait()` 等待异步任务 | UI 死锁或线程池饥饿 | 沿调用链使用 `async/await` |
| 用 `Task.Run` 包装所有异步 I/O | 无意义占用线程池线程 | 直接调用真正异步的 I/O API |
| 忽略 `CancellationToken` | 页面关闭后仍继续请求和更新 | 将取消令牌传到 HTTP、数据库和设备调用 |
| 并发刷新不做版本控制 | 旧响应覆盖新筛选结果 | 取消前一请求或比较请求版本 |
| 每条数据单独 `Dispatcher.Invoke` | UI 队列拥塞和卡顿 | 批处理、节流、虚拟化 |

### 3. CPU 密集型任务

图像分析、报表汇总、压缩、加密等 CPU 密集任务可以在后台执行，但必须有取消、进度和异常边界：

```csharp
var progress = new Progress<int>(value => ExportProgress = value);
await Task.Run(
    () => reportExporter.Export(request, progress, cancellationToken),
    cancellationToken);
```

对于实时遥测，先在后台缓冲和聚合，再按固定频率（例如每秒数次）提交 UI 更新。将每一条原始消息都绑定到界面，通常会使渲染和垃圾回收成为瓶颈。

---

## 六、进阶机制与可复用控件

### 1. 用户控件与自定义控件如何选择

| 类型 | 特征 | 适用场景 |
| --- | --- | --- |
| `UserControl` | XAML 组合已有控件，封装快 | 业务页面片段、稳定的复合组件 |
| 自定义控件 `Control` | 行为代码与默认模板分离，外观可换 | 设计系统控件、需多主题支持的通用组件 |
| 附加属性/附加行为 | 为现有控件增加可复用能力 | 自动聚焦、滚动到底部、快捷键映射 |
| `Behavior` | 将可交互行为声明式附着到 View | 拖放、事件转命令、视觉动画 |

若控件的使用者需要替换视觉结构或主题，请创建自定义控件；若只是将固定布局打包复用，使用 `UserControl` 更简单。不要为了“架构纯粹”把每一个小组件都做成自定义控件。

### 2. 可视树、逻辑树与内存泄漏

WPF 中的对象关系不只有父子控件：数据绑定、事件订阅、计时器、命令、静态缓存、消息总线都会持有引用。常见泄漏来源包括：

- 长生命周期服务订阅了 ViewModel 事件，但页面关闭时没有取消订阅。
- `DispatcherTimer`、`System.Timers.Timer`、`FileSystemWatcher` 在视图离开后仍运行。
- 静态事件、全局消息总线持有短生命周期对象。
- 弹窗关闭后仍被导航缓存、任务闭包或集合引用。

处理方式不是无差别使用弱引用，而是明确对象生命周期：页面激活时订阅，停用或释放时取消；使用 `IDisposable`/`IAsyncDisposable` 管理订阅和取消令牌；导航框架应定义缓存策略。

### 3. 导航、对话框和通知的抽象

ViewModel 不应知道具体窗口对象。定义与 UI 无关的抽象，例如 `INavigationService`、`IDialogService`、`INotificationService`，由 WPF 项目提供实现。这样可在单元测试中使用替身，并避免业务模块依赖 `System.Windows`。

抽象必须保持小而有业务意义。不要把 `Window.Show()` 的所有参数原样复制成一个巨大的服务接口；应定义如“编辑设备”“确认高风险操作”“显示非阻塞通知”等用例级 API。

---

## 七、真实产品的推荐架构

### 1. 以组合根为中心，而非以窗口为中心

应用启动时，在唯一的组合根创建依赖注入容器、读取配置、初始化日志、注册全局异常边界并启动主窗口。推荐使用 .NET Generic Host，使桌面端与现代 .NET 服务端采用相同的配置、日志、依赖注入模型。

```csharp
public partial class App : Application
{
    private IHost? host;

    protected override async void OnStartup(StartupEventArgs eventArgs)
    {
        base.OnStartup(eventArgs);

        host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                services.AddSingleton<MainWindow>();
                services.AddSingleton<MainWindowViewModel>();
                services.AddTransient<DeviceListViewModel>();
                services.AddHttpClient<IDeviceApiClient, DeviceApiClient>();
                services.AddSingleton<IDialogService, WpfDialogService>();
                services.AddSingleton<IDeviceQueryService, DeviceQueryService>();
            })
            .Build();

        await host.StartAsync();
        host.Services.GetRequiredService<MainWindow>().Show();
    }

    protected override async void OnExit(ExitEventArgs eventArgs)
    {
        if (host is not null)
        {
            await host.StopAsync();
            host.Dispose();
        }

        base.OnExit(eventArgs);
    }
}
```

仅 `OnStartup`、事件处理器等框架要求的入口可以是 `async void`；业务逻辑应返回 `Task`，使异常、等待、取消和测试都可控。

### 2. 推荐解决方案结构

```text
src/
  Operations.Client.Wpf/             WPF 启动、视图、资源、控件、UI 适配器
  Operations.Application/            用例、命令查询、端口、DTO、校验
  Operations.Domain/                 实体、值对象、领域规则、领域事件
  Operations.Infrastructure/         HTTP、EF Core、文件、设备 SDK、身份实现
  Operations.Contracts/              稳定的 API 契约和共享枚举（按需）
tests/
  Operations.Domain.Tests/
  Operations.Application.Tests/
  Operations.Client.Wpf.Tests/
  Operations.IntegrationTests/
```

依赖方向应始终向内：WPF 依赖 Application，Infrastructure 实现 Application 定义的端口，Domain 不依赖外层项目。不要让 Domain 引用 WPF、EF Core 或 Web API 客户端。

### 3. 模块化而非过度微服务化

大型桌面产品可以按业务能力分模块，例如设备、告警、报表、用户管理。每个模块拥有自己的 View、ViewModel、用例和导航注册，但共享设计系统、基础设施和跨模块契约。

模块边界应按业务能力划分，而不是按“所有 View 在一个项目、所有 ViewModel 在另一个项目”机械划分。后者往往让一次功能修改跨越太多目录，并弱化业务聚合。

### 4. 生命周期设计

| 对象 | 建议生命周期 | 原因 |
| --- | --- | --- |
| HTTP Client | 由 `IHttpClientFactory` 管理 | 连接复用、超时和策略集中配置 |
| 当前会话 | 单例或显式会话上下文 | 保存认证身份、租户和授权结果 |
| 页面 ViewModel | 短生命周期或导航作用域 | 避免跨页面状态污染和订阅泄漏 |
| 缓存 | 有限容量和过期策略 | 防止长时间运行后内存持续增长 |
| 数据库上下文 | 每个用例/工作单元 | 避免跟踪器膨胀与跨线程使用 |

---

## 八、产品案例：设备运维工作台

设想一个部署在工厂内网的设备运维客户端。它需要登录、展示数千台设备、实时告警、设备配置编辑、离线缓存、导出报表，并由多个角色使用。

### 1. 从需求映射到架构

| 需求 | 设计 | 原因 |
| --- | --- | --- |
| 数千设备列表 | 服务端分页、筛选下推、`DataGrid` 行列虚拟化 | 客户端不能一次加载全部明细并渲染 |
| 实时告警 | 后台连接服务接收消息，批量投递 UI | 防止消息风暴阻塞渲染线程 |
| 编辑设备配置 | 编辑副本 + 乐观并发版本号 | 取消编辑不污染原状态，防止覆盖他人修改 |
| 离线可用 | 本地只读缓存、待同步命令队列 | 提升现场网络不稳定时的可用性 |
| 权限控制 | 服务端强制授权，客户端根据能力隐藏/禁用入口 | 客户端体验不等于安全边界 |
| 审计 | 用例层记录操作者、操作、对象、前后摘要和结果 | 满足可追溯要求 |

### 2. 主工作区状态模型

主窗口只负责壳层：导航、标题栏、全局通知和当前工作区。页面不应相互直接查找控件或调用彼此 ViewModel，而通过导航请求、共享的只读会话状态或领域事件协作。

```text
MainWindowViewModel
  CurrentUser
  NavigationItems
  ActiveWorkspace
  NotificationCenter

DeviceListViewModel
  Filter
  PagedDevices
  SelectedDevice
  RefreshCommand
  OpenEditorCommand

DeviceEditorViewModel
  OriginalVersion
  Draft
  ValidationErrors
  SaveCommand
  CancelCommand
```

### 3. 列表加载的正确流程

1. 用户改变筛选条件后，先经过短暂防抖，避免每输入一个字符都发请求。
2. 取消上一次仍在执行的请求，并为当前请求增加版本号。
3. Application 层将筛选条件转换为查询对象，调用 API 获取分页结果和总数。
4. UI 线程一次性替换当前页数据、总数和加载状态；旧版本响应即使稍后返回也不得覆盖新结果。
5. 失败时保留上次成功数据，展示可操作的错误状态和重试入口，而不是清空页面。

### 4. 编辑与并发冲突

页面打开时将 DTO 映射为可编辑 Draft；保存时携带记录版本（如 ETag、RowVersion 或更新时间戳）。服务端发现版本不一致时返回冲突，客户端展示“服务器版本”和“当前草稿”的关键字段差异，允许用户刷新、覆盖或取消。不要悄悄以最后一次保存覆盖先前操作。

### 5. 设备连接与实时数据

将 MQTT、WebSocket、串口或厂商 SDK 封装在 Infrastructure 层。连接服务只负责连接、重连、反序列化和发布领域无关的数据事件；Application 层负责把事件解释为告警或状态变化；ViewModel 仅订阅被投影后的界面数据。

这条边界很重要：如果 ViewModel 直接引用设备 SDK，测试无法模拟断线和异常，换供应商时也会把改动扩散到大量页面。

---

## 九、数据、网络、配置与安全

### 1. 网络访问

- 使用 `IHttpClientFactory` 或集中管理的 HTTP 客户端，不在每个命令中随意 `new HttpClient()`。
- 为连接、读取和整体请求设置符合业务的超时；超时不是“无限等待后的异常”。
- 对幂等读取可采用有上限的退避重试；对创建、扣款、下发设备指令等非幂等操作，必须配合幂等键或明确的状态查询，不能盲目重试。
- 统一映射网络错误、认证失效、业务错误和并发冲突，避免 ViewModel 到处解析状态码。

### 2. 本地数据与缓存

缓存的目标是降低延迟或提供有限离线能力，不是第二个未经治理的数据库。每类数据要定义：所有者、最大容量、过期时间、刷新策略、加密要求、清除条件和冲突策略。

敏感令牌、私钥、密码和医疗/金融敏感数据不应以明文放在 JSON、注册表或日志中。可结合 Windows 凭据管理器、DPAPI、企业密钥管理体系，并按组织合规要求进行保护。

### 3. 配置分层

建议按以下优先级合并配置：内置默认值、环境配置文件、受控部署配置、用户非敏感偏好、命令行参数。将 API 地址、功能开关、日志等级等放入配置；不要把密钥、永久令牌和授权绕过开关放入可编辑的客户端配置文件。

### 4. 客户端安全边界

- 客户端代码、UI 隐藏和禁用按钮都不是授权边界，服务端必须验证身份和权限。
- 使用系统证书链、企业设备管理和可信更新渠道降低被篡改风险。
- 日志中脱敏令牌、密码、个人信息和完整业务载荷。
- 对导出、打印、剪贴板和本地文件使用场景，按数据分级评估泄漏风险。

---

## 十、性能、可访问性与本地化

### 1. 性能优化顺序

先测量，再优化。应先通过 Visual Studio Diagnostic Tools、dotnet-trace、PerfView、Windows Performance Recorder 或产品遥测确认瓶颈属于 CPU、渲染、分配、I/O、锁竞争还是服务端延迟。

常见 WPF 优化点：

- 大数据列表启用 UI 虚拟化和回收虚拟化，避免外层 `ScrollViewer` 破坏它。
- 数据源分页；虚拟化只能减少 UI 元素，不能消除一次性传输和内存占用。
- 避免复杂嵌套模板、频繁触发器和每帧创建画笔、转换器、字符串。
- 对高频事件节流/合并，图表仅保留可视窗口或采样后的数据。
- 使用 `Freezable.Freeze()` 共享不会变化的画笔、几何和图像资源。
- 先验证绑定错误、异常重试循环和日志风暴，它们常被误判为“WPF 慢”。

### 2. 高 DPI 与多显示器

产品应在 100%、125%、150%、200% 缩放和不同 DPI 显示器间拖动窗口时测试。使用逻辑像素、合适的最小尺寸和矢量图标；不要为迁就固定分辨率而截断文本。涉及旧 Win32、WebView 或第三方控件时，要单独验证 DPI 感知策略。

### 3. 可访问性

可访问性不是最后补一个“无障碍模式”。应确保：

- 所有关键操作可通过键盘到达，焦点顺序符合任务流。
- 图标按钮提供可理解名称和自动化属性。
- 错误不仅靠颜色表达，也提供文本、图标或结构化提示。
- 文本与背景具有足够对比度，焦点状态清晰可见。
- 自定义控件提供合理的 AutomationPeer 或等效自动化信息，以支持屏幕阅读器和自动化测试。

### 4. 国际化

将面向用户的字符串、日期格式、数字格式、单位和从右到左布局纳入设计。资源键应表达语义而非英文原文，避免把字符串通过拼接构造句子；不同语言的语序和长度会使拼接失效。动态切换语言时，需要明确哪些页面即时刷新、哪些页面重建。

---

## 十一、测试、诊断与可观测性

### 1. 测试金字塔

| 类型 | 关注点 | 示例 |
| --- | --- | --- |
| Domain 单元测试 | 核心规则、边界条件 | 设备状态转换是否允许 |
| Application 单元测试 | 用例编排、权限、异常映射 | 保存冲突是否返回正确结果 |
| ViewModel 单元测试 | 命令可用性、加载状态、校验、取消 | 筛选改变是否取消旧请求 |
| 集成测试 | HTTP、数据库、设备适配器契约 | API 客户端是否正确序列化 |
| 少量 UI 自动化 | 关键用户旅程 | 登录、编辑保存、导出、升级后启动 |

UI 自动化最脆弱、运行成本最高，不能代替业务层单元测试。它应只覆盖高价值路径，并依赖稳定的 AutomationId 和可预测的测试数据。

### 2. 全局异常处理

至少覆盖三个边界：`Application.DispatcherUnhandledException`、`TaskScheduler.UnobservedTaskException`、`AppDomain.CurrentDomain.UnhandledException`。处理器的职责是记录上下文、尽力通知用户、执行有限清理和决定退出策略；它不能保证应用在所有损坏状态下安全继续运行。

对于可恢复的命令失败，应在命令边界捕获可预期异常，更新对应页面的错误状态并记录结构化日志。不要依赖全局异常处理器处理常规网络错误。

### 3. 可观测性最小集

- 日志：结构化记录操作名、相关 ID、用户角色、耗时、结果和脱敏错误。
- 指标：启动耗时、命令失败率、请求延迟、同步积压、未处理异常次数。
- 追踪：将一次用户操作的客户端请求 ID 传到服务端，串起端到端排障路径。
- 崩溃报告：收集版本、操作系统、异常栈、最近操作摘要和可选 dump，严格控制隐私数据。

发布版本必须可识别。日志应包含产品版本、提交标识或构建号、部署渠道和配置版本，否则现场问题无法与代码和发布物关联。

---

## 十二、部署、升级与运维

### 1. 发布方式选择

| 方式 | 优势 | 注意事项 | 适用情况 |
| --- | --- | --- | --- |
| MSIX | 安装隔离、声明式能力、系统集成较好 | 企业分发、权限和更新策略需提前验证 | 新建企业桌面产品 |
| 企业软件分发 | 可与 Intune、SCCM、组策略等结合 | 依赖企业 IT 流程 | 受管终端、大规模内网 |
| ClickOnce | 上手和自动更新较快 | 定制能力与现代部署诉求需评估 | 内部工具、简单分发 |
| 自定义安装器 | 对驱动、服务、遗留依赖控制强 | 维护和安全责任更高 | 需安装硬件驱动或复杂前置条件 |

部署不只是“打包成 exe”。发布前应验证：签名、证书更新、代理环境、离线安装、回滚、升级中断恢复、旧版本数据迁移、最小权限运行、日志路径和卸载清理。

### 2. 升级策略

- 使用语义化或可追溯的版本号，明确强制升级与可选升级规则。
- 升级包签名并通过受控渠道分发，客户端验证来源和完整性。
- 数据迁移应幂等、可重试，并在失败时保留诊断证据。
- 灰度发布：先在内部和小范围真实环境验证，再逐步扩大；崩溃率和关键功能指标异常时支持暂停发布。
- 明确兼容窗口：客户端版本与服务端 API 版本如何共存，过期客户端如何提示或限制。

### 3. 现场支持能力

对于生产现场软件，建议提供“支持包”功能：在用户授权后导出脱敏日志、版本信息、配置摘要、网络连通性检查结果和可选诊断数据。它能把“现场说打不开”转化为可复现的工程信号。

---

## 十三、常见反模式与改进路径

| 反模式 | 为什么会出问题 | 改进 |
| --- | --- | --- |
| Window 代码隐藏包含所有业务 | 无法测试，窗口成为全局耦合中心 | 将状态和命令移到 ViewModel，用例移到 Application |
| ViewModel 直接访问数据库/硬件 SDK | UI 与基础设施耦合，替换和测试成本极高 | 通过 Application 端口和 Infrastructure 实现隔离 |
| 全局 Service Locator | 依赖隐藏，运行时才发现缺失注册 | 构造函数注入，组合根集中注册 |
| 到处 `Dispatcher.Invoke` | 同步等待导致卡顿和潜在死锁 | 明确后台计算与 UI 提交边界 |
| 列表一次加载全部数据 | 内存、首屏和渲染都不可控 | 服务端分页、筛选、虚拟化、按需加载 |
| `async void` 业务方法 | 异常无法由调用者处理，无法等待与取消 | 除框架事件外返回 `Task` |
| 主题颜色散落在页面 | 视觉不一致，换肤成本指数增长 | 语义资源和集中资源字典 |
| 用客户端隐藏按钮实现权限 | 用户可篡改或绕过，安全错误 | 服务端执行授权，客户端只改善体验 |
| 只在开发机测试 | DPI、代理、权限、证书和旧系统差异被忽略 | 在受控的真实部署矩阵验证 |

---

## 十四、WPF 的未来发展趋势

### 1. 合理预期：稳定演进，而非颠覆式换代

WPF 仍是 .NET 的 Windows 桌面 UI 选项之一，适合维护和扩展已有产品。它的未来更可能表现为：跟随现代 .NET 版本获得运行时、工具链、无障碍、兼容性和局部平台能力改进，而不是在每个视觉或跨平台方向与新框架正面竞争。

对技术负责人而言，这意味着：

- 新项目可继续选择 WPF，但应基于 Windows 专属、高数据密度、企业集成等明确理由。
- 不应把“WPF 不会立刻消失”误读为“无需规划现代化”。依赖升级、安装签名、DPI、无障碍、测试自动化和可观测性仍需持续投入。
- 应避免把业务逻辑锁死在 WPF 层。保持 Application/Domain 独立，可为未来更换 UI 技术保留选择权。

### 2. 值得跟进的方向

| 方向 | 对 WPF 产品的意义 | 建议行动 |
| --- | --- | --- |
| 现代 .NET LTS | 获得受支持运行时、性能、安全和工具链 | 制定年度升级演练与兼容性测试 |
| Windows App SDK/WinUI | 新 Windows 能力和现代设计语言持续集中 | 关注互操作与迁移成本，不盲目重写 |
| 混合 UI | 在局部使用 WebView 或嵌入新 UI 技术 | 只用于独立、边界清晰的页面 |
| AI 辅助工作流 | 本地/云端模型可增强搜索、摘要、告警分析 | 将权限、数据脱敏、审计和失败降级设计在前面 |
| 可访问性法规与企业标准 | 采购、合规和用户体验要求提升 | 将键盘、自动化属性、对比度纳入 Definition of Done |
| 安全供应链 | 客户端更新、依赖和签名成为攻击面 | 引入依赖审计、签名发布和 SBOM 流程 |

### 3. 迁移不是唯一答案

若现有 WPF 产品稳定且其价值主要在业务、设备集成和工作流，全面重写通常风险很高：功能回归、交付停滞、现场兼容性和培训成本都可能超过视觉收益。更现实的策略常是“绞杀式现代化”：先抽离业务层，再替换局部页面或新模块，以共享契约和清晰导航边界控制风险。

---

## 十五、替代方案与选型矩阵

### 1. 主要替代技术

| 技术 | 平台 | 优势 | 主要限制 | 优先考虑的场景 |
| --- | --- | --- | --- | --- |
| WinUI 3 | Windows | 现代 Windows 视觉与 Windows App SDK 对齐 | 生态与迁移路径需逐项验证 | 全新 Windows 11 风格应用、重视新平台能力 |
| Avalonia UI | Windows/macOS/Linux | .NET 跨平台、XAML/MVVM 心智模型接近 WPF | 原生控件和平台特性需评估 | 工业工具、开发者工具、多桌面系统交付 |
| .NET MAUI | Windows/macOS/Android/iOS | 一套 .NET 技术栈覆盖桌面和移动 | 桌面高密度复杂 UI 需做原型验证 | 移动端是核心且桌面功能较轻 |
| Uno Platform | Windows/Web/macOS/Linux/移动端 | 统一跨端 UI 和 Windows 风格 API 经验 | 团队需接受其工具链和平台差异 | 已有 WinUI/UWP 经验、明确多端目标 |
| Web + Electron/Tauri | 多平台 | Web 人才与组件生态丰富，适合快速迭代 | 本地集成、内存、安全和打包策略需治理 | 表单/协作/信息展示为主的跨平台产品 |
| Qt | 多平台 | C++ 原生能力、图形和嵌入式生态成熟 | 与 .NET 技术栈不一致，学习和集成成本高 | 高性能图形、嵌入式、已有 Qt 团队 |

### 2. 决策问题

在框架选型评审中，用可验证问题替代“哪个更现代”：

1. 未来三年必须支持哪些操作系统、CPU 架构和离线环境？
2. 是否需要硬件驱动、串口、COM、Office、Active Directory、智能卡或企业证书集成？
3. 最高数据量、刷新频率、图表复杂度和冷启动目标是多少？
4. 团队现有技能、招聘市场和第三方控件预算如何？
5. 是否能做两个最高风险页面的原型，并在真实硬件、DPI、代理和辅助功能环境验证？
6. 迁移的业务收益是什么，是否大于功能回归和双轨维护成本？

### 3. 简化的选型结论

- **Windows 专属、复杂企业桌面、设备/Win32 集成强**：WPF 仍是稳健选择。
- **Windows 专属、强调最新 Windows 外观与平台能力**：评估 WinUI 3。
- **桌面跨平台是硬需求，且团队以 .NET 为主**：优先验证 Avalonia。
- **移动端和桌面端都重要，但桌面交互不重**：评估 .NET MAUI。
- **以表单、协作、在线交付为中心，团队 Web 能力强**：评估 Web 容器路线。

---

## 十六、落地路线图与面试自检

### 1. 新产品的 90 天落地路线

| 阶段 | 目标 | 关键产出 |
| --- | --- | --- |
| 第 1-2 周 | 验证技术风险 | 启动、登录、主导航、大列表、硬件/网络接入原型 |
| 第 3-4 周 | 建立工程骨架 | 分层项目、DI、日志、配置、异常边界、设计令牌、CI |
| 第 2 月 | 交付第一个业务闭环 | 一条可演示且可测试的查询-编辑-保存-审计流程 |
| 第 3 月 | 产品化加固 | 性能基线、自动化测试、部署、升级、诊断包、试点反馈 |

技术原型必须覆盖最高风险，而不是只做一个漂亮登录页。例如，设备产品应尽早验证驱动安装、断线重连、高 DPI、设备峰值数据量和受限网络；金融产品应尽早验证证书、身份、审计和升级策略。

### 2. 架构自检清单

- [ ] ViewModel 是否能在不创建 `Window` 的情况下被单元测试？
- [ ] 业务规则是否位于 WPF 之外，并有对应的自动化测试？
- [ ] 所有耗时命令是否支持取消、错误呈现和重复触发控制？
- [ ] 大列表是否同时验证了服务端分页与客户端虚拟化？
- [ ] 是否定义了页面离开时的订阅、定时器、任务和缓存释放策略？
- [ ] 是否在不同 DPI、语言、键盘操作和辅助功能工具下验证关键流程？
- [ ] 是否有签名、升级、回滚、脱敏日志和现场诊断方案？
- [ ] 是否可以从日志关联一次用户操作到服务端调用和具体发布版本？

### 3. 面试中的高质量回答框架

回答 WPF 架构题时，按以下顺序组织：

1. **结论**：说明选择的架构或技术，例如“以 MVVM 加分层架构实现，WPF 只保留 UI 适配职责”。
2. **原理**：解释绑定、线程亲和性、命令和依赖方向如何支持该结论。
3. **场景**：给出列表分页、实时告警、离线缓存或并发编辑的真实例子。
4. **边界**：指出 WPF 仅 Windows、绑定运行时错误、UI 自动化脆弱等限制。
5. **验证**：说明测试、性能剖析、部署试点、日志和指标如何证明设计有效。

真正成熟的 WPF 工程能力，不是能写出复杂 XAML，而是让一个长期运行的客户端在网络抖动、数据增长、版本升级、权限变化和现场故障下仍然可维护、可诊断、可演进。