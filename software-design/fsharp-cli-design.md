---
description: Opinionated style guide for designing F# CLI applications using Argu, with patterns for argument parsing, command routing, result handling, and composition in the main function.
tags: [fsharp, cli, style-guide]
---

# F# CLI Design Style Guide

Date: 2026-05-29

This guide covers how to structure a CLI host project in F# using [Argu](https://fsprojects.github.io/Argu/) for argument parsing. It assumes the CLI is a host in a clean/ports-and-adapters architecture (see `clean-architecture-style-guide.md`). The CLI project (`Shipping.Cli`) wires everything together; it contains no business logic.

---

## Core Principle

The composition root is where production implementations are assembled and wired together. In a CLI this happens at startup, called from `main` — it could be inline in `main` for small tools or extracted into a dedicated `CompositionRoot` module for larger ones. Either way, the wiring is in one place.

Everything else — parsing, mapping, routing, result handling — is a pure transformation in a pipeline:

```
parse args -> map to domain command -> route to use case -> handle result -> exit code
```

Argu types are an infrastructure detail. They never cross into the domain. The `Mapper` module is the seam between Argu and the domain.

---

## Module Organization

Within `Shipping.Cli`, organize files in this order (F# files are order-dependent):

```
Shipping.Cli/
  Args.fs                -- Argu discriminated union(s) and parser setup
  Mapper.fs              -- Translates Argu parse results to domain Command types
  Router.fs              -- Matches domain Commands and calls use cases
  CompositionRoot.fs     -- (optional) Wires adapters into use case functions
  Program.fs             -- Entry point; parses args, calls composition root, handles result
```

Each file has a single responsibility. Nothing leaks upward: `Args` knows nothing of the domain; `Mapper` knows both; `Router` knows the domain and use cases; `CompositionRoot` (or `main` directly) knows the adapters; `Program` coordinates the pipeline.

---

## Argument Parsing with Argu

Define the CLI spec as a discriminated union. Each case maps to a subcommand or flag. Use Argu attributes to control help text and validation.

```fsharp
// Args.fs
module Shipping.Cli.Args

open Argu

[<CliPrefix(CliPrefix.DoubleDash)>]
type RegisterArgs =
    | [<Mandatory; AltCommandLine("-i")>] Id of shipment_id: string
    | [<Mandatory; AltCommandLine("-o")>] Origin of origin: string
    | [<Mandatory; AltCommandLine("-d")>] Destination of destination: string
    interface IArgParserTemplate with
        member this.Usage =
            match this with
            | Id _ -> "Unique shipment identifier."
            | Origin _ -> "Origin location code."
            | Destination _ -> "Destination location code."

[<CliPrefix(CliPrefix.DoubleDash)>]
type CancelArgs =
    | [<Mandatory; AltCommandLine("-i")>] Id of shipment_id: string
    | [<AltCommandLine("-r")>] Reason of reason: string
    interface IArgParserTemplate with
        member this.Usage =
            match this with
            | Id _ -> "Unique shipment identifier."
            | Reason _ -> "Optional cancellation reason."

type CliArguments =
    | [<CliPrefix(CliPrefix.None)>] Register of ParseResults<RegisterArgs>
    | [<CliPrefix(CliPrefix.None)>] Cancel of ParseResults<CancelArgs>
    interface IArgParserTemplate with
        member this.Usage =
            match this with
            | Register _ -> "Register a new shipment."
            | Cancel _ -> "Cancel an existing shipment."
```

Parse in `Program.fs` using `ArgumentParser.Create`. Do not parse inside individual modules.

---

## Mapper Module

The `Mapper` module translates Argu `ParseResults` into domain `Command` types. This is the only module that references both Argu types and domain types. Keep this translation explicit and total — every Argu case has a corresponding domain case.

Domain commands live in the domain project. They carry only validated data; no Argu types leak through.

```fsharp
// In Shipping (domain project)
module Shipping.Commands

type RegisterShipmentCommand = {
    ShipmentId: string
    Origin: string
    Destination: string
}

type CancelShipmentCommand = {
    ShipmentId: string
    Reason: string option
}

type Command =
    | RegisterShipment of RegisterShipmentCommand
    | CancelShipment of CancelShipmentCommand
```

```fsharp
// Mapper.fs
module Shipping.Cli.Mapper

open Argu
open Shipping.Cli.Args
open Shipping.Commands

let toCommand (results: ParseResults<CliArguments>) : Command =
    match results.GetSubCommand() with
    | Register registerArgs ->
        RegisterShipment {
            ShipmentId = registerArgs.GetResult RegisterArgs.Id
            Origin = registerArgs.GetResult RegisterArgs.Origin
            Destination = registerArgs.GetResult RegisterArgs.Destination
        }
    | Cancel cancelArgs ->
        CancelShipment {
            ShipmentId = cancelArgs.GetResult CancelArgs.Id
            Reason = cancelArgs.TryGetResult CancelArgs.Reason
        }
```

The mapper is pure. It does not call use cases, perform IO, or produce side effects. If argument validation beyond what Argu provides is needed, return `Result<Command, string>` here and handle the error at the boundary.

---

## Command Routing

The `Router` module matches on `Command` and dispatches to use cases. Use cases receive their dependencies as function parameters.

```fsharp
// Router.fs
module Shipping.Cli.Router

open Shipping.Commands

let handle
    (registerShipment: RegisterShipmentCommand -> Result<unit, string>)
    (cancelShipment: CancelShipmentCommand -> Result<unit, string>)
    (command: Command)
    : Result<unit, string> =
    match command with
    | RegisterShipment cmd -> registerShipment cmd
    | CancelShipment cmd -> cancelShipment cmd
```

The router knows the domain and the use case function signatures. It does not know which adapter implements them — that is wired in `main`. No business logic lives here.

---

## Use Cases

Use cases live in the domain project and follow the pattern from the clean architecture guide: a module with an `execute` function that takes its dependencies as function parameters.

```fsharp
// In Shipping (domain project)
module Shipping.UseCases.RegisterShipment

open Shipping.Commands
open Shipping.Ports

let execute
    (saveShipment: Shipment -> Result<unit, string>)
    (command: RegisterShipmentCommand)
    : Result<unit, string> =
    // domain logic here
    let shipment = Shipment.create command.ShipmentId command.Origin command.Destination
    saveShipment shipment
```

The CLI passes adapters into these functions. The use case never references the CLI or any adapter directly.

---

## Composition Root

The composition root assembles adapters and partially applies them into use case functions. For small CLIs this lives inline in `main`. For larger CLIs, extract it into a `CompositionRoot` module and call it from `main`.

`Program.fs` parses arguments, invokes the composition root to get wired use cases, routes the command, and translates the result to an exit code.

```fsharp
// Program.fs
module Shipping.Cli.Program

open Argu
open Shipping.Cli.Args
open Shipping.Cli.Mapper
open Shipping.Cli.Router
open Shipping.UseCases
open Shipping.Adapter.Postgres

[<EntryPoint>]
let main argv =
    let parser = ArgumentParser.Create<CliArguments>(programName = "shipping")

    let result =
        try
            let parseResults = parser.ParseCommandLine(inputs = argv, raiseOnUsage = true)
            let command = toCommand parseResults

            // Wire adapters
            let connectionString = System.Environment.GetEnvironmentVariable("DB_CONNECTION")
            let saveShipment = ShipmentRepository.save connectionString
            let deleteShipment = ShipmentRepository.delete connectionString

            // Partially apply dependencies into use cases
            let registerShipment = RegisterShipment.execute saveShipment
            let cancelShipment = CancelShipment.execute deleteShipment

            // Route and execute
            handle registerShipment cancelShipment command
            |> Result.mapError (fun err -> err)

        with
        | :? ArguParseException as ex ->
            eprintfn "%s" ex.Message
            Error "Argument parsing failed."

    match result with
    | Ok () ->
        0
    | Error msg ->
        eprintfn "Error: %s" msg
        1
```

The composition root:
- Is the only place where adapter implementations are instantiated
- Produces fully-wired use case functions ready to be called

`main`:
- Invokes the composition root
- Is the only place where exit codes are determined
- Contains no domain logic
- Contains no conditional business routing

---

## Result Handling at the Boundary

Use `Result` throughout the pipeline. Use cases return `Result<'a, string>` (or a typed error DU if needed). The mapper may return `Result<Command, string>` if argument validation is non-trivial.

Only at `main` does `Result` get translated to an exit code. Errors are written to `stderr`; success output to `stdout`.

```fsharp
// For use cases that produce output (e.g. a query)
let result =
    FindShipment.execute getShipment command

match result with
| Ok shipment ->
    printfn "%s -> %s" shipment.Origin shipment.Destination
    0
| Error msg ->
    eprintfn "Error: %s" msg
    1
```

Do not call `printfn` or `eprintfn` inside use cases or the router. Side effects for output belong at the boundary.

---

## Error Types

For simple CLIs, `Result<unit, string>` is sufficient. When error handling needs to be more structured — for example, distinguishing a not-found error from a validation error — define an error DU in the domain:

```fsharp
// In Shipping (domain project)
type ShipmentError =
    | NotFound of id: string
    | ValidationError of message: string
    | ConflictError of message: string
```

Use cases return `Result<'a, ShipmentError>`. The CLI boundary maps `ShipmentError` cases to appropriate messages and exit codes. The mapping lives in `Program.fs` or a dedicated `ErrorHandler` module if it grows large.

```fsharp
let toExitCode (error: ShipmentError) : int =
    match error with
    | NotFound id ->
        eprintfn "Shipment not found: %s" id
        2
    | ValidationError msg ->
        eprintfn "Validation error: %s" msg
        3
    | ConflictError msg ->
        eprintfn "Conflict: %s" msg
        4
```

Using distinct exit codes allows shell scripts to branch on specific failure conditions.

---

## Environment Configuration

Read environment variables and configuration in `main`, not in adapters or use cases. Pass values as parameters.

```fsharp
let connectionString =
    match System.Environment.GetEnvironmentVariable("DB_CONNECTION") with
    | null | "" ->
        eprintfn "DB_CONNECTION environment variable is not set."
        System.Environment.Exit(1)
        failwith "unreachable"
    | s -> s
```

Fail fast on missing configuration before any domain logic runs. This surfaces configuration problems immediately rather than at first use.

---

## Conventions

- One `CliArguments` DU per CLI. Subcommands are separate DUs (`RegisterArgs`, `CancelArgs`) composed into it.
- The `Mapper` module is the only file that imports both `Args` and domain types.
- Use cases take their dependencies as the first curried parameters so they can be partially applied in `main`.
- Return `Result` from use cases; never `unit` when something can fail.
- Exit codes: `0` success, `1` general error, `2`+ domain-specific errors if needed.
- Write success output to `stdout`, errors to `stderr`.
- Do not add logging infrastructure to the CLI host beyond `eprintfn`. If structured logging is needed, inject a logger into adapters from `main`.

---

## What Does Not Belong in the CLI Host

- Business logic — belongs in the domain
- Data access — belongs in adapters (`Shipping.Adapter.*`)
- Validation of domain invariants — belongs in the domain
- Mapping between domain types — belongs in the domain

The CLI host parses, maps, routes, and translates results. That is its entire responsibility.
