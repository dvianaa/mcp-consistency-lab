# DurableMCP Domain Model

**Status:** Draft

**Scope:** engine-core

**Audience:** Contributors, reviewers, thesis supervisor, and future maintainers

## 1. Purpose

DurableMCP coordinates workflows composed of operations executed against independent external systems. The first
reference use case is a corporate trip workflow:

1. book a flight
2. reserve a hotel
3. capture a payment

Each operation may succeed, fail before producing an effect, or produce an effect without the coordinator receiving a
conclusive response. DurableMCP must represent these situations explicitly and coordinate compensation or reconciliation
when required.

The domain model is intentionally independent of:

- MCP
- Spring
- HTTP
- databases
- message brokers
- serialization formats
- LLM providers

MCP is an integration mechanism used by an adapter. Its not part of the core workflow model.

## 2. Problem Statement

A workflow executed across autonomous systems does not form a traditional ACID transaction. External systems may:

- be managed by different organizations
- expose unrelated APIs and consistency guarantees
- become unavailable independently
- process requests after the caller times out
- return incomplete or ambiguous results
- not support rollback
- provide compensating operations with different semantics from the original operation

DurableMCP therefore provide a deterministic and inspectable execution model that can:

- track the progress of a workflow
- distinguish confirmed success, confirmed failure and unknown outcomes
- select previously successful steps for compensation
- execute compensations in reverse order
- support future recovery from persisted execution history
- surface situations that cannot be resolved safely without intervention

## 3. Ubiquitous Language

The following terms must be used consistently in code, documentation, tests, and the thesis.

| Term                    | Meaning                                                                                                              |
|-------------------------|----------------------------------------------------------------------------------------------------------------------|
| **Workflow**            | A versioned definition of an ordered business process composed of steps.                                             |
| **Workflow Definition** | The immutable description of a workflow, including its identifier, version, name, and ordered steps.                 |
| **Workflow Execution**  | One concrete run of a workflow definition. Multiple executions of the same definition are independent.               |
| **Step**                | One logical unit of work in a workflow.                                                                              |
| **Step Definition**     | The immutable description of a step, including its operation and optional compensation or reconciliation operations. |
| **Step Execution**      | The runtime state of one step within one workflow execution.                                                         |
| **Operation**           | An externally executed action, such as booking a flight or capturing a payment.                                      |
| **Operation Reference** | A protocol-independent identifier for an external operation.                                                         |
| **Compensation**        | A business operation intended to mitigate or semantically reverse the effect of a previously successful operation.   |
| **Reconciliation**      | An operation used to discover the actual outcome of an operation whose result is unknown.                            |
| **Attempt**             | One dispatch of an operation or compensation. A step may require multiple attempts in later versions.                |
| **Confirmed Success**   | The coordinator received sufficient evidence that the operation completed successfully.                              |
| **Confirmed Failure**   | The coordinator received sufficient evidence that the operation did not complete successfully.                       |
| **Unknown Outcome**     | The coordinator cannot determine whether the operation produced an external effect.                                  |
| **Terminal State**      | A state in which no normal transition is allowed without an explicit recovery or administrative action.              |
| **Intervention**        | A manual or administrative action required because the system cannot determine a safe automatic resolution.          |
| **Saga**                | A coordination strategy in which successful local operations are compensated when a later operation fails.           |

A compensation may create new side effects, incur fees, preserve audit records, or complete asynchronously. Therefore,
COMPENSATED does not mean that the world has returned to its exact original state.

## 4. Domain Boundary

The `engine-core` module owns:

- workflow definitions;
- workflow executions;
- step definitions;
- step executions;
- workflow and step state semantics;
- domain invariants;
- compensation ordering;
- protocol-independent operation references;
- domain-level failure classification.

The `engine-core` module does not own:

- MCP sessions or JSON-RPC messages;
- HTTP clients;
- Spring components;
- PostgreSQL schemas;
- serialization;
- authentication;
- observability exporters;
- Docker configuration;
- retry scheduling;
- network fault injection.

Infrastructure modules will implement ports required by the core.

Conceptually:

```text
Host or Agent
      |
      v
MCP Adapter / Gateway
      |
      v
Durable Workflow Engine
      |
      +---- Execution Journal Port
      |
      +---- Operation Invoker Port
      |
      +---- Workflow Definition Repository Port
```

The core must be testable without starting Spring, a database, a container, or an MCP server.

## 5. Workflow Definition

A `WorkflowDefinition` is an immutable, versioned description of an ordered process.

It contains at least:

- a workflow definition identifier;
- a positive integer version;
- a human-readable name;
- an ordered, non-empty list of step definitions.

Example:

```text
WorkflowDefinition
  id: book-corporate-trip
  version: 1
  name: Book Corporate Trip
  steps:
    1. book-flight
    2. reserve-hotel
    3. capture-payment
```

### Definition invariants

A workflow definition is invalid when:

- its identifier is absent or blank;
- its version is lower than `1`;
- its name is absent or blank;
- its step list is null or empty;
- the step list contains null elements;
- two steps share the same step identifier;
- the order of its steps is not deterministic.

A definition is immutable after construction. Changing the process creates a new version instead of modifying an
existing definition in place.

For example:

```text
book-corporate-trip:1
book-corporate-trip:2
```

An execution always retains the exact definition identifier and version with which it started.

---

## 6. Step Definition

A `StepDefinition` describes one logical unit of work.

It contains at least:

- a unique step identifier within the workflow;
- a primary operation reference;
- an optional compensation operation reference;
- an optional reconciliation operation reference.

Example:

```text
StepDefinition
  id: book-flight
  operation: flights/book
  compensation: flights/cancel
  reconciliation: flights/get-booking
```

### Primary operation

The primary operation produces the intended business effect.

Examples:

- `flights/book`;
- `hotels/reserve`;
- `payments/capture`.

### Compensation operation

The compensation mitigates or semantically reverses a confirmed effect.

Examples:

- `flights/cancel`;
- `hotels/cancel`;
- `payments/refund`.

A compensation is optional because not every external operation is reversible. However, a workflow containing an
irreversible operation may require a stricter execution policy in future versions.

### Reconciliation operation

The reconciliation operation determines the real external state after an ambiguous outcome.

Examples:

- `flights/get-booking`;
- `hotels/get-reservation`;
- `payments/get-status`.

Reconciliation is different from compensation:

- reconciliation observes or queries;
- compensation changes external state.


