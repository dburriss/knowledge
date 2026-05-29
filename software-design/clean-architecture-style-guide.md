---
description: Opinionated style guide for structuring projects using clean/onion/ports-and-adapters architecture, with naming conventions, use case patterns, and anti-patterns.
tags: [architecture, style-guide]
---

# Clean Architecture Style Guide

Date: 2026-05-29

This guide covers how to structure projects that follow clean/onion/ports-and-adapters architecture. The names differ; the principle is the same: the domain sits at the center, all dependencies point inward, and infrastructure is a detail.

---

## Overview

Clean architecture, onion architecture, and ports-and-adapters are three names for a single core idea: the business domain should not depend on how it is deployed, persisted, or exposed. Infrastructure details — databases, HTTP clients, message brokers, UI frameworks — are plugged in from the outside.

The payoff is a domain that is independently testable, replaceable at its edges, and honest about what it actually does.

---

## Core Principle

The domain is the center. Every dependency arrow points inward toward it.

```
Host (Api, Cli)
  └── Adapters (Postgres, Stripe, ...)
        └── Domain (Shipping, Pricing, ...)
```

- The domain knows nothing about adapters or hosts.
- Adapters know about the domain (they implement its ports).
- Hosts know about adapters (they wire everything together).

This is not a suggestion. It is the constraint that makes the architecture work.

---

## Project and Module Structure

### Naming conventions

Name the domain project after the domain itself. Not `Core`, not `Domain`, not `Application`. The domain is `Shipping`, `Pricing`, `Ordering`. Everything else is named relative to it.

| Project | Example | Role |
|---|---|---|
| Domain | `Shipping` | Entities, value objects, domain logic, port interfaces |
| Use cases (optional split) | `Shipping.UseCases` | One module/class per use case |
| Adapter | `Shipping.Adapter.Postgres` | Implements a port defined in the domain |
| Adapter | `Shipping.Adapter.Stripe` | Implements a port defined in the domain |
| Host | `Shipping.Api` | HTTP entry point; wires DI, no business logic |
| Host | `Shipping.Cli` | CLI entry point; wires DI, no business logic |

### What lives in each project

**Domain (`Shipping`)**

- Entities and value objects
- Domain logic and invariants
- Port interfaces (the abstractions adapters will implement)
- Domain events
- No references to any infrastructure library or framework

Prefer domain-specific types over primitives. A `ShipmentId` is not a `Guid`. A `Weight` is not a `float`. Honest types make invalid states unrepresentable and make the domain self-documenting. Wrapping a primitive costs almost nothing; passing the wrong primitive into the wrong parameter costs a production incident.

**Use cases (`Shipping` or `Shipping.UseCases`)**

- One use case per module (F#) or class (C#)
- Each exposes a single `execute` function or `Execute` method
- Depends only on the domain; never on adapters or hosts
- Orchestrates domain logic and calls ports by their interface

For small domains, use cases live directly in the domain project. Extract `Shipping.UseCases` only when the project gets large enough to warrant it.

**Adapters (`Shipping.Adapter.Postgres`, `Shipping.Adapter.Stripe`, ...)**

- Implement port interfaces defined in the domain
- Reference the domain project
- Reference any external library they need (EF Core, Npgsql, Stripe SDK, etc.)
- No business logic — only translation between the domain and the external system

**Hosts (`Shipping.Api`, `Shipping.Cli`)**

- Compose the application: register use cases, adapters, and any cross-cutting infrastructure via DI
- Translate external input (HTTP requests, CLI args) into use case calls
- Translate use case results into external output (HTTP responses, exit codes, stdout)
- No business logic — controllers and handlers are routing, not logic

---

## Dependency Rule

State this explicitly in code review and architecture decisions:

- The domain project has **zero external dependencies** beyond the base class library.
- Adapter projects depend on the domain. They do not depend on each other.
- Host projects depend on adapters and the domain. They wire everything together.

If a pull request adds an infrastructure package reference to the domain project, that is a violation. Reject it.

---

## Use Case Pattern

Use one use case per entry point to the application. Do not aggregate all use cases into a single `ApplicationServices` class. That pattern destroys discoverability and creates a god object.

Each use case has one job. Name it for what it does: `RegisterShipment`, `CancelOrder`, `ChargeCustomer`.

Make external dependencies visible at the use case level. If a use case reads from a database and calls a payment gateway, both ports should appear in its signature — not buried three layers down in a helper that pulls them from a service locator. A reader should be able to understand what a use case touches by looking at its inputs alone.

### F# — module with `execute` function

```fsharp
// Shipping/UseCases/RegisterShipment.fs

module Shipping.UseCases.RegisterShipment

open Shipping.Domain
open Shipping.Ports

type Command = {
    TrackingNumber: string
    Origin: Address
    Destination: Address
}

type Result =
    | Registered of ShipmentId
    | InvalidAddress of string

let execute (repo: IShipmentRepository) (clock: IClock) (command: Command) : Async<Result> =
    async {
        match Address.validate command.Origin, Address.validate command.Destination with
        | Error msg, _ | _, Error msg -> return InvalidAddress msg
        | Ok origin, Ok destination ->
            let shipment = Shipment.create (clock.Now()) command.TrackingNumber origin destination
            do! repo.save shipment
            return Registered shipment.Id
    }
```

### C# — class with `Execute` method

```csharp
// Shipping/UseCases/RegisterShipment.cs

namespace Shipping.UseCases;

public sealed class RegisterShipment
{
    private readonly IShipmentRepository _repository;
    private readonly IClock _clock;

    public RegisterShipment(IShipmentRepository repository, IClock clock)
    {
        _repository = repository;
        _clock = clock;
    }

    public async Task<Result> Execute(Command command, CancellationToken ct = default)
    {
        var originResult = Address.Validate(command.Origin);
        var destinationResult = Address.Validate(command.Destination);

        if (originResult.IsFailure) return Result.InvalidAddress(originResult.Error);
        if (destinationResult.IsFailure) return Result.InvalidAddress(destinationResult.Error);

        var shipment = Shipment.Create(_clock.Now(), command.TrackingNumber, originResult.Value, destinationResult.Value);
        await _repository.Save(shipment, ct);
        return Result.Registered(shipment.Id);
    }
}

public record Command(string TrackingNumber, string Origin, string Destination);

public abstract record Result
{
    public sealed record Registered(ShipmentId Id) : Result;
    public sealed record InvalidAddress(string Reason) : Result;
}
```

The use case owns its `Command` and `Result` types. The adapter or host translates into and out of them at the boundary.

---

## Ports and Adapters

A port is an interface defined in the domain. It describes what the domain needs; it says nothing about how it is satisfied.

An adapter is the implementation. It lives in an adapter project and bridges the domain's expectation to the actual external system.

### Port — defined in the domain

**F#**

```fsharp
// Shipping/Ports/IShipmentRepository.fs

module Shipping.Ports

open Shipping.Domain

type IShipmentRepository =
    abstract member save : Shipment -> Async<unit>
    abstract member findById : ShipmentId -> Async<Shipment option>
```

**C#**

```csharp
// Shipping/Ports/IShipmentRepository.cs

namespace Shipping.Ports;

public interface IShipmentRepository
{
    Task Save(Shipment shipment, CancellationToken ct = default);
    Task<Shipment?> FindById(ShipmentId id, CancellationToken ct = default);
}
```

### Adapter — implementing the port

**F#**

```fsharp
// Shipping.Adapter.Postgres/ShipmentRepository.fs

module Shipping.Adapter.Postgres.ShipmentRepository

open Npgsql
open Shipping.Domain
open Shipping.Ports

type PostgresShipmentRepository(connectionString: string) =
    interface IShipmentRepository with
        member _.save shipment =
            async {
                use conn = new NpgsqlConnection(connectionString)
                // ... SQL persistence logic
                ()
            }

        member _.findById id =
            async {
                use conn = new NpgsqlConnection(connectionString)
                // ... SQL query logic
                return None
            }
```

**C#**

```csharp
// Shipping.Adapter.Postgres/ShipmentRepository.cs

namespace Shipping.Adapter.Postgres;

public sealed class ShipmentRepository : IShipmentRepository
{
    private readonly string _connectionString;

    public ShipmentRepository(string connectionString)
    {
        _connectionString = connectionString;
    }

    public async Task Save(Shipment shipment, CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        // ... SQL persistence logic
    }

    public async Task<Shipment?> FindById(ShipmentId id, CancellationToken ct = default)
    {
        await using var conn = new NpgsqlConnection(_connectionString);
        // ... SQL query logic
        return null;
    }
}
```

---

## What Goes Where

| Thing | Lives in |
|---|---|
| Entity | Domain (`Shipping`) |
| Value object | Domain (`Shipping`) |
| Domain event | Domain (`Shipping`) |
| Port interface | Domain (`Shipping`) |
| Domain invariant / rule | Domain (`Shipping`) |
| Use case | Domain or `Shipping.UseCases` |
| Use case `Command` / `Result` types | Same project as use case |
| Adapter implementation | `Shipping.Adapter.*` |
| DB schema / migrations | `Shipping.Adapter.Postgres` |
| HTTP client / external SDK call | `Shipping.Adapter.*` |
| DI registration | Host (`Shipping.Api`, `Shipping.Cli`) |
| Controller / handler | Host |
| Request/response DTO | Host |
| Configuration loading | Host |
| Cross-cutting middleware | Host |

---

## Anti-Patterns to Avoid

**God `ApplicationServices` class**

Do not put all use cases as methods on a single service class. It becomes a dumping ground, grows without bound, and makes the application's entry points opaque. Use one use case per module or class.

**Infrastructure leaking into the domain**

The domain must not import EF Core, Dapper, Npgsql, HttpClient, or any other infrastructure library. If you see a package reference to an infrastructure library in the domain project, the dependency rule has been broken. Move it to an adapter.

**Business logic in controllers or handlers**

Controllers translate HTTP into use case calls. They do not make decisions. Validation of input shape belongs in the controller; validation of business rules belongs in the domain. If a controller has an `if` statement that enforces a business rule, that rule belongs in a use case or domain entity.

**Anemic domain with fat use cases**

Use cases orchestrate; they do not contain domain logic. If a use case is calculating prices, enforcing invariants, or running business rules, those belong in domain entities or value objects. Use cases call domain methods — they do not replace them.

**Adapters depending on other adapters**

Adapters implement domain ports. They do not call each other. If one adapter needs behavior from another, that coordination belongs in a use case or domain service, mediated through ports.
