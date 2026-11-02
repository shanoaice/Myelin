# Ranvier 细粒度集成观察

## 状态

- 日期：2026-11-16
- Ranvier 源码：`/home/shanoaice/build/Ranvier`
- Ranvier 版本：tag `v0.2.1`，commit `1fd09a3`
- 工作区状态：干净
- 目标：评估 signal/memo/effect 如何挂载到 Myelin 的 Blazor/RazorConsole 组件，并决定 `MvuComponentBase` 的命名与职责是否应调整

## 结论

Ranvier fine-grained 模式可以直接挂载到 Blazor 组件，不需要伪造一个 `'model` 或 `'msg`。

推荐时序如下：

1. 在 `OnInitialized` 的 renderer context 上构造或取得 graph。
2. 在 `graph.Run` 中调用 `createRoot`，创建 signal、memo、async value 和 effect。
3. effect 在依赖变化后调用组件的 `RequestRender`。
4. 事件处理器运行在 renderer context 时，可以直接写 signal。
5. 后台任务通过 `Graph.Dispatch`、`Graph.DispatchAsync` 或 Ranvier 自带的 async 原语回到 graph。
6. 组件销毁时释放自己的 root owner；如果 graph 由组件独占，再释放 graph。

首批集成可以实现“细粒度状态更新 + 整棵 `Node` 重新计算 + Blazor diff”。
当前 Myelin 基类只持有一个 `currentView: Node`，还没有区域级增量渲染结构。

当前的 `MvuComponentBase<'model, 'msg>` 不适合作为 Ranvier fine-grained 的公共基类。
建议保留 `MvuComponentBase` 作为 MVU 专用类型，并拆出一个通用节点渲染基类。
首选名称是 `NodeComponentBase`。

## Ranvier 的线程与调度契约

### Thread affinity

Ranvier 定义三种线程策略：

- `Guarded`：graph 由构造它的线程独占。离线程 mutation 会抛异常。
- `Unchecked`：不做线程检查，调用方自行保证串行访问。
- `Serialised`：多个线程可以依次进入 graph，但必须拥有构造时捕获的 `SynchronizationContext`，并且同一时刻只能有一个线程在 graph 内。

来源：

- `src/Ranvier/Types.fs:36-60`

`Serialised` 的源码注释明确指出 Blazor Server 场景。
Blazor Server circuit 的 work item 运行在同一 context 中，但可能使用不同线程池线程。

### Context 与默认 dispatcher

`Graph` 在构造时读取：

- `ThreadAffinity`。
- 当前 `SynchronizationContext`。
- `GraphOptions.Dispatcher`，未提供时解析默认 dispatcher。

来源：

- `src/Ranvier/Core.fs:2390-2416`
- `src/Ranvier/Core.fs:2518-2530`

默认 dispatcher 的行为是：

- 当前存在 `SynchronizationContext`：使用 `SynchronizationContextDispatcher`。
- 当前没有 context：使用 `ManualDispatcher`。

来源：

- `src/Ranvier/Platform.fs:549-572`

因此 graph 必须在正确的 renderer context 上构造。
构造之后，后台 settle 才会被默认 dispatcher 投递回该 context。

### Dispatch 与 Pump

`Graph.Dispatch work` 的行为是：

- 已在 graph 线程或 graph 内：inline 执行。
- 在其他线程：写入 graph inbox，然后由 dispatcher 唤醒 drain。
- `ManualDispatcher` 不会自动唤醒，必须由 graph owner 调用 `Graph.Pump`。

来源：

- `src/Ranvier/Core.fs:2914-2958`
- `src/Ranvier/Core.fs:3112-3185`

`Graph.DispatchAsync` 提供等待 work 完成的 Task。
不要在 graph owner 线程同步阻塞这个 Task；当 owner 是唯一 drain 线程时会死锁。

来源：

- `src/Ranvier/Core.fs:2985-2995`

### Graph 激活与 root ownership

`graph.Run body` 激活 graph，执行 body，然后恢复此前的 ambient graph。
`graph.Run` 不负责销毁 graph。

来源：

- `src/Ranvier/Api.fs:911-925`

`createRoot` 创建嵌套 owner，并把 body 中创建的节点挂到该 owner。
释放 owner 会一起释放它的子树。

来源：

- `src/Ranvier/Api.fs:567-572`
- `src/Ranvier/Core.fs:2793-2813`

`graph.Dispose` 会释放 `graph.Root`、关闭 pending dispatch，并故障尚未执行的 `DispatchAsync` Task。

来源：

- `src/Ranvier/Core.fs:2857-2887`

### Effect、memo 与 async value

- `createSignal` 创建可写 source。signal 本身不被 owner 管理。
- `createMemo` 创建纯派生值，归属当前 owner。
- `createEffect` 创建 side effect，第一次运行现在发生，之后在读取的 source 变化时重跑。
- `createEffectOn` 把依赖计算和 action 分开。action 只在计算结果相对上次 action 不等时执行。
- `createAsync` 创建异步派生值，完成结果通过 graph 回到一致性域。

来源：

- `src/Ranvier/Api.fs:268-329`
- `src/Ranvier/Api.fs:383-420`

Ranvier 官方 Blazor Server 指南已经给出完整组件模式：

- 用 `Serialised` 为 circuit 建 graph。
- 在 `graph.Run` 和 `createRoot` 中创建节点。
- 用 effect 请求渲染。
- 组件销毁时释放 root owner。

来源：

- `docs/content/guide/blazor-server.md:10-32`
- `docs/content/guide/blazor-server.md:49-132`

## 挂载到 Blazor/RazorConsole

### 推荐生命周期

| 阶段 | 工作 |
| --- | --- |
| `OnInitialized` | 在 renderer context 上取得或构造 graph；`graph.Run`；`createRoot`；创建 signals、memos、async values 和 render effect；保存 owner 和视图状态 |
| 首次 `OnAfterRenderAsync` | 启动不能早于真实挂载的外部任务或订阅；启用后续 render request |
| 事件回调 | 已在 renderer context 时可以同步写 signal；后台来源改用 `Graph.Dispatch` |
| 后台完成 | 使用 Ranvier async 原语，或 `Graph.Dispatch`/`DispatchAsync` 回到 graph |
| `Dispose` | 释放组件 root owner；组件独占 graph 时再释放 graph |

Ranvier 官方模式在 `OnInitialized` 中直接建立 effect。
effect 首次运行会立即执行，因此 `RequestRender` 可能发生在首次 render 之前。
可以接受这个行为，因为 Blazor 会合并一定的 render request。
如果希望严格避免首次多余请求，可以在 effect 中使用 `firstRenderCompleted` 标志，只让后续变化请求渲染。

### Renderer context 是关键

本仓库的 `TestRenderer` 实测显示：

- `ComponentBase.OnInitialized` 执行时，`SynchronizationContext.Current` 是 `Microsoft.AspNetCore.Components.Rendering.RendererSynchronizationContext`。

这意味着在 `OnInitialized` 中构造 Ranvier graph，可以捕获 renderer context。
默认 dispatcher 随后会把后台 drain 投递回同一个 renderer context。

仍需在真实 `RazorConsole.Core.ConsoleRenderer` 中重复验证。

### 最小 Ranvier 组件草图

下面的草图基于本提案建议的通用 `NodeComponentBase`。
它是设计示意，不是已编译实现。

```fsharp
type RanvierCounterComponent() =
    inherit NodeComponentBase()

    let graph =
        new Graph(
            GraphOptions.Default.WithThreadAffinity Serialised
        )

    let mutable root: Owner option = None
    let mutable firstRenderCompleted = false

    override this.InitializeNode(context) =
        graph.Run(fun () ->
            createRoot(fun owner ->
                let count = createSignal 0

                let view =
                    createMemo(fun _ ->
                        let current = count.Value

                        Node(fun receiver builder sequence ->
                            builder.AddContent(sequence, $"Count: {current}")
                            sequence + 1))

                createEffectOn (fun () -> view.Value) (fun node ->
                    this.SetView(node)

                    if firstRenderCompleted then
                        this.RequestRender())

                root <- Some owner))

    override this.OnAfterRenderAsync(firstRender) =
        if firstRender then
            firstRenderCompleted <- true

        Task.CompletedTask

    interface IDisposable with
        member _.Dispose() =
            root |> Option.iter _.Dispose()
            graph.Dispose()
```

关键点如下：

- effect 第一次运行设置初始 `Node`，所以首次 render 不依赖 effect 触发额外 `StateHasChanged`。
- `createMemo` 在生成 `Node` 时读取 `count.Value`，因此 memo 真正订阅 count；`Node` 的渲染委托只捕获当时的快照。
- 后续 effect 先更新 cached `Node`，再请求 Blazor render。
- `BuildRenderTree` 仍由通用基类执行，不直接读取可能 stale 的 graph 值。
- graph 与 root 都由组件独占时，两者都要释放。
- 如果 graph 是 circuit 级共享服务，组件只释放自己的 root，graph 由服务生命周期释放。

### Event 与后台任务

事件处理器通常已在 renderer context。
下面可以直接执行：

```fsharp
count.Value <- count.Value + 1
```

后台线程不能直接写 signal。
使用：

```fsharp
graph.Dispatch(fun () ->
    count.Value <- count.Value + 1)
```

也可以使用 `Graph.DispatchAsync` 等待 graph 完成 mutation。
但不要从 graph owner 线程阻塞等待它。

### Dispose

组件 root owner 应在 renderer context 上释放。
Ranvier 官方文档明确说 renderer 会在 circuit context 上释放组件。

如果容器或应用服务可能在非 renderer context 上释放共享 graph，应使用：

```fsharp
graph.Dispatch(fun () -> graph.Dispose ())
```

来源：

- `docs/content/guide/blazor-server.md:54-57`
- `docs/content/guide/blazor-server.md:131-132`

组件的 `IDisposable` 不能等待 `Graph.DispatchAsync`。
因此组件级清理应保证在正确 context 上同步执行。
如果宿主的 disposal 线程无法保证，应改用 `IAsyncDisposable`，或由更长生命周期的服务负责 graph 释放。

## 渲染粒度

Ranvier 的细粒度位于状态依赖图：

- signal 只在值变化时唤醒读者。
- memo 只在派生值变化时唤醒读者。
- effect 只在其读取依赖变化时重跑。

Myelin 当前的渲染边界仍是一个 `Node`：

- `MvuComponentBase` 缓存 `currentView: Node option`。
- `BuildRenderTree` 调用当前 `Node`。
- 一次 render 仍会重新遍历整棵 `Node`，再由 Blazor/ConsoleRenderer diff。

来源：

- `src/Myelin.Core/MvuComponentBase.fs:33-34`
- `src/Myelin.Core/MvuComponentBase.fs:123-127`

因此首批 Ranvier 集成可以做到：

- 细粒度决定何时重算视图。
- 避免无关 signal 变化触发 render。
- 每次仍重建完整 `Node`。

它暂时不能做到：

- 只重建一个 region 的 `Node`。
- 多个 Ranvier region 独立提交各自子树。
- 基于 view diff 的局部 render 调度。

这些能力需要新的 Node 分区模型。
不应把它们和 Ranvier 状态细粒度混为一谈。

## Graph.Dispatcher 与 Myelin render dispatcher

两个 dispatcher 服务不同的一致性域：

| 工作 | 首选入口 |
| --- | --- |
| 修改 signal/memo 依赖图 | `Graph.Dispatch` 或 graph 内执行 |
| async settle / stream item | Ranvier async 原语内部调用 graph dispatch |
| 创建或释放组件 root | 在 renderer context 调用 `graph.Run` / `owner.Dispose` |
| 请求 Blazor render | `RequestRender` |
| 在 renderer context 执行非 graph 工作 | `MvuRenderDispatcher`/通用 render dispatcher |
| 自定义 Ranvier graph drain | 实现 `IGraphDispatcher` 适配到 render dispatcher |

不要形成重复队列：

```text
background -> render dispatcher -> Graph.Dispatch -> graph dispatcher -> renderer
```

状态工作应直接进入 `Graph.Dispatch`。
render dispatcher 只处理组件或 renderer 范围的工作。

`MvuRenderDispatcher` 仍有价值：

- 它给其他集成适配器提供通用的 renderer 调度能力。
- 它能在 graph 尚未建立或已经释放时管理组件生命周期工作。
- 它可以把 Ranvier 的 `IGraphDispatcher` 绑定到组件生命周期。

但 Ranvier 的主要状态调度不应依赖 Myelin 的 dispatcher。
Ranvier 自己已经实现 graph inbox、thread affinity、async settle 和 Pump 契约。

## 基类职责与命名

### 当前 `MvuComponentBase` 的职责

当前类型同时包含两类职责。

MVU 专用职责：

- `'model` 与 `'msg` 泛型。
- `currentModel`。
- `dispatchFunc` 和 `Dispatch`。
- `ShouldRender(oldModel, newModel)`。
- `SetInitialState(model, view, dispatch)`。
- `UpdateState(newModel, renderView)`。
- `InitializeMvu` 和 `MvuInitContext`。

通用渲染职责：

- `currentView: Node`。
- `RequestRender`。
- render dispatcher 访问。
- `BuildRenderTree`。
- Blazor 生命周期桥接。

来源：

- `src/Myelin.Core/MvuComponentBase.fs:29-127`

Ranvier fine-grained 只需要通用渲染职责。
它没有单一 `'model` 和 `'msg`，因此不应被迫实现 `ShouldRender` 或 `UpdateState`。

### 方案比较

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| A：保持一个基类，让 Ranvier 也继承 | 类型少 | 泛型、model 和 dispatch 语义错误；Ranvier 需要伪造 MVU 参数 |
| B：拆出通用基类，`MvuComponentBase` 保留为子类 | 职责清楚；两种范式共享 render 能力；迁移较小 | 增加一个公共类型和初始化上下文 |

推荐方案 B。

### 推荐 API 形状

```fsharp
[<AbstractClass>]
type NodeComponentBase() =
    inherit ComponentBase()

    member CurrentView: Node option
    member SetView(view: Node) : unit
    member RequestRender() : unit

    abstract InitializeNode:
        context: NodeInitContext -> unit

    override BuildRenderTree(builder: RenderTreeBuilder) : unit


[<AbstractClass>]
type MvuComponentBase<'model, 'msg>() =
    inherit NodeComponentBase()

    member CurrentModel: 'model option
    member Dispatch: 'msg -> unit
    member SetInitialState(model, view, ?dispatch) : unit
    member UpdateState(model, renderView) : unit

    abstract InitializeMvu:
        context: MvuInitContext<'msg> -> unit
```

`NodeInitContext` 提供：

- `RenderDispatcher`。
- 只与组件通用 render 生命周期有关的能力。

`MvuInitContext<'msg>` 提供：

- `RenderDispatcher`。
- one-shot `SetDispatch`。

`MvuComponentBase` 覆盖通用初始化入口，并把它转给 `InitializeMvu`。

### 命名候选

| 名称 | 评价 |
| --- | --- |
| `NodeComponentBase` | 推荐。直接表达核心输出类型 `Node` |
| `MyelinComponentBase` | 可作为品牌型名称，但职责不够具体 |
| `RenderComponentBase` | 过于宽泛，容易和 renderer/view 短语混淆 |
| `ReactiveComponentBase` | 能覆盖 Elmish 和 Ranvier，但“reactive”含义过宽 |
| `ComponentHostBase` | 太泛，无法体现 `Node` 渲染 |

相应命名建议：

- `MvuRenderDispatcher` 在通用层改名为 `RenderDispatcher` 或 `NodeRenderDispatcher`。
- `MvuInitContext` 保留给 MVU。
- 通用上下文使用 `NodeInitContext`。
- `MvuComponentBase` 保留，明确它只服务 MVU。

### 迁移影响

- 现有 `MvuComponentBase` 的派生代码可以基本保持不变。
- 通用成员从子类移到父类后，仍可通过继承访问。
- `InitializeMvu` 和 `MvuInitContext<'msg>` 保持 MVU 名称。
- 新增 `NodeComponentBase` 后，Ranvier 和未来其他 reactive 适配器不必写假的 model/msg。
- 如果在项目稳定前调整，迁移成本最低。

## 待验证项

1. 在真实 `RazorConsole.Core.ConsoleRenderer` 中确认 `OnInitialized` 的 `SynchronizationContext` 与 `TestRenderer` 一致。
2. 编译并运行最小 `RanvierComponent`，验证 `createEffectOn` 首次 action 与首次 render 的时序。
3. 验证组件 `Dispose` 在 RazorConsole 中始终位于 renderer context。
4. 验证 circuit 级共享 graph 与组件级 root 的引用和释放关系。
5. 评估首批集成是否需要 Node region，还是整树 diff 已满足 RazorConsole 性能目标。
6. 决定通用初始化 hook 的名称：`InitializeNode`、`InitializeMyelin` 或其它候选。

## 源码索引

### Ranvier

- `src/Ranvier/Types.fs:36-60`：`ThreadAffinity`。
- `src/Ranvier/Types.fs:188-195`：`IGraphDispatcher`。
- `src/Ranvier/Types.fs:505-562`：`GraphOptions` 和 builder 方法。
- `src/Ranvier/Platform.fs:549-572`：默认 dispatcher。
- `src/Ranvier/Core.fs:2390-2416`：owner thread、context 和 affinity。
- `src/Ranvier/Core.fs:2518-2530`：graph dispatcher。
- `src/Ranvier/Core.fs:2793-2813`：`createRoot`。
- `src/Ranvier/Core.fs:2857-2887`：`Graph.Dispose`。
- `src/Ranvier/Core.fs:2914-2958`：`Graph.Dispatch`。
- `src/Ranvier/Core.fs:2985-2995`：`Graph.DispatchAsync` 阻塞约束。
- `src/Ranvier/Core.fs:3112-3185`：`Graph.Pump`。
- `src/Ranvier/Api.fs:268-329`：memo 与 effect。
- `src/Ranvier/Api.fs:383-420`：async value。
- `src/Ranvier/Api.fs:550-572`：`runWithOwner` 与 `createRoot`。
- `src/Ranvier/Api.fs:911-925`：`graph.Run`。
- `docs/content/guide/blazor-server.md:10-132`：官方 Blazor Server 挂载模式。
- `docs/content/concepts/contracts.md:120-136`：`Serialised` 契约。

### Myelin

- `src/Myelin.Core/MvuComponentBase.fs:29-127`：当前基类职责。
- `test/TestRenderer.fs:1-59`：内存 renderer 与 dispatcher。
