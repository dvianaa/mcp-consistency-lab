# DurableMCP Domain Model

**Status:** Draft

**Scope:** engine-core

**Audience:** Contributors, reviewers, thesis supervisor, and future maintainers

---

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

---

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

---

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

---

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

---

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

```
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

### Operation reference

An operation reference identifies an operation without encoding specific MCP types. Conceptually:

```
OperationRef
  provider: payments
  operation: capture
```

The MCP adapter may later translate this into an MCP tool name like: `payments.capture`
The core most not depend on that naming convention.

---

## 7. Workflow Execution

A `WorkflowExecution` represents one concrete run of a workflow definition. It must contain at least:

- an execution identifier
- the workflow definition identifier
- the workflow definition version
- the current workflow status
- one step execution for each step definition

for ex:

```
WorkflowExecution
  executionId: execution-2026-0001
  definition: book-corporate-trip:1
  status: RUNNING
  steps:
    - book-flight: SUCCEEDED
    - reserve-gotel: DISPATCHED
    - capture-payment: PENDING
```

### Workflow status vocabulary

The initial model must use the following states:

| Status                  | Meaning                                                                           |
|-------------------------|-----------------------------------------------------------------------------------|
| `CREATED`               | The execution exists but has not started.                                         |
| `RUNNING`               | The workflow is progressing through its forward steps.                            |
| `COMPENSATING`          | Previously successful steps are being compensated in reverse order.               |
| `COMPLETED`             | Every required forward step has confirmed success.                                |
| `COMPENSATED`           | All required compensations completed successfully.                                |
| `FAILED`                | The execution reached an unrecoverable confirmed failure under the active policy. |
| `REQUIRES_INTERVENTION` | The system cannot determine a safe automatic resolution.                          |
| `CANCELLED`             | The execution was cancelled before producing effects that require compensation.   |

The detailed transition rules will be documented on separately in `state-machine.md`

---

## 8. Step Execution

A `StepExecution` represents the runtime state of one step within one workflow execution. Initially containing:

- the step identifier
- the current step status
- an attempt count

Future iterations may add:

- input
- output
- idempotency key
- failure details
- dispatch timestamps
- completion timestamps
- reconciliation result
- compensation attempts

### Step status vocabulary

| Status                    | Meaning                                                                |
|---------------------------|------------------------------------------------------------------------|
| `PENDING`                 | The step has not been dispatched.                                      |
| `DISPATCHED`              | The operation was sent, but no conclusive result has been recorded.    |
| `SUCCEEDED`               | The operation has confirmed success.                                   |
| `FAILED`                  | The operation has confirmed failure.                                   |
| `UNKNOWN`                 | The outcome cannot be determined safely.                               |
| `COMPENSATION_PENDING`    | Compensation is required but has not been dispatched.                  |
| `COMPENSATION_DISPATCHED` | The compensation was sent, but no conclusive result has been recorded. |
| `COMPENSATED`             | The compensation has confirmed success.                                |
| `COMPENSATION_FAILED`     | The compensation has confirmed failure.                                |

The idea is to deliberately separate the workflow and step status. for ex:

```
Workflow: COMPENSATING

book-flight: COMPENSATION_PENDING
reserve-hotel: COMPENSATED
capture-payment: FAILED
```

A single enum will not represent accurately both the global workflow and each local operation lifecycles.

---

## 9. Outcome Semantics

Every externally dispatched operation has one of three fundamental outcomes from the coordinator perspective.

### 9.1 Confirmed success

The coordinator has sufficient evidence that the operation succeeded. for ex:

`payments.capture returned a successful result with paymentId=pay-123`

In this case the step can move to `SUCCEEDED`.

### 9.2 Confirmed failure

The coordinator has sufficient evidence that the operation dis not succeed. For ex:

`hotels.reserve returned ROOM_UNAVAILABLE before creating a reservation`

the step may move to `FAILED`.

### 9.3 Unknown outcome

The coordinator cannot determine if an external effect occurred.

for ex:

```
1. the gateway sends payments.capture
2. the payment service captures the payment
3. the response is lost
4. the gateway times out or crashes
```

in this scenario the step must not be classified automatically as `SUCCESS` or `FAILURE`. its the best to become
`UNKNOWN`

this cases may later be resolved with:

- an idempotent retry using the same key
- reconciliation against the external service
- a domain specific recovery policy
- manual intervention

---

## 10. Failure Classification

The domain must distinguish the following failure categories, even if the first implementation does not yet model every
detail.

| Failure type                          | Example                             | Outcome certainty         |
|---------------------------------------|-------------------------------------|---------------------------|
| **Business failure**                  | Room unavailable                    | Usually confirmed         |
| **Validation failure**                | Invalid payment amount              | Confirmed                 |
| **Transport failure before dispatch** | Connection could not be established | Usually confirmed failure |
| **Timeout after dispatch**            | Response not received               | Potentially unknown       |
| **Connection loss after dispatch**    | Socket closed during response       | Potentially unknown       |
| **Remote process crash**              | Server stops during execution       | Potentially unknown       |
| **Coordinator crash**                 | Gateway stops after dispatch        | Potentially unknown       |
| **Compensation failure**              | Refund service unavailable          | Confirmed or unknown      |
| **Reconciliation failure**            | Status endpoint unavailable         | Outcome remains unknown   |

A timeout is not equivalent to a business failure.

This distinction is a central part of the thesis and must be reflected in future recovery policies and experiments.

 ---

## 11. Compensations Semantics

Compensation will follow Saga-Style reverse ordering.

for ex, the sequence:

```
1. book-flight
2. reserve-hotel
3. capture-payment
```

If `capture-payment` fails with a confirmed failure after the two first steps succeeded, compensation candidates are:

```
1. reserve-hotel
2. book-flight
```

the failed step is not compensated because no effect was confirmed.

### Compensation candidate rule

Onlt a step with confirmed success is automatically eligible for compensation. The following states are not compensation
candidates:

- `PENDING`
  `DISPATCHED`
- `FAILED`
- `UNKNOWN`
- `COMPENSATION_FAILED`

if `UNKNOWN` the step must be first reconciled or handled by a recovery.

### Compensation does not imply exact restoration

after compensation there may be multiple scenarios:

- cancellation fees may exist
- the audit trail will remain
- refunds may be asynchronous
- external identifiers may remain valid
- temporary inconssistency may be visible

so, `COMPENSATED` is supposed to mean that the configured compensating actions completed successfully. It does not prove
that the complete external world is byte-for-byte equal to its original state.

## 12. Core Invariants

The domain model must enforce the following invariants.

### Definition invariants

1. A workflow definition has a non-blank identifier.
2. A workflow definition has a version greater than or equal to `1`.
3. A workflow definition contains at least one step.
4. Step identifiers are unique within a workflow definition.
5. Step order is stable and deterministic.
6. Definitions are immutable.

### Execution invariants

1. A workflow execution references one exact definition version.
2. A workflow execution contains exactly one step execution per defined step.
3. Step execution order matches definition order.
4. A workflow cannot be `COMPLETED` unless all required steps are `SUCCEEDED`.
5. A workflow cannot be `COMPENSATED` while a required compensation is pending, dispatched, failed, or unknown.
6. A workflow containing an unresolved `UNKNOWN` step cannot be declared `COMPLETED`.
7. Compensation candidates are selected from successful steps in reverse order.
8. A terminal state cannot be changed through a normal transition.
9. State changes occur through intention-revealing domain methods, not generic setters.
10. The engine must never convert uncertainty into certainty without evidence or an explicit policy.

---

## 13. Initial Execution Model

The first implementation supports only linear, sequential workflows.

A step is not dispatched until the previous step has confirmed success.

```text
book-flight
      |
      v
reserve-hotel
      |
      v
capture-payment
```

The first version does not support:

- parallel branches;
- conditional branches;
- loops;
- dynamic graph mutation;
- subworkflows;
- choreography;
- distributed locking between external participants;
- Two-Phase Commit;
- Three-Phase Commit.

Sequential execution is sufficient to investigate:

- partial failures;
- ambiguous outcomes;
- compensation ordering;
- durable execution history;
- coordinator recovery;
- idempotent retries;
- reconciliation.

---

## 14. Reference Scenario

### Workflow definition

```text
id: book-corporate-trip
version: 1

steps:
  - id: book-flight
    operation: flights/book
    compensation: flights/cancel
    reconciliation: flights/get-booking

  - id: reserve-hotel
    operation: hotels/reserve
    compensation: hotels/cancel
    reconciliation: hotels/get-reservation

  - id: capture-payment
    operation: payments/capture
    compensation: payments/refund
    reconciliation: payments/get-status
```

### Successful execution

```text
book-flight      SUCCEEDED
reserve-hotel    SUCCEEDED
capture-payment  SUCCEEDED

workflow         COMPLETED
```

### Confirmed failure with compensation

```text
book-flight      SUCCEEDED
reserve-hotel    FAILED
capture-payment  PENDING

workflow         COMPENSATING
```

Compensation order:

```
book-flight -> flights/cancel
```

After confirmed compensation:

```
book-flight      COMPENSATED
reserve-hotel    FAILED
capture-payment  PENDING

workflow         COMPENSATED
```

### Ambiguous payment outcome

```
book-flight      SUCCEEDED
reserve-hotel    SUCCEEDED
capture-payment  UNKNOWN

workflow         REQUIRES_INTERVENTION
```

The workflow must not immediately refund or retry the payment without first applying a safe policy.

Possible later resolution:

```
payments/get-status -> CAPTURED
```

The system may then treat the payment as successful and continue according to the workflow policy.

---

## 15. Guarantees Targeted by the Project

The complete project aims to provide the following guarantees under documented assumptions:

- explicit and inspectable execution state;
- deterministic ordering for linear workflows;
- durable recording of execution intent and outcomes;
- recovery of the coordinator after a crash;
- reverse-order compensation for confirmed successful steps;
- safe representation of ambiguous outcomes;
- idempotency-aware retries;
- reconciliation before making unsafe assumptions;
- convergence to `COMPLETED`, `COMPENSATED`, or `REQUIRES_INTERVENTION` when dependencies eventually become available.

These guarantees depend on cooperation from external operations.

For example, safe automatic recovery may require at least one of:

- idempotency support;
- a queryable external status;
- a compensation operation;
- a stable business identifier.

Without those capabilities, intervention may be the only correct result.

---

## 16. Non-Goals

The initial domain model does not attempt to provide:

- universal ACID transactions across MCP servers;
- transparent rollback of arbitrary tools;
- exactly-once message delivery;
- automatic compensation discovery;
- automatic inference of business semantics from tool names;
- Byzantine fault tolerance;
- consensus between participants;
- strong isolation between concurrent workflows;
- a general-purpose BPMN engine;
- a visual workflow designer;
- LLM-based recovery decisions.

The LLM or host may choose which workflow to start. It must not decide low-level recovery semantics for an execution
already in progress.

---

## 17. Assumptions

The first prototype assumes:

1. external systems fail independently;
2. failures are crash, timeout, transport, or business failures rather than malicious Byzantine behavior;
3. the workflow definition is stable during one execution;
4. the engine can assign a globally unique execution identifier;
5. operation identifiers remain stable for the duration of an experiment;
6. the execution journal will eventually be available;
7. configured compensations and reconciliation operations accurately describe the external domain;
8. the coordinator may crash at any point;
9. a request may be processed even when its response is not received;
10. external state is the source of truth when reconciliation is required.

These assumptions must later be validated or explicitly stated in the thesis evaluation.

---

## 18. Open Decisions for Later Iterations

The following questions are intentionally deferred:

- How are operation inputs and outputs represented?
- How are outputs from one step mapped into later steps?
- How are compensation arguments derived?
- Which workflow definition format is used: Java API, YAML, JSON, or another DSL?
- How are idempotency keys generated and propagated?
- Which retry and backoff policies are supported?
- How is a lease acquired when multiple engine instances recover the same execution?
- Which data is stored as events and which data is stored as a current-state projection?
- How are long-running operations represented?
- Can a workflow be resumed after `REQUIRES_INTERVENTION`?
- How are irreversible operations handled?
- How is definition compatibility managed across software releases?

These decisions must not be introduced into the Week 1 model unless they are required by a core invariant.

---

This document is considered reflected in the implementation when:

- `engine-core` has no dependency on MCP, Spring, or persistence;
- workflow definitions are immutable and versioned;
- workflow and step executions are modeled separately;
- workflow and step states are modeled separately;
- `UNKNOWN` is a first-class step state;
- state changes use domain methods instead of generic setters;
- duplicate step identifiers are rejected;
- completion requires confirmed success of every required step;
- compensation candidates are returned in reverse order;
- uncertain outcomes cannot be silently converted into success or failure;
- the reference trip workflow can be represented in unit tests.

---

## 20. Summary

DurableMCP models distributed tool execution as a durable, compensable workflow rather than as a transparent ACID
transaction.

The core principles are:

1. keep the domain independent from MCP and infrastructure;
2. distinguish workflow definition from workflow execution;
3. distinguish workflow state from step state;
4. treat unknown outcomes as a normal distributed-systems condition;
5. compensate only confirmed effects;
6. never claim stronger guarantees than the participating systems can support;
7. make unsafe situations visible through `REQUIRES_INTERVENTION`.

This vocabulary and these invariants form the basis for the Java model implemented during Week 1.
