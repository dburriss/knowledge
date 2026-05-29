---
description: How stratified design applies to F# projects using clean/ports-and-adapters architecture, covering layer mapping, rate of change, layer-skipping smells, and a code-review checklist.
tags: [architecture, fsharp, style-guide]
---

# Stratified Design in F# / Clean Architecture

Date: 2026-05-29

---

## What Stratified Design Is

From SICP (pg 190):

> Stratified design is the notion that a complex system should be structured as a sequence of levels that are described using a sequence of languages. Each level is constructed by combining parts that are regarded as primitive at that level, and the parts constructed at each level are used as primitives at the next level. The language used at each level of a stratified design has primitives, means of combination, and means of abstraction appropriate to that level of detail.

The practical implication is that every piece of code should be written at a single level of abstraction. A function or module should either build a new abstraction from lower-level parts, or use existing abstractions to do something meaningful — not both at once. Dependencies flow strictly downward. Nothing at a given level should depend on something at the same level; if it does, coordination is missing from a layer above.

---

## Mapping Layers to the Architecture

Stratified design applies at two scales in a clean architecture project: across the project boundary (macro) and within the domain project itself (micro). Both scales follow the same rule: each layer is built from the layer beneath it, and never the other way around.

See `clean-architecture-style-guide.md` for the full project structure and naming conventions. This document treats that structure as given and explains the design principle underneath it.

### Macro Scale: Across Projects

```
Shipping.Api / Shipping.Cli        (host — outermost, changes most)
  └── Shipping.Adapter.Postgres
  └── Shipping.Adapter.Stripe      (adapters — change when infrastructure changes)
        └── Shipping                (domain — innermost, changes least)
```

Each project is a layer. The domain project is the stable base. Adapters compose domain types and port interfaces into concrete infrastructure implementations. Hosts compose adapters and use cases into a runnable application.

This is stratification made visible at the filesystem level. You can tell the layer of a file from its project name alone.

### Micro Scale: Within the Domain Project

Inside `Shipping`, there is a second stratification that the project boundary does not enforce — only discipline does:

```
Use cases          (execute: orchestrate domain objects through ports)
  └── Domain services  (coordinate multiple entities or value objects)
        └── Entities   (identity, lifecycle, invariants)
              └── Value objects  (validation, equality by value, no identity)
                    └── Primitive/core types  (ShipmentId, Weight, Address)
```

Each level uses the level below as its vocabulary. Value objects are built from primitive types. Entities are built from value objects. Use cases call entity methods and domain services, not raw primitives. A use case that directly constructs a `Weight` from a `float` and immediately passes it to a repository has skipped a layer.

---

## Rate of Change as a Design Signal

Lower layers should change slowly. Upper layers should change often. If the inverse is true, something is misplaced.

- The `ShipmentId` discriminated union should almost never change.
- The `Address` value object might change when address validation rules change.
- The `RegisterShipment` use case changes every time a business rule around registration changes.
- `Shipping.Api` route handlers change when the HTTP contract changes.

If a value object is changing every sprint, ask whether it is carrying business logic that belongs one level up. If a use case never changes, ask whether it is so thin that it belongs inside an entity. Rate of change is a diagnostic. An unusually volatile low-level type is a symptom, not a root cause.

---

## Layer Skipping as a Code Smell

Layer skipping happens when a high-level module reaches past its immediate layer to operate directly on primitives that should be encapsulated lower down. The code works, but the intermediate abstraction that should exist is missing.

### Example: skipping the value-object layer

```fsharp
// BAD — use case builds and validates a Weight from raw floats inline.
// The concept of "valid weight" belongs in a value object, not here.
let execute (repo: IShipmentRepository) (command: Command) =
    async {
        if command.WeightKg <= 0.0 then
            return Error "Weight must be positive"
        elif command.WeightKg > 10_000.0 then
            return Error "Weight exceeds maximum"
        else
            let shipment = Shipment.create command.TrackingNumber command.WeightKg
            do! repo.save shipment
            return Ok shipment.Id
    }
```

```fsharp
// GOOD — Weight encapsulates its own rules. The use case speaks at the domain level.
let execute (repo: IShipmentRepository) (command: Command) =
    async {
        match Weight.create command.WeightKg with
        | Error reason -> return Error reason
        | Ok weight ->
            let shipment = Shipment.create command.TrackingNumber weight
            do! repo.save shipment
            return Ok shipment.Id
    }
```

In the bad version, the use case knows about the rules for a valid weight. If those rules change, the use case changes — even though the business operation (`RegisterShipment`) did not change. The good version treats `Weight.create` as the boundary: the use case knows only that a weight either validates or it does not.

The tell-tale sign of layer skipping is primitive types appearing in the body of a use case alongside validation or transformation logic that belongs in a type one level lower.

---

## Same-Level Dependencies as a Code Smell

Modules at the same layer must not depend on each other. This applies at both scales.

At the macro scale, `clean-architecture-style-guide.md` states this explicitly: adapters do not depend on other adapters. If `Shipping.Adapter.Stripe` needs data from `Shipping.Adapter.Postgres`, that coordination belongs in a use case, mediated through ports. Connecting two adapters directly bypasses the domain and turns the adapter layer into an ad-hoc integration platform.

At the micro scale, the same rule applies to modules within the domain project. Value objects should not reference other value objects in ways that create horizontal coupling. Entities should not call other entities directly; if two entities need to collaborate, a domain service sits above both of them.

### Example: same-level coupling between entities

```fsharp
// BAD — Shipment reaches into Customer to compute a discount. Two entities coupled horizontally.
module Shipping.Domain.Shipment

let computeCost (customer: Customer) (shipment: Shipment) =
    let base' = Weight.toKg shipment.Weight * 1.50m
    let discount = Customer.loyaltyDiscount customer   // Shipment depends on Customer
    base' * (1.0m - discount)
```

```fsharp
// GOOD — a domain service sits above both entities and owns the coordination.
module Shipping.Domain.ShippingCostService

let compute (loyaltyRate: decimal) (shipment: Shipment) =
    let base' = Weight.toKg shipment.Weight * 1.50m
    base' * (1.0m - loyaltyRate)
```

The use case retrieves the loyalty rate from `Customer` and passes it down. Neither entity knows about the other.

---

## Naming Layers

Names should signal the layer without requiring the reader to inspect dependencies.

| Layer | Naming conventions |
|---|---|
| Primitive / core types | Named noun: `ShipmentId`, `Weight`, `Address`, `TrackingNumber` |
| Value objects | Named noun with `create` / `validate` smart constructors: `Weight.create`, `Address.validate` |
| Entities | Named noun with domain verbs: `Shipment.register`, `Shipment.cancel`, `Shipment.reroute` |
| Domain services | Named as `*Service` or `*Calculator` or `*Policy`: `ShippingCostService`, `RoutePolicy` |
| Use cases | Named as imperative verb phrase: `RegisterShipment`, `CancelOrder`, `RerouteShipment` |
| Ports | Prefixed with `I`, named as capability: `IShipmentRepository`, `IPaymentGateway`, `IClock` |
| Adapters | Named after technology and port: `PostgresShipmentRepository`, `StripePaymentGateway` |

A reviewer who sees a module called `RegisterShipment` should immediately expect: it calls ports, it calls domain methods, and it contains no primitive manipulation. A module called `Weight` should contain validation logic and nothing that touches a repository.

If a name does not signal its layer, the layer may not be clear to the author either.

---

## Practical Checklist for Code Review

Use these during review to detect stratification violations:

- [ ] Does the use case operate on domain types exclusively, or does it manipulate primitives that should be encapsulated in a value object?
- [ ] Is any validation logic duplicated across multiple use cases that should live in a value object or entity instead?
- [ ] Does the domain project reference any infrastructure library? (Automatic reject; see `clean-architecture-style-guide.md`.)
- [ ] Does any adapter reference another adapter directly? Coordination belongs in a use case.
- [ ] Does any entity reference another entity directly? If so, should a domain service own that coordination?
- [ ] Is a low-level type (value object, entity) changing frequently? If yes, it may be carrying logic that belongs one level higher.
- [ ] Can you read a use case signature and know exactly what external systems it touches, without reading its body?
- [ ] Does every module have a name that signals its layer (type/value object/entity/service/use case)?
- [ ] Is there logic in a controller or handler that belongs in a use case or domain entity?
- [ ] Are two modules at the same level coupled? If yes, introduce a layer above them that owns the coordination.
