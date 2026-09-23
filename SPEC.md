# Universal CI Interface Specification

**Draft v0.1**

## 1. Purpose

The Universal CI Interface (UCII) defines a standard, implementation-independent interface for executing software delivery operations.

The specification separates **what a project can do** from **how the project does it**.

A project MAY use any execution engine that fits its needs: a task runner, build system, package manager, shell scripts, custom CLI, or another mechanism. The chosen execution engine MUST expose the operations and semantics defined by this specification.

The execution engine is therefore the authoritative entry point for software-delivery operations within the project.

```text
                    Actors

       ┌──────────────┬──────────────┐
       │              │              │
    Developer       Agent           CI
       │              │              │
       ├──────────────┼──────────────┤
       │              │              │
   Git Hook          IDE         Scheduler
       │              │              │
       └──────────────┴──────────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │ Universal CI         │
          │ Interface            │
          └───────────┬───────────┘
                      │
                      ▼
          ┌───────────────────────┐
          │ Project Execution     │
          │ Engine                │
          └───────────┬───────────┘
                      │
             ┌────────┼────────┐
             ▼        ▼        ▼
           Build    Test     Deploy
           tools    tools     tools
```

The fundamental principle is:

> **Software-delivery operations belong to the project. Orchestration belongs to the caller. Implementation belongs to the project.**

---

# 2. Goals

UCII has six primary goals.

**Portability.** A caller that understands UCII SHOULD be able to operate an unfamiliar project without knowing its underlying toolchain.

**Actor independence.** The same operation SHOULD be executable by a person, AI agent, CI system, Git hook, IDE, scheduler, or other automation.

**Implementation independence.** UCII MUST NOT require a particular task runner, CI vendor, programming language, build system, deployment platform, or infrastructure technology.

**Local/CI parity.** Operations executed locally and operations executed in CI SHOULD use the same project interface.

**Discoverability.** A caller MUST be able to determine which capabilities a project exposes and how to invoke them.

**Composability.** Operations MUST be independently invocable so external systems can compose them into workflows.

---

# 3. Non-goals

UCII is NOT:

* a CI pipeline language;
* a workflow orchestration system;
* a CI server;
* a build system;
* a deployment system;
* an agent framework;
* a replacement for existing project tooling.

UCII standardizes the **interface presented by those systems**.

It deliberately does not prescribe:

```text
build → test → scan → package → deploy
```

That sequencing belongs to the caller.

---

# 4. Terminology

### Project

A software project implementing this specification.

### Actor

Anything requesting an operation.

Examples include:

```text
human
AI agent
CI system
Git hook
IDE
scheduler
release automation
another operation
```

### Execution Engine

The project-selected mechanism responsible for exposing and executing UCII operations.

Examples could include a task runner, build tool, package manager, project CLI, or custom execution framework.

UCII does not prescribe which one.

### Operation

A named unit of software-delivery functionality such as:

```text
build
test
verify
package
deploy
```

### Capability

An operation actually supported by a particular project.

### Invocation

A request by an actor to execute an operation.

### Artifact

An immutable or identifiable output produced by an operation.

### Evidence

Machine-consumable information establishing what occurred during an operation.

### Orchestrator

An actor that invokes multiple operations according to some workflow.

---

# 5. Conformance

A conforming project MUST designate exactly one logical **project execution interface**.

The underlying implementation MAY consist of multiple tools, but callers MUST NOT be required to understand those implementation details.

For example:

```text
                     UCII
                      │
                      ▼
              Execution Engine
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       compiler    container    Terraform
                     tool
```

The execution engine MAY delegate freely.

It remains responsible for presenting the standardized interface.

---

# 6. Execution Engine Rule

This is a core requirement of UCII.

> **Software-delivery automation SHOULD invoke the project execution engine rather than directly invoking its underlying implementation tools.**

For example, suppose a project's `test` implementation ultimately runs a language-specific test framework.

A developer SHOULD invoke:

```text
<execution-engine> test
```

CI SHOULD invoke:

```text
<execution-engine> test
```

An agent SHOULD invoke:

```text
<execution-engine> test
```

rather than having each actor independently encode knowledge of the underlying test framework.

This creates:

```text
              test
               │
        ┌──────┴───────┐
        │ Project      │
        │ Interface    │
        └──────┬───────┘
               │
       implementation
               │
               ▼
        underlying tool
```

instead of:

```text
Developer ─────► tool A
CI ────────────► tool A + flags
Agent ─────────► script B
Git hook ──────► tool A + different flags
```

The former is conformant with the architectural intent of UCII; the latter creates execution-path drift.

---

# 7. Standard Operation Vocabulary

UCII v0.1 defines the following standard operations:

```text
describe
plan

build
test
verify
scan

package
publish
promote

provision
configure
deploy
rollback

status
observe

clean
```

Projects are NOT required to implement every operation.

Projects MUST advertise which operations they support.

Additional project-specific operations MAY exist.

---

# 8. Operation Semantics

## `describe`

Returns the capabilities and interface metadata of the project.

`describe` MUST NOT mutate the project or external systems.

It SHOULD be the primary mechanism used by agents and generic automation to understand an unfamiliar project.

---

## `plan`

Calculates the expected effects of an operation without intentionally applying those effects.

Typical uses include:

```text
infrastructure changes
deployment changes
configuration changes
release changes
```

A project MAY only support `plan` for operations where planning has meaningful semantics.

---

## `build`

Transforms project source into its build representation.

Examples include compilation, transpilation, code generation, bundling, or other project-specific build processes.

---

## `test`

Executes behavioral tests against the project or its components.

Test suites MAY be selectable through parameters.

Examples include:

```text
unit
integration
functional
acceptance
end-to-end
```

---

## `verify`

Determines whether the project satisfies project-defined correctness requirements that are not more appropriately represented as `test`.

Examples include:

```text
linting
format validation
type checking
schema validation
configuration validation
static analysis
```

`verify` SHOULD generally be non-destructive.

---

## `scan`

Performs security, supply-chain, compliance, or vulnerability analysis.

Examples include:

```text
dependency scanning
SAST
secret scanning
container scanning
license analysis
SBOM analysis
```

---

## `package`

Creates a distributable artifact from project outputs.

Examples include:

```text
binary archives
containers
language packages
deployment bundles
installers
```

`package` does not imply publication.

---

## `publish`

Makes an artifact available through an artifact distribution system.

Examples include:

```text
artifact repository
container registry
package registry
release repository
object storage
```

---

## `promote`

Advances an existing artifact to another lifecycle stage.

Promotion SHOULD preserve artifact identity.

In particular:

> **Promotion SHOULD NOT rebuild source.**

This supports the principle of building once and promoting the resulting artifact.

---

## `provision`

Creates resources required to operate the project.

Examples include infrastructure, databases, queues, namespaces, cloud resources, or environments.

---

## `configure`

Applies configuration to a component or environment.

Configuration SHOULD be distinguishable from provisioning where practical.

---

## `deploy`

Makes a specified project version or artifact active within a target environment.

The implementation MAY use any deployment technology.

---

## `rollback`

Returns a deployed system toward a previously identified state.

Rollback semantics MUST be defined by the project when the capability is advertised.

---

## `status`

Returns current state relevant to the project or a specified environment.

`status` MUST NOT intentionally mutate that state.

---

## `observe`

Retrieves operational information about a running component or environment.

Examples include:

```text
health
logs
metrics
traces
deployment information
runtime diagnostics
```

`observe` SHOULD be read-only.

---

## `clean`

Removes disposable outputs generated by project operations.

`clean` MUST NOT implicitly mean destruction of persistent or production infrastructure.

Destructive infrastructure operations require explicit project-specific capabilities.

---

# 9. Discovery

A conforming project MUST provide a machine-readable description of its UCII implementation.

Conceptually:

```text
<execution-engine> describe
```

The result SHOULD contain at least:

```json
{
  "spec": "ucii/v0.1",
  "project": "payments-service",
  "operations": {
    "build": {},
    "test": {},
    "verify": {},
    "package": {},
    "deploy": {}
  }
}
```

The actual execution-engine syntax is implementation-specific.

The semantic operation is not.

---

# 10. Operation Description

Discovery SHOULD provide enough information for an unfamiliar actor to invoke an operation correctly.

For example:

```json
{
  "deploy": {
    "description": "Deploy an artifact into an environment.",
    "parameters": {
      "environment": {
        "type": "string",
        "required": true
      },
      "artifact": {
        "type": "string",
        "required": true
      },
      "strategy": {
        "type": "string",
        "required": false,
        "values": [
          "rolling",
          "canary"
        ]
      }
    },
    "effects": {
      "local": false,
      "external": true
    }
  }
}
```

This enables an agent to reason about the interface without requiring bespoke knowledge of the project's execution engine.

---

# 11. Inputs

Operations MAY accept inputs.

UCII standardizes their **semantic names**, not necessarily their command-line representation.

Common inputs SHOULD include:

```text
artifact
environment
target
version
suite
strategy
format
source
```

For example, these two implementations could both be conformant:

```text
engine deploy --environment staging
```

and:

```text
engine deploy ENVIRONMENT=staging
```

provided the project advertises how its UCII `environment` input maps onto its execution engine.

This distinction allows UCII to remain independent of execution-engine syntax.

---

# 12. Invocation Result

Every operation has a logical result containing:

```text
operation
status
outputs
evidence
warnings
metadata
```

A canonical representation is:

```json
{
  "spec": "ucii/v0.1",
  "operation": "build",
  "status": "succeeded",
  "outputs": {},
  "evidence": [],
  "warnings": [],
  "metadata": {}
}
```

Execution engines MAY render additional human-readable output.

---

# 13. Status

UCII defines the following baseline statuses:

```text
succeeded
failed
blocked
skipped
```

Execution engines MAY internally support richer state models.

Those states SHOULD map onto the UCII status model when communicating with generic callers.

---

# 14. Process Semantics

Where the execution engine operates through a process interface:

```text
0      success
non-0  operation did not succeed
```

MUST remain sufficient for basic callers.

Machine-readable results MAY communicate richer information.

This ensures basic compatibility with shells, hooks, CI runners, and other conventional automation.

---

# 15. Outputs

Operations MAY produce named outputs.

For example:

```json
{
  "operation": "package",
  "status": "succeeded",
  "outputs": {
    "artifact": {
      "type": "container",
      "identifier": "registry.example.com/payments@sha256:abc123"
    }
  }
}
```

Outputs SHOULD be machine consumable.

An orchestrator SHOULD be able to pass an output from one operation into another without understanding the underlying implementation.

Conceptually:

```text
build
  │
  ▼
package
  │
  │ artifact
  ▼
scan
  │
  ▼
publish
  │
  ▼
deploy
```

---

# 16. Evidence

Operations SHOULD expose evidence when useful.

Evidence describes facts established during execution.

Examples include:

```text
test results
coverage
SBOM
vulnerability report
artifact signature
provenance
deployment record
policy evaluation
logs
```

Example:

```json
{
  "operation": "test",
  "status": "succeeded",
  "evidence": [
    {
      "type": "test-results",
      "location": ".ci/evidence/tests.json"
    }
  ]
}
```

Evidence SHOULD be referenceable by subsequent operations, agents, policy systems, and auditors.

---

# 17. Side Effects

Discovery SHOULD identify an operation's expected side-effect class.

At minimum:

```text
read-only
local
external
destructive
```

For example:

```json
{
  "status": {
    "effects": "read-only"
  },
  "build": {
    "effects": "local"
  },
  "deploy": {
    "effects": "external"
  },
  "destroy-preview-environment": {
    "effects": "destructive"
  }
}
```

This information is particularly important for autonomous callers.

Discovery metadata is descriptive; it does **not** constitute authorization.

---

# 18. Idempotency

Operations SHOULD declare their expected idempotency characteristics.

For example:

```text
status       idempotent
verify       idempotent
deploy       implementation-defined
publish      implementation-defined
```

An actor MUST NOT assume that retrying an arbitrary operation is safe.

---

# 19. Secrets and Credentials

UCII MUST NOT define a universal secret-storage mechanism.

Credentials remain the responsibility of the execution environment.

An operation SHOULD consume credentials through the project's established credential mechanisms rather than requiring callers to embed credentials in operation parameters.

The same operation might therefore obtain credentials differently when invoked from:

```text
developer workstation
CI
agent sandbox
production automation
```

without changing the UCII operation itself.

---

# 20. Policy and Authorization

Capability discovery does not imply permission.

For example, an agent may discover:

```text
deploy(environment=production)
```

while lacking authorization to execute it.

Authorization MAY be enforced by:

```text
execution engine
underlying platform
credential system
policy engine
CI environment
human approval mechanism
```

UCII SHOULD preserve these existing organizational controls rather than circumvent them.

---

# 21. Orchestration

UCII explicitly separates **operation definition** from **operation orchestration**.

The project provides:

```text
build
test
verify
scan
package
publish
deploy
```

A Git hook might compose:

```text
verify → test
```

CI might compose:

```text
verify → test → build → package → scan
```

Release automation might compose:

```text
publish → promote → deploy
```

An agent might dynamically determine:

```text
describe
    ↓
verify
    ↓
test
    ↓
inspect evidence
    ↓
build
    ↓
package
```

UCII does not prescribe any of these workflows.

---

# 22. Actor Independence

A conforming operation MUST NOT depend on the identity of its orchestrator unless that distinction is intrinsically required for authorization or policy.

Conceptually:

```text
                         ┌── Human
                         │
                         ├── Agent
                         │
                         ├── Git Hook
                         │
       operation ◄───────┼── CI
                         │
                         ├── IDE
                         │
                         ├── Scheduler
                         │
                         └── Release System
```

The project owns the operation.

The actor merely invokes it.

---

# 23. Execution-Engine Adaptation

A major intended use case for UCII is automatic adaptation of existing projects.

An agent encountering an existing repository SHOULD be able to:

```text
Inspect project
      │
      ▼
Identify existing execution conventions
      │
      ▼
Select existing/preferred execution engine
      │
      ▼
Map UCII operations onto existing tooling
      │
      ▼
Add missing adapters
      │
      ▼
Expose UCII discovery
      │
      ▼
Validate conformance
```

For example, one repository might expose UCII through its existing task runner.

Another might expose it through its existing build system.

Another might use a custom project CLI.

**UCII does not require replacing any of them.**

Instead, adaptation brings the existing execution engine into conformance with UCII.

---

# 24. No Parallel Automation

Once an operation is represented by the project execution engine, external automation SHOULD invoke that operation rather than duplicate its implementation.

For example, CI SHOULD NOT independently contain the project's actual build procedure:

```text
CI:
  install compiler
  generate sources
  compile
  copy resources
  create artifact
```

when the project exposes:

```text
build
```

CI SHOULD instead invoke the project's `build` operation.

This makes CI primarily an **orchestrator**:

```text
CI
 │
 ├─ checkout
 │
 ├─ invoke verify
 │
 ├─ invoke test
 │
 ├─ invoke build
 │
 ├─ invoke package
 │
 └─ retain results
```

The project execution engine owns the implementation.

---

# 25. Conformance Levels

It may be useful to define three levels of conformance.

**Level 1 — Invocation.** The project exposes standardized operation names with predictable success/failure behavior.

**Level 2 — Discovery.** Level 1 plus machine-readable capabilities, parameters, effects, and operation metadata.

**Level 3 — Structured Execution.** Level 2 plus standardized outputs, evidence, statuses, and machine-readable results.

This allows adoption to start extremely cheaply.

A legacy project can become Level 1 compliant by mapping a handful of operations onto what it already does, while an agent-native project can expose the complete Level 3 contract.

---

# 26. Minimal Conforming Example

A small project might expose only:

```text
describe
build
test
verify
clean
```

A service might expose:

```text
describe
build
test
verify
scan
package
publish
deploy
rollback
status
observe
```

An infrastructure repository might expose:

```text
describe
plan
verify
provision
configure
status
```

All three can conform.

UCII describes capabilities; it does not force irrelevant lifecycle stages onto a project.

---

# 27. Architectural Invariant

The most important invariant in the specification is:

```text
                     WHAT
                standardized
                     │
                     ▼
Actor ───────► UCII Operation ───────► Execution Engine
                                        │
                                        ▼
                                      HOW
                                  project-owned
```

Neither side should leak unnecessarily across that boundary.

An agent does not need to know that `build` uses a particular build system.

CI does not need to know that `deploy` ultimately calls a particular platform API.

A developer does not need a separate "local" implementation of `test`.

The execution engine owns those details.

This yields the central UCII contract:

> **Any actor can invoke a standard operation. The project chooses how that operation is implemented. The execution engine is the single interface to that implementation. Orchestration remains outside the interface.**

That makes UCII less a universal *CI system* and more a **universal interface between software projects and anything that wants to build, verify, package, release, deploy, or operate them**. CI is then simply one consumer of that interface.
