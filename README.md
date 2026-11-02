# Myelin

Lightweight, high-performance F# reactive UI paradigms and view engines for Microsoft Blazor and [RazorConsole](https://github.com/shanoaice/RazorConsole).

This repository provides sequence-threading view delegates and state-management adapters for terminal applications.
It renders terminal user interfaces using Microsoft Blazor and Spectre.Console.

## Packages

This project decouples view rendering from state management:

| Package | Role | Description |
| :--- | :--- | :--- |
| `Myelin.Core` | Core View Engine | Runtime-agnostic view delegates (`Node`, `Attr`), `MvuComponentBase`, and dispatcher synchronization. |
| `Myelin.Elmish` | State Adapter | Connects upstream `Elmish.Program` to Blazor lifecycles via `ProgramComponent<'model, 'msg>`. |
| `Myelin.Ranvier` *(Planned)* | State Adapter | Planned adapter connecting Ranvier reactive loops, signals, and memos to Blazor lifecycles. |
| `Myelin.DSL.RazorConsole` | Terminal View DSL | Declarative terminal view combinators (`View.panel`, `View.rows`, etc.) targeting RazorConsole and Spectre.Console. |

---

## Quick Start (Elmish)

Create an application component inheriting `ProgramComponent`:

```fsharp
namespace SampleApp

open Elmish
open Myelin
open Myelin.Elmish
open Myelin.DSL.RazorConsole

type Model = { Count: int }

type Msg =
    | Increment
    | Decrement

type CounterComponent() =
    inherit ProgramComponent<Model, Msg>()

    override _.Program =
        let init () = { Count = 0 }, Cmd.none

        let update msg model =
            match msg with
            | Increment -> { model with Count = model.Count + 1 }, Cmd.none
            | Decrement -> { model with Count = model.Count - 1 }, Cmd.none

        let view model dispatch =
            View.panel "Counter" "rounded" (
                View.rows [
                    View.text $"Count: {model.Count}"
                    View.button "Increment" Increment dispatch
                    View.button "Decrement" Decrement dispatch
                ]
            )

        Program.mkSimple init update view
```

Mount the component in `Program.fs` using `RazorConsole.Core`:

```fsharp
open Microsoft.Extensions.Hosting
open RazorConsole.Core

[<EntryPoint>]
let main args =
    Host.CreateDefaultBuilder(args)
        .UseRazorConsole<CounterComponent>()
        .Build()
        .Run()
    0
```
