# Universal CI Interface — Doctor Profile

**Draft v0.1** · identifier `ucii/doctor/v0.1`

## 1. Purpose

The Universal CI Interface separates what a project can do from how it does it. Two questions are answered by the operations in the base specification: `describe` says what the project claims to implement, and self-verification says whether that claim holds.

`doctor` answers a third question:

> **What is the measurable health of this project, where is risk concentrated, and how is it changing?**

A conforming implementation exposes a `doctor` operation that measures the project and returns a machine-readable health report. This document defines that report: what a measurement is, what may be derived from one, what a policy is, what a finding is, and what a caller may rely on when reading one.

This profile declares no dimension, metric, or threshold to be mandatory. A health report names the dimensions it examined and reports the status of every one of them — including the ones it did not measure, and why (§17).

---

## 2. A Profile Capability

`doctor` is not a standard operation name. It is a capability this profile defines, and the profile is identified as `ucii/doctor/v0.1`.

A project claims this profile by exposing `doctor` and conforming to this document, and by identifying the profile, with its version, in the report it produces (§9).

The profile's **required core** — the operations a conforming implementation must expose and must mark as required in discovery (UCII §9) — is `describe` and `self-verify`: the two introspection operations that make the interface's own claims checkable (UCII §29). `doctor` is the capability this profile defines, and a claiming project advertises it as such.

Health measurement is not a standard operation for a reason worth stating. The operation in the base specification closest to it is `observe`, which retrieves operational information about a *running component or environment* — health, logs, metrics, traces, runtime diagnostics. `doctor` examines the *project*: its source, configuration, dependencies, history, and delivery configuration. A health report about a repository is not an observation of a running deployment, and neither operation stands in for the other.

`doctor` measures how a project is built, verified, and delivered. It therefore belongs at the project boundary, beside the operations it measures, and not inside the product the project builds.

---

## 3. Providers Observe, Operations Act

`doctor` measures by observing. Nothing it does modifies source, dependencies, configuration, or any external system, so a caller can diagnose a project without acting on it.

An implementation therefore distinguishes two kinds of provider:

* Where an existing operation already produces the measurement in machine-readable form, `doctor` MUST compose that result rather than re-derive it. A conformance measurement that re-implemented the interface's own self-verification would be a second implementation of the check, and the two would drift.
* Where no such operation exists, a provider invokes a tool to observe the project and never to change it.

A provider MUST NOT write into the working tree, and MUST NOT modify the project it measures. This is what keeps `doctor`'s declared side-effect class `read-only` (§8) while it still runs analyzers: **measuring is not acting**.

A provider that cannot produce its measurement MUST be reported as such (§17) rather than silently omitted, and MUST NOT stop the rest of the analysis.

---

## 4. Terminology

### Dimension

A named area of project health — `build`, `test`, `coverage`, `security`. A dimension is measured or it is not; either way it is reported (§17).

### Measurement

A fact established by observing the project: a value, the unit it is in, and the provider that produced it. A measurement is not a verdict.

### Metric

The named form of a measurement in the report, so a reader — or a later run — can identify the same quantity across reports.

### Derived metric

A deterministic transformation of one or more measurements, with an explicit and versioned formula (§7).

### Policy

A project's declaration of the conditions it considers acceptable (§12). A policy holds thresholds; it holds no measurements.

### Finding

A condition discovered from evidence: a violated threshold, or something a provider reported that a reader should see. A finding names the rule that produced it and carries the observations behind it.

### Provider

The instrument behind a measurement: a tool the implementation invoked, an analyzer it computed with, or an existing operation whose result it composed. A provider is identified by name and version (§9).

### Unit

An independently testable code unit — a function, a method, a type — used to attribute a measurement to a place in the source.

### Hotspot

Code where several measured risk factors intersect, selected by a named rule rather than a score (§14).

### Scope

What the analysis examined (§15, §16).

### Baseline, Trend

A previously recorded report, and the change between a measurement in it and the same measurement now (§13).

---

## 5. Evidence

A measurement is only as good as what it rests on. `doctor` MUST record the artifacts its measurement produced — the raw output of a tool it invoked, the profile a derived metric reads, and the report itself when a caller retains it — as evidence, referenceable by later operations, agents, and auditors (UCII §16).

Evidence MUST be kept out of the way of the tree being measured:

* a health check MUST NOT add a file to the working tree it is measuring;
* evidence lives in the project's evidence directory, which `clean` MUST NOT remove (UCII §16), because deleting the measurement a report refers to would strand that report;
* a retained report is what a later run compares against (§13), so retaining it is what makes trend possible at all.

A finding SHOULD reference the evidence it rests on, so a reader can reproduce the claim rather than trust it.

---

## 6. The Four-Layer Model

Every canonical claim in a health report belongs to exactly one of four layers, and a report MUST keep them distinguishable:

```text
MEASUREMENT      a fact, with the provider that established it
      │
      ▼
DERIVED METRIC   a deterministic transformation of facts
      │
      ▼
POLICY           a declared acceptable condition
      │
      ▼
FINDING          a violation or noteworthy condition derived from the two
```

* A **measurement** states what is. Whether it is acceptable is not part of it.
* A **derived metric** states what follows from measurements under a stated formula. It is still not a verdict.
* A **policy** states what the project considers acceptable. It is a declaration, not an observation.
* A **finding** is the only layer that carries a verdict, and it exists only where a rule connects the two above it.

A threshold is not a property of a metric, and a verdict is not a measurement. A report that blurs the layers cannot be read: a reader cannot tell whether a number is a fact, a requirement, or a complaint.

A finding produced by a declared threshold MUST be marked as a policy violation, so that enforcement (§19) can gate on violations and only on violations.

---

## 7. Derived Metrics

A derived metric MUST be deterministic, and its formula MUST be explicit, versioned, and inspectable. A metric that cannot be reproduced from its stated inputs and formula does not belong in a health report.

A derived metric MUST preserve the components it was calculated from, so a reader sees the facts behind a number rather than the number alone. The change-risk score is the worked example:

```text
CRAP(m) = CC(m)² × (1 − COV(m))³ + CC(m)

CC(m)    the unit's measured cyclomatic complexity
COV(m)   the unit's measured coverage, from 0.0 through 1.0
```

A score nobody can decompose into its complexity and coverage is a number a reader must take on trust, and a report that asks to be trusted has stopped being evidence.

A derived metric MUST NOT assert what is acceptable. That is policy (§6), and a formula that carried a threshold inside it would hide a decision where a reader would look for a measurement.

---

## 8. Effect Class

`doctor` declares the side-effect class `read-only` (UCII §17).

Measuring a project is observing it: the operation reads source, configuration, dependencies, history, and delivery configuration, and changes none of them. Running analyzers, invoking tools to observe the project, and recording evidence do not change the class — the class describes what the operation changes, not what it consumes, and the evidence it records is a separate artifact (UCII §16).

The declared class is descriptive only and is not authorization (UCII §17). It tells an autonomous caller what invoking `doctor` will not change; it does not tell the caller it may act on what the report says (§24).

---

## 9. Measurement Provenance

Every measurement MUST name the provider that produced it, with the provider's version, and the report MUST list the providers that measured it. The version identifies the instrument: a tool's version, or the toolchain a built-in analyzer was compiled with. Two numbers produced by materially different instruments are not the same quantity, and a report MUST NOT invite the comparison (§13).

A finding MUST name the rule that produced it and carry the observations that triggered it — the measurement, the requirement it violated, and the evidence behind it — so that a reader can reproduce any claim in the report.

The report MUST identify what it examined and under what contract:

```text
the report specification — the profile identifier and version
the project
the revision examined
the scope measured (§15)
```

A report that names a revision that no longer exists is a historical document; a report that names none is not evidence.

---

## 10. Units and Preserved Components

A unit-level derived score MUST NOT appear without its components (§7). A unit's entry carries its location, the measurements its score was derived from, and the score — with a component that nobody measured reported as *missing*, never as zero. Zero is a measured fact, and reporting an unmeasured component as zero would state something false about the code.

The same rule holds at the report level: a proportion nobody could take — a documented-symbol ratio in a tree that exports nothing — MUST be reported as missing rather than as perfect, because an empty denominator rendered as 100% reads as a documented interface rather than an absent one.

---

## 11. Providers and Agents

An implementation MUST let providers be supplied or configured, so the operation can be exercised against substitutes and against another project. A dimension whose provider is absent is not an error: it is a dimension this project does not measure, and it is reported as such (§17).

An agent MAY configure providers, invoke the operation, and prioritise what the report contains. It MUST NOT report a measurement it did not take, and MUST NOT present an inference as a measurement (§21).

---

## 12. Policy

A policy is a project's declaration of the conditions it considers acceptable. It is optional: a project MAY declare none.

A policy holds thresholds — floors and ceilings on measurements — and nothing else. It holds no measurements, and no measurement holds a verdict (§6). Thresholds MUST be evaluated in one place, so that the fact, the declaration, and the finding cannot blur.

A report MUST state the policy in force and where it came from, so a reader knows which document to change to change the outcome:

```json
{
  "doctor": {
    "policy": {
      "coverage": { "statements": { "minimum": 80 } },
      "complexity": { "cyclomatic": { "maximum": 15 } },
      "crap": { "maximum": 30 },
      "security": { "reachable": { "maximum": 0 }, "atSeverity": "error" },
      "tests": { "failed": { "maximum": 0 } }
    }
  }
}
```

* A policy that declares no threshold enforces nothing, and the report MUST say so rather than leave a reader to infer permission from a document full of empty objects.
* A threshold a project does not declare evaluates nothing. An absent threshold is not a permissive one; it is a threshold nobody set.
* A metric that was not measured cannot violate a threshold, and a policy MUST NOT be evaluated against a number nobody established.
* A policy that exists but cannot be read or parsed MUST NOT be replaced by defaults. Falling back silently would tell a caller its policy passed when the policy could not be read at all: that is a failure of the analysis (§18), reported as such.

A policy is not authorization, and a violated policy is not a refusal: it is a statement that the project does not meet a condition it declared for itself.

---

## 13. Baseline and Trends

A comparison is not a measurement. A report MAY compare its measurements against a previously recorded report, and every comparison MUST name its reference point: the baseline document and the revision that document examined. A trend whose reference point the report does not name MUST NOT be reported.

Only commensurable measurements may be compared. Two reports are commensurable only when they were produced under the same report specification and by the same set of providers at the same versions; a metric name under a different instrument is a materially different quantity, and comparing the two would report a change the project never made.

Where no baseline exists, the report MUST say that nothing was compared rather than report an empty comparison, which reads as "nothing changed" when it means "nothing was compared". A baseline that cannot be read is a warning, not a failure of the measurement: this run measured the project correctly, and simply has nothing to compare against.

---

## 14. Hotspot Analysis

A hotspot selects code where several measured factors intersect. The selection MUST be a named, versioned rule — a stated crossing of measurements, not an opaque risk score:

```text
rule:    hotspot.complexity-x-coverage
version: v1
selects: units whose measured complexity is at or above the declared
         complexity ceiling AND whose measured coverage is below the
         declared coverage floor
factors: cyclomatic_complexity, coverage, crap
```

Every hotspot MUST carry its rule, the rule's version, the factors that triggered the selection, and the location, so a caller can inspect the selection rather than trust it. Selection order MUST be deterministic, so two runs over an unchanged project produce the same report.

A hotspot rule reads policy (§12), because what counts as high complexity or low coverage is a project's declared condition. Where the policy declares no such threshold, no hotspot can be selected, and the report says so rather than inventing one.

---

## 15. Scope

A report MUST name the scope it measured. `scope=project` is the whole project: every file the analysis can read, at the revision the report names.

`scope=project` is the default and the only scope a conforming implementation MUST support.

---

## 16. Changed-Code Scope

A changed-code scope — measuring only what a branch, a commit range, or a pull request changed — is a scope a project MAY support. It requires a declared baseline revision: a scope of "changed" is meaningless without a named thing to have changed from.

A project that does not take a baseline revision MUST refuse the parameter rather than approximate it. Refusing is correct because the alternative is worse than a missing capability: a report labelled as changed-code analysis that in fact measured the whole project is wrong about its own scope, and every finding in it is mis-attributed. The parameter is not declared, so the request is refused (UCII §30) — it is not silently substituted.

---

## 17. Dimensions and Their Statuses

A report MUST carry every dimension it names, whether or not it could be measured. An unmeasured dimension is declared with the reason it was not measured, never omitted: a dimension that disappears from a report is indistinguishable from one that passed.

Each dimension carries exactly one of four statuses:

| Status | Meaning |
|---|---|
| `measured` | a provider reported on this dimension |
| `not-measured` | this project could measure the dimension but configures no provider for it |
| `unsupported` | no reproducible mechanism exists for this dimension here |
| `provider-failed` | a provider was configured and could not produce a measurement |

The four MUST NOT be conflated:

* `not-measured` and `unsupported` are statements about **this project's providers**, not about its health. Neither is evidence of a problem, and neither is evidence of health.
* `provider-failed` is a statement about the **instrument**, not about the code it was measuring.
* A project MUST NOT appear healthy because the dimensions that would show a problem were never measured. The statuses exist so that a reader can tell an absent measurement from a healthy one, and a report that reported both the same way would be misleading in exactly the case that matters.

The standard dimension names are:

```text
architecture    build           change          ci
complexity      configuration   coverage        crap
dependencies    documentation   duplication     license
maintainability performance     reliability     repository
security        static-analysis supply-chain    test
ucii
```

A dimension may be measured by any provider, or by none. `not-measured` is a legitimate state for a dimension a project does not measure, and a report that names it is more useful than one that stayed silent about it.

---

## 18. Execution and Findings

`doctor`'s own execution MUST be reported separately from the project's health:

```text
execution.status    whether doctor completed the analysis
execution.mode      the mode it ran in (§19)
execution.detail    why it did not, when it did not
```

The two are different facts, and a caller that conflates them cannot tell a broken tool from a broken project:

* A provider that cannot measure its dimension, or a policy that cannot be read, is a failure of the **analysis**, and MUST NOT be reported as a finding about the project.
* A report full of critical findings produced by a scan that ran correctly means the project is **unhealthy** — not that `doctor` malfunctioned.

A dimension that failed to measure is reported as `provider-failed` (§17) and does not stop the other dimensions from being measured and reported.

---

## 19. Modes and Exit Semantics

`doctor` runs in one of two modes, and the mode MUST be a declared, discoverable parameter with a documented default:

| Mode | Exits non-zero when |
|---|---|
| `diagnostic` (default) | the analysis could not complete |
| `enforce` | the analysis could not complete, or the policy in force is violated |

Diagnostic is the default because a diagnosis must not fail a developer's exploration: findings are for reading. Enforcement is one parameter away, and is what a build or release gate wants.

The mode MUST be visible in the report, so a caller can tell which semantics produced the exit code it received. A caller MUST NOT have to guess whether a zero exit means "no violations" or "violations were not gated on", and a diagnostic run MUST NOT silently enforce.

---

## 20. Minimum Conforming Report

A conforming report carries, at least:

```text
the report specification — profile identifier and version
the project, and the revision examined
the scope measured
the execution status, the mode, and the duration
the status of every dimension, with a reason where it measured nothing
the measurements, each named, valued, united, and attributed to a provider
the policy in force, and where it came from
the findings, each with its rule, severity, and evidence
the units a derived score was computed over, with their components
the hotspots, each with its rule and version
the trends, each naming its baseline — or an explicit statement that none was compared
the providers that measured the report
the evidence the report rests on
```

It is the canonical invocation result of UCII §12 — operation, status, outputs, evidence, warnings, metadata — with the report's own fields beside it. A caller MUST NOT have to parse prose to learn any of the above.

---

## 21. Interpretation Is Not Measurement

A canonical finding's severity MUST come from an explicit rule or from provider data. It MUST NOT be assigned by opinion, and it MUST NOT vary between two runs over the same project.

An agent's interpretation is separate from the report and MUST NOT be substituted for a measurement. An agent may prioritise findings, explain them, group them by what would have to change, and propose remediation; it MUST NOT silently promote or demote a severity, and MUST NOT present its judgement as the project's measured state.

`doctor` measures; policy evaluates; agents interpret and act. A report that let an opinion in would stop being evidence.

---

## 22. Standard and Namespaced Dimensions

The dimension names in §17 are the profile's own. A project MAY add dimensions and metrics of its own, and MUST namespace them so they cannot be mistaken for standard ones — a project-specific prefix, for example:

```text
acme.conventions        a project's own dimension
acme.conventions.undocumented
```

A project MUST NOT redefine the semantics of a standard dimension or metric name, and MUST NOT report a standard dimension as measured when what it measured was something else under the same name. A caller reading `security.reachable` is reading one quantity, whichever implementation produced the report; a project that means something different by it is not extending the profile but breaking it.

---

## 23. Distinguishable States

A reader MUST be able to tell these states apart from the report alone, without knowing the implementation:

```text
a healthy measurement            measured, and within policy
a policy violation               measured, and outside policy
a dimension not measured         no provider configured for it
a dimension unsupported          no mechanism exists for it
a measurement failure            the provider was configured and failed
a failure of doctor itself       the analysis did not complete
```

The sixth is the most important to keep separate (§18), and the second is the most important not to overstate: a policy violation is not a failed measurement. A report that renders any two of these six the same way has taken away the reader's ability to decide.

---

## 24. Acting on a Report

A health report is descriptive. It is not authorization to change the project it describes (UCII §17, §20), and invoking `doctor` grants nothing.

The intended use is a loop:

```text
measure
   │
   ▼
prioritise findings      interpretation, not measurement (§21)
   │
   ▼
remediate
   │
   ▼
re-measure, against the retained report (§13)
```

The loop closes only if the second measurement is comparable with the first: same report specification, same providers at the same versions (§13). A remediation that changes the instruments along with the code has changed the measurement and the project at once, and cannot show which produced the difference.
