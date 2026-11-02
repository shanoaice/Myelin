# MvuComponentBase 渲染调度能力提案

## 状态

- 状态：Proposed
- 日期：2026-11-16
- 范围：`Myelin.Core`、`Myelin.Elmish`、未来的 `Myelin.Ranvier`
- 目标版本：首个公开 API 稳定版本之前

## 结论

从 `MvuComponentBase` 移除公开的 `InvokeOnDispatcher` 方法。

新增一个不透明、组件级、长生命周期的 `MvuRenderDispatcher` 能力对象。
派生组件在 `InitializeMvu` 中从 `MvuInitContext` 取得该对象，并保存到实例字段。
派生组件在整个组件存续期内使用它调度需要渲染线程的同步或异步工作。

`MvuComponentBase` 继续通过受保护的 `ComponentBase.InvokeAsync` 调用渲染器。
这个 protected 调用只出现在基类的一个 private 成员实现中。
派生组件不再直接调用 protected `InvokeAsync`，也不依赖公开的组件方法。

该设计满足以下约束：

- 应用代码不能通过组件实例取得渲染调度能力。
- Elmish、Ranvier 和其他集成程序集可以在闭包中调度工作。
- 调度能力可跨多次渲染、异步命令、定时器和 effects 使用。
- F# 的 protected 访问限制只由基类处理一次。
- 不需要 `InternalsVisibleTo`。
- 不需要降低 `ComponentBase.InvokeAsync` 的原始访问级别。

## 背景与问题

`Microsoft.AspNetCore.Components.ComponentBase.InvokeAsync` 有两个 protected 重载：

```csharp
protected Task InvokeAsync(Action workItem);
protected Task InvokeAsync(Func<Task> workItem);
```

F# 可以访问继承链上的 protected 成员，但只能在实际派生类型成员的直接实现中访问。
F# 禁止从内部 lambda 或逃逸闭包访问 protected 成员。

下面的调用是合法的：

```fsharp
member private this._InvokeOnDispatcher(action: unit -> Task) =
    this.InvokeAsync(action)
```

下面的调用不合法：

```fsharp
let schedule action =
    fun () -> this.InvokeAsync(action)
```

Elmish 的 `syncDispatch`、Ranvier 的 effects 和异步命令完成回调都需要在闭包中调度工作。
因此，这些集成组件不能直接使用 protected `InvokeAsync`。

F# 目前也不能声明新的 protected 成员。`protected` 是保留标识符。
如果把包装方法改为 `internal`，所有集成程序集都必须加入 friend assembly。
这种授权范围过宽，也把 `Myelin.Core` 和具体集成程序集耦合起来。

公开 `InvokeOnDispatcher` 可以解决编译问题，但会把渲染调度能力暴露给所有持有组件引用的代码。
能力对象把授权范围收窄到组件初始化上下文，同时保留跨程序集的扩展能力。

## 术语与生命周期

### 消息 dispatch

`Dispatch<'msg>` 或 `MvuComponentBase.Dispatch` 负责把应用消息送入状态管理循环。
它不是本文讨论的渲染调度能力。

### 渲染 dispatch

`MvuRenderDispatcher` 负责把工作项放入当前组件的 Blazor renderer synchronization context。
它等价于受控地使用 `ComponentBase.InvokeAsync`。

### 组件附加生命周期

本文使用“组件附加生命周期”表示以下区间：

1. renderer 已为组件分配 `RenderHandle`。
2. `MvuComponentBase.OnInitialized` 已执行 `InitializeMvu`。
3. 组件尚未销毁。

调度能力不需要等到首次渲染完成才能创建。
集成循环仍应在 `OnAfterRenderAsync(firstRender = true)` 后启动，避免在首次视图生成前执行应用副作用。

## API 形状

### `MvuRenderDispatcher`

`MvuRenderDispatcher` 是公开类型，构造函数保持 internal。
调用方只能从 `MvuInitContext` 获得实例。

```fsharp
namespace Myelin

open System
open System.Threading
open System.Threading.Tasks

[<Sealed>]
type MvuRenderDispatcher internal
    (
        invokeAsync: (unit -> Task) -> Task,
        lifetime: CancellationToken
    ) =

    /// 当前组件附加生命周期的取消令牌。
    member _.Lifetime: CancellationToken =
        lifetime

    /// 将异步工作调度到当前组件的 renderer synchronization context。
    member _.InvokeAsync(workItem: unit -> Task) : Task =
        invokeAsync workItem

    /// 将同步工作调度到当前组件的 renderer synchronization context。
    member _.InvokeAsync(workItem: unit -> unit) : Task =
        invokeAsync(fun () ->
            workItem ()
            Task.CompletedTask)
```

API 契约如下：

- 同一个实例只能调度到创建它的组件 renderer。
- 调用可以从任意线程发起。
- 工作项在组件 renderer synchronization context 中执行。
- `Lifetime` 在组件销毁时取消。
- `Lifetime` 取消后，新的调度请求返回已取消的 `Task`。
- 已经排队的 work item 在执行前也要检查取消状态。
- work item 抛出异常时，异常写入返回的 `Task`。
- 调用方不得把该对象保存到跨越组件实例生命周期的静态字段或单例服务中。
- 调用方不得自行创建 `MvuRenderDispatcher`。

### `MvuInitContext<'msg>`

`MvuInitContext<'msg>` 是初始化和能力授予边界。
它继续负责一次性 dispatch 配置，同时暴露长期存活的渲染调度能力。

```fsharp
type MvuInitContext<'msg> internal
    (
        setDispatch: ('msg -> unit) -> unit,
        renderDispatcher: MvuRenderDispatcher
    ) =

    let mutable consumed = 0

    /// 当前组件绑定的渲染调度能力。
    member _.RenderDispatcher: MvuRenderDispatcher =
        renderDispatcher

    /// 配置消息 dispatch。只能调用一次。
    member _.SetDispatch(dispatch: 'msg -> unit) =
        ArgumentNullException.ThrowIfNull(dispatch, nameof dispatch)

        if Interlocked.CompareExchange(&consumed, 1, 0) <> 0 then
            invalidOp "Dispatch delegate has already been configured on this component."

        setDispatch dispatch
```

调用方应保存 `context.RenderDispatcher`，不要为了运行时调度而长期保存整个初始化上下文。

### `MvuComponentBase<'model, 'msg>`

`MvuComponentBase` 不再公开 `InvokeOnDispatcher(unit -> Task)` 和 `InvokeOnDispatcher(unit -> unit)`。
它只保留一个 private 成员，用于完成 protected 访问。

示意实现如下：

```fsharp
[<AbstractClass>]
type MvuComponentBase<'model, 'msg>() =
    inherit ComponentBase()

    let lifetimeCts = new CancellationTokenSource()

    /// F# 允许在这个直接成员实现中访问 protected InvokeAsync。
    member private this._InvokeOnDispatcher(action: unit -> Task) : Task =
        this.InvokeAsync(action)

    override this.OnInitialized() =
        base.OnInitialized()

        let renderDispatcher =
            MvuRenderDispatcher(
                (fun action -> this._InvokeOnDispatcher(action)),
                lifetimeCts.Token
            )

        this.InitializeMvu(
            MvuInitContext(configureDispatch, renderDispatcher)
        )

    interface IDisposable with
        member _.Dispose() =
            lifetimeCts.Cancel()
            lifetimeCts.Dispose()
```

实际实现还需要在 `MvuRenderDispatcher.InvokeAsync` 内完成以下检查：

1. `Lifetime` 已取消时，不调用 protected `InvokeAsync`。
2. 返回 `Task.FromCanceled(lifetime)`。
3. work item 开始执行前再次检查 `Lifetime`。
4. 调度过程中出现的同步异常转换为 faulted `Task`。

`RequestRender`、`UpdateState`、`ShouldRender`、`SetInitialState` 和消息 `Dispatch` 的职责不变。
本提案只替换渲染调度能力的暴露方式。

## 可见性与授权边界

| API | 可见性 | 生命周期 | 用途 |
| --- | --- | --- | --- |
| `MvuRenderDispatcher` 类型 | public | 实例与组件同生命周期 | 描述调度能力 |
| `MvuRenderDispatcher` 构造函数 | internal | 不适用 | 只允许 `Myelin.Core` 创建能力 |
| `MvuInitContext.RenderDispatcher` | public | 与组件实例相同 | 向派生组件授予能力 |
| `MvuInitContext.SetDispatch` | public | one-shot | 配置消息 dispatch |
| `MvuComponentBase.InvokeOnDispatcher` | 移除 | 不适用 | 禁止应用代码直接访问 |
| `ComponentBase.InvokeAsync` | protected | 组件附加生命周期 | 基类内部的原始能力 |

应用代码无法从 `MvuComponentBase` 或 `MvuInitContext` 的公开构造入口取得调度能力。
派生组件可以决定是否再次暴露该能力，这属于应用自己的 API 决策。

## 消费方获取与保留

### 获取

消费方在 `InitializeMvu` 中取得能力：

```fsharp
type IntegrationComponent() =
    inherit MvuComponentBase<Model, Msg>()

    let mutable renderDispatcher: MvuRenderDispatcher option = None

    override this.InitializeMvu(context) =
        ArgumentNullException.ThrowIfNull(context, nameof context)

        renderDispatcher <- Some context.RenderDispatcher

        // 初始化模型、视图和消息 dispatch。
        ()
```

### 保留

消费方把 `MvuRenderDispatcher` 保存在组件实例字段中。
这个字段可以跨多次渲染、命令完成、订阅回调和 effects 使用。

消费方不应执行以下操作：

- 保存到静态字段。
- 注册为进程级单例。
- 传给未来可能比组件存活更久的服务。
- 在组件销毁后继续调用。
- 保存 `MvuInitContext` 来代替调度能力。

如果业务工作必须比组件实例存活更久，应把状态和任务所有权移到应用服务或外部 store。
新组件挂载时再绑定新的 render dispatcher。

## 使用方式

### 通用调用

异步工作直接返回 `Task`：

```fsharp
match renderDispatcher with
| Some dispatcher ->
    dispatcher.InvokeAsync(fun () ->
        updateState ()
        Task.CompletedTask)
    |> ignore
| None ->
    invalidOp "The MVU component is not initialized."
```

同步工作使用同步重载：

```fsharp
dispatcher.InvokeAsync(fun () ->
    updateState ())
|> ignore
```

纯计算和 I/O 不需要占用渲染线程。
只有读取或修改 UI 相关状态、调用 `StateHasChanged`、更新 reactive graph 中与视图绑定部分的工作需要通过 scheduler。

### Elmish

`ProgramComponent` 在 `InitializeMvu` 中保存 render dispatcher。
首次渲染完成后，`syncDispatch` 使用该能力序列化所有消息。

```fsharp
let mutable renderDispatcher: MvuRenderDispatcher option = None
let mutable mappedProgram = None

override this.InitializeMvu(context) =
    ArgumentNullException.ThrowIfNull(context, nameof context)

    renderDispatcher <- Some context.RenderDispatcher

    // 原有 init、view、SetInitialState 和 Program.map 逻辑。
    ()

override this.OnAfterRenderAsync(firstRender) =
    if firstRender then
        let dispatcher =
            renderDispatcher
            |> Option.defaultWith (fun () ->
                invalidOp "ProgramComponent has not been initialized.")

        let syncDispatch (dispatch: Dispatch<'msg>) =
            fun msg ->
                dispatcher.InvokeAsync(fun () ->
                    dispatch msg
                    Task.CompletedTask)
                |> ignore

        mappedProgram
        |> Option.iter (fun program ->
            Program.runWithDispatch syncDispatch this program)

        Task.CompletedTask
    else
        Task.CompletedTask
```

该调度能力必须覆盖整个 Elmish 循环生命周期，不只覆盖挂载握手。
`Cmd.OfTask`、定时器、订阅和初始化命令产生的后台消息都通过同一个能力回到渲染线程。

生产实现还应观察 `InvokeAsync` 返回的 faulted `Task`，并把错误交给 Elmish `program.onError`。
不能把异常完全静默丢弃。

### Ranvier

`RanvierComponent` 使用同一个获取方式。
effects、异步 memo 完成回调和视图刷新都通过保存的 `MvuRenderDispatcher` 调度。

```fsharp
override this.InitializeMvu(context) =
    renderDispatcher <- Some context.RenderDispatcher

    context.RenderDispatcher.Lifetime.Register(fun () ->
        owner |> Option.iter _.Dispose())

    ()
```

effect 回调不能在闭包内直接调用 `this.InvokeAsync`。
应使用保存的 render dispatcher。

如果 `Ranvier` 的 `ThreadAffinity.Serialised` 依赖当前的 `SynchronizationContext`，创建 affinity root 或捕获上下文的代码必须在调度后的 render-thread work item 中执行。

## 生命周期规则

| 阶段 | render dispatcher 状态 | 要求 |
| --- | --- | --- |
| `InitializeMvu` 之前 | 不可用 | 不得启动依赖 UI 线程的循环 |
| `InitializeMvu` 内 | 可获取 | 保存能力，只执行必要初始化 |
| 首次渲染完成后 | 可用 | Elmish 和 Ranvier 可以启动长期循环 |
| 后续多次渲染 | 可用 | 持续使用同一个组件级能力 |
| 组件销毁中 | `Lifetime` 取消 | 停止订阅、effects 和后台任务 |
| 组件销毁后 | 不再有效 | 不得重新使用，也不得跨实例传递 |
| 组件重新挂载 | 新实例、新能力 | 重新获取并重新绑定 |

组件销毁后，普通 dispatch 消息也不应再驱动旧组件。
Elmish subscriptions 和 Ranvier effects 必须在销毁时终止。
仅依赖 `InvokeAsync` 返回已取消任务不足以停止后台生产者。

## 跨组件实例的场景

以下需求不属于组件级 `MvuRenderDispatcher` 的范围：

- 消息在组件销毁后到达，但业务状态仍需更新。
- 根组件在保持 renderer 的情况下重新挂载。
- 多个组件需要调度到同一个 renderer。
- Blazor prerender 切换到新的交互实例。

这些场景需要 renderer 级 scheduler 或应用级状态服务。
组件级能力只连接“当前组件实例”和“其 renderer”。

建议后续单独设计 renderer 级调度抽象。
不要让组件级 `MvuRenderDispatcher` 承担跨实例生命周期。

## API 兼容与迁移

建议按以下顺序迁移：

1. 在 `Myelin.Core` 增加 `MvuRenderDispatcher`。
2. 扩展 `MvuInitContext`，增加 `RenderDispatcher`。
3. 在 `MvuComponentBase` 增加 private protected-access 适配器、生命周期令牌和 `IDisposable`。
4. 迁移 `Myelin.Elmish.ProgramComponent`，移除对公开 `InvokeOnDispatcher` 的依赖。
5. 从 `MvuComponentBase` 删除两个公开 `InvokeOnDispatcher`。
6. 更新 Ranvier 可行性文档中的直接 `this.InvokeAsync(Action(...))` 示例。
7. 在 Ranvier 集成中使用相同能力。

如果已有外部包依赖公开 `InvokeOnDispatcher`，可以先保留一个版本的 obsolete 成员。
该成员应转发到 `MvuRenderDispatcher`，并在下一主版本删除。
项目尚未发布稳定 API 时，直接删除更清晰。

## 测试与验收

### API 表面

- `MvuComponentBase` 不包含公开的 `InvokeOnDispatcher` 方法。
- `MvuRenderDispatcher` 没有公开构造函数。
- 普通应用代码不能从组件实例取得 render dispatcher。

### 能力获取

- 派生组件可以在 `InitializeMvu` 中取得并保存 `RenderDispatcher`。
- 第二个程序集中的派生组件可以完成同样操作，不需要 friend assembly。
- `SetDispatch` 仍然只能调用一次。

### 线程调度

- 后台线程调用 `RenderDispatcher.InvokeAsync` 后，work item 在 renderer synchronization context 中执行。
- 多个并发调用保持串行。
- 同步和异步重载都可用。
- work item 异常反映在返回的 `Task` 上。

### 生命周期

- 调度能力可跨首次渲染和后续多次渲染使用。
- 组件销毁时 `Lifetime` 取消。
- 销毁后的新调用返回已取消 `Task`。
- 已排队但尚未执行的 work item 在销毁后不执行。
- 两个组件实例的调度能力相互隔离。
- 组件重新挂载后取得新的调度能力。

### Elmish

- `Cmd.OfTask` 从后台线程发消息后，界面正确更新。
- 定时器和订阅产生的消息通过 render dispatcher 串行处理。
- 首次渲染前不启动长期循环。
- 组件销毁后订阅停止。
- faulted scheduler task 进入 `program.onError`。

### Ranvier

- effect 从非渲染线程触发时，通过 render dispatcher 更新 reactive graph。
- `ThreadAffinity.Serialised` 在正确上下文中创建。
- 组件销毁时 owner 和 effects 被释放。

## 未纳入本提案

- renderer 级跨组件 scheduler。
- 应用级状态 store 的 API。
- `Dispatch<'msg>` 在组件销毁后的统一策略。
- 自动终止任意 Elmish Program 的通用机制。
- 跨 prerender 和交互实例的任务迁移。

这些能力会改变所有权边界，应与组件级调度能力分开设计。
