---
description: Style guide for test design using ABC testing (Acceptance, Building, Communication) and behaviour-focused unit tests.
tags: [testing, style-guide]
---

# Testing Style Guide

This guide uses two complementary ideas for test design:

- Devon Burriss's maintainable unit test approach: test behavior, prefer
  in-memory dependencies, and build small test tooling that keeps setup clear.
- ABC testing: separate Acceptance, Building, and Communication tests by the
  question each test is answering.

## Default test split

- **Acceptance tests** answer: "Did I build the right thing?"
- **Building tests** are temporary scaffolding while designing or debugging.
- **Communication tests** answer: "Did this boundary or mapping still speak the
  right language?"

For domain/usecase work in this repo:

- Prefer fast, in-memory **Acceptance** tests for usecase behavior.
- Keep **Communication** tests small and focused on IO contracts, mappings, and
  formatting.
- Treat **Building** tests as disposable. Delete them when they stop paying for
  themselves.

## Maintainable test rules

- Test behavior, not structure.
- Use the natural boundary as the test entry point.
- Prefer in-memory doubles over mocks for domain/usecase tests.
- Use small builders/helpers to make setup explicit and composable.
- Assert the outcome the behavior promises.
- Only assert IO details in a usecase test when those details are part of the
  claimed behavior.
- Put adapter, mapping, serialization, filesystem shape, process, harness, and
  formatter contract checks in Communication tests.
- Prefer tests to be self-contained. Avoid shared fixtures that accumulate setup for multiple tests. Instead, build helpers that enable composing just the state needed for each test.

## Test double terminology

Use Meszaros's terminology precisely:

- **Test Double** is the generic term for any pretend object used in a test.
- **Dummy** values fill a parameter list but are not actually used.
- **Fake** implementations work, but use shortcuts that make them unsuitable for
  production.
- **Stub** doubles return canned answers.
- **Spy** doubles record information about how they were used.
- **Mock** doubles carry call expectations that form part of the test's
  specification.

In this repo:

- `InMemoryFileSystem`, `InMemoryTaskStore`, `InMemoryBacklogStore`,
  `InMemoryViewStore`, `InMemoryPortfolioConfig`, and
  `InMemoryProductConfig` are fakes.
- `InMemoryAgentHarness` is a stub when it returns canned responses and a spy
  when it records prompts.
- Scenario and dependency builders usually compose a mix of doubles, so they
  should be described as test double builders or dependency builders rather
  than "fakes".

Do not use `mock` as the generic word for all test doubles. Use `mock` only
when you mean a Meszaros mock with call expectations.

## Naming convention

Prefer the Burriss-style accessors when creating test data and test doubles:

- `A.<Thing>` for entity and value builders, for example `A.Task` or
  `A.Profile`.
- `Given.<Thing>` for dependency and scenario builders, for example
  `Given.TaskStore`, `Given.AgentHarness`, or `Given.Deps`.
- Use F# pipeline style for setup composition, for example
  `Given.Deps |> Deps.withTask task |> Deps.withPlanTemplate "template"`.

This keeps setup concise while still making the important parts of the scenario
explicit.

## Repo expectations

### Acceptance tests

Use Acceptance tests for domain behavior at the natural entry point:

- Command operations: `Tasks.Plan.execute`, `Backlogs.Create.execute`,
  `Portfolios.RegisterProduct.execute`, etc.
- Query operations: `Tasks.Query.list`, `Tasks.Query.getDetail`,
  `Portfolios.Query.resolveProduct`, etc.

Acceptance tests should:

- Prefer in-memory test doubles so they stay fast and reliable.
- Set up only the dependencies required by the function under test.
- Use focused helpers such as `Given.Deps |> Deps.withTask task` rather than
  broad fixtures.
- Assert the returned result first.
- Assert persisted or recorded side effects only when the behavior being proved
  includes those externally visible details.

Acceptance tests should not:

- Assert collaborator call counts.
- Assert internal orchestration details.
- Reach through the natural boundary to test CLI dispatch when the behavior
  belongs to a domain usecase.

### Communication tests

Use Communication tests when the important question is about a boundary or
contract rather than domain behavior.

Examples in this repo include:

- Files written to the expected paths.
- YAML/JSON/text/table formatting shape.
- Adapter mappings between domain values and external representations.
- Harness request/response contract handling.
- Filesystem lookup rules and path translation.

Communication tests should be minimal and focused. If a test mixes domain logic
and IO contract assertions, split it so the behavior stays in Acceptance tests
and the boundary check stays in Communication tests.

## Test tooling style

Prefer simple in-memory doubles and small setup helpers.

Good patterns:

- `Given.Deps` creates the minimal dependency set.
- `A.Task`, `A.BacklogItem`, and `A.Profile` make test data setup readable.
- `Deps.withTask`, `Deps.withBacklogItem`, `Deps.withPortfolio`, and
  `Deps.withAgentResponse` add just the state needed for one scenario.
- Builders compose with `|>` so each test states only what matters.

Avoid large shared fixtures that hide important setup or force unrelated data
into every test.

## Examples

### Acceptance-style usecase test

```fsharp
[<Fact>]
let ``plan execute returns plan path for approved task`` () =
    let task =
        A.Task
        |> Task.withId "feat-a"
        |> Task.approved

    let deps =
        Given.Deps
        |> Deps.withTask task
        |> Deps.withPlanTemplate "template"
        |> Deps.withAgentResponse "plan text"

    let input = { TaskId = TaskId.create "feat-a"; UseAi = true; Debug = false }

    match Tasks.Plan.execute input |> Effect.run deps with
    | Ok output -> Assert.Equal("/repo/.itr/tasks/feat-a/plan.md", output.PlanPath)
    | Error e -> failwithf "expected Ok, got %A" e
```

Why this is good:

- It enters through the natural boundary.
- It uses in-memory doubles.
- It asserts the behavior promised by the usecase.
- It does not assert which helper or dependency method was called internally.

### Communication-style contract test

```fsharp
[<Fact>]
let ``plan formatter renders json with planPath`` () =
    let output = { PlanPath = "/repo/.itr/tasks/feat-a/plan.md" }

    let text = captureOutput (fun () -> Cli.Tasks.Plan.Format.result Json output)

    Assert.Contains("\"planPath\":\"/repo/.itr/tasks/feat-a/plan.md\"", text)
```

Why this is good:

- It checks a boundary contract.
- It stays small.
- It does not retest the domain behavior that produced `PlanPath`.

## Practical rule of thumb

When writing a test, ask:

1. Am I proving feature behavior at a natural boundary? Use an Acceptance test.
2. Am I proving an external contract, mapping, or format? Use a Communication
   test.
3. Am I only using this test to discover a design or pin down an implementation
   detail temporarily? It is a Building test and may be deleted later.