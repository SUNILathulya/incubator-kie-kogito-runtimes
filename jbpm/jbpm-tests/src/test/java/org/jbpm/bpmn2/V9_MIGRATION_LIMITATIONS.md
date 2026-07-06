# v9 Code Generation Migration — Limitations & Findings

## Background

This document captures the limitations discovered while migrating jBPM test files
from the **v7 runtime-based API** to the **v9 code generation-based API**.

It is intended to help beginners understand:
- What v7 and v9 are
- Why some tests could be migrated and others could not
- The specific technical reasons for each limitation

---

## What is v7 vs v9?

### v7 (Legacy Runtime-Based API)
In v7, BPMN process files are **loaded at runtime** — meaning the test reads the `.bpmn2`
file from disk when the test runs, parses it, and creates a process engine on the fly.

```java
// v7 approach — loads BPMN file at test runtime
kruntime = createKogitoProcessRuntime("org/jbpm/bpmn2/sla/BPMN2-UserTaskWithSLA.bpmn2");
KogitoProcessInstance processInstance = kruntime.startProcess("UserTaskWithSLA");
```

Think of it like a restaurant that cooks food **only when you order it**.

### v9 (Code Generation-Based API)
In v9, BPMN process files are **compiled at build time** — a Maven plugin reads all `.bpmn2`
files and generates Java classes from them before the tests even run.

```java
// v9 approach — uses pre-generated Java class
Application app = ProcessTestHelper.newApplication();
UserTaskWithSLAProcess processDefinition = UserTaskWithSLAProcess.newProcess(app);
ProcessInstance<UserTaskWithSLAModel> processInstance =
    processDefinition.createInstance(processDefinition.createModel());
processInstance.start();
```

Think of it like a restaurant that **preps all ingredients in the morning** so food is
ready faster when ordered.

### Why Migrate from v7 to v9?
- v9 is faster (no runtime parsing of BPMN files)
- v9 is type-safe (generated Java classes catch errors at compile time)
- v9 is the future direction of the framework
- v7 APIs are deprecated and will eventually be removed

---

## Limitation 1: Tests Using `SLATimerMode=false` (External SLA Tracking)

### Affected Tests (SLAComplianceTest.java)
- `testSLAonProcessViolatedExternalTracking`
- `testSLAonUserTaskViolatedExternalTracking`
- `testSLAonProcessViolatedNoTracking`

### What Do These Tests Do?

These tests verify a special mode of SLA tracking where the **internal timer is disabled**
and SLA violations are triggered manually from outside the process (externally).

**Real-world analogy:**
> Normally, a kitchen timer automatically rings when food is ready (internal timer).
> These tests simulate a scenario where the timer is switched OFF, and instead a chef
> manually announces "time is up" (external signal).

### The v7 Code That Cannot Be Migrated

```java
// This line disables the internal SLA timer
kruntime.getKieRuntime().getEnvironment().set("SLATimerMode", "false");

// This line manually triggers an SLA violation from outside
kruntime.signalEvent("slaViolation", null, processInstance.getStringId());

// For node-level SLA, uses a node-specific signal
kruntime.signalEvent("slaViolation:" + userTaskNode.getStringId(), null,
    processInstance.getStringId());
```

### Why v7 Supports This
In v7, the `KieRuntime` exposes an `Environment` object — essentially a key-value store
where you can set configuration values that affect how the runtime behaves.

```java
// How it works internally in v7
// WorkflowProcessInstanceImpl.java line 1247-1255
protected boolean useTimerSLATracking() {
    String mode = (String) getKnowledgeRuntime().getEnvironment().get("SLATimerMode");
    if (mode == null) {
        return true; // default = use internal timer
    }
    return Boolean.parseBoolean(mode); // "false" = disable timer, use external
}
```

When `SLATimerMode=false`, the process does NOT start an internal timer for SLA.
Instead, it listens for a `"slaViolation"` signal that must be sent externally.

### Why v9 Cannot Support This

The v9 API (`ProcessTestHelper`, `Application`, `ProcessInstance`) is a **high-level API**
that intentionally hides the internal runtime details. There is no way to:

1. Access the `KieRuntime.getEnvironment()` from v9's `Application` or `ProcessInstance`
2. Set `SLATimerMode=false` before the process starts (the SLA timer is registered
   during `processInstance.start()`, before we can intercept it)
3. Send a raw `"slaViolation"` signal at the runtime level from v9 API

```java
// There is NO v9 equivalent of these v7 lines:
kruntime.getKieRuntime().getEnvironment().set("SLATimerMode", "false"); // ❌ not possible
kruntime.signalEvent("slaViolation", null, processId);                  // ❌ not possible
```

While `processInstance.send(SignalFactory.of("slaViolation", null))` exists in v9,
it sends a **business signal** through the BPMN signal flow — not a raw internal
runtime event. The `"slaViolation"` event in v7 is an internal infrastructure event,
not a BPMN signal.

### What Would Be Needed to Fix This

To migrate these tests, one of the following would be required:
1. Add a new configuration mechanism in v9 `ProcessConfig` to support `SLATimerMode`
2. Expose a way to set runtime environment variables through the v9 API
3. Redesign the tests to test the same business behavior through a different approach

---

## Limitation 2: Tests Using Business Rule Tasks with `.drl` Files

### Affected Tests (ErrorEventTest.java)
- `testErrorBoundaryEventOnBusinessRuleTask`
- `testMultiErrorBoundaryEventsOnBusinessRuleTask`

### What Do These Tests Do?

These tests verify that when a **Business Rule Task** (a task that executes Drools rules)
throws an error, a boundary event correctly catches and handles it.

**Real-world analogy:**
> A loan approval process uses a set of business rules (written in a `.drl` file)
> to decide if a loan should be approved. If the rules throw an error (e.g., bad credit
> score), an error boundary event catches it and routes the process to a rejection path.

### The v7 Code That Cannot Be Migrated

```java
// Loads BOTH a BPMN file AND a DRL (Drools Rule Language) file
kruntime = createKogitoProcessRuntime(
    "BPMN2-ErrorBoundaryEventOnBusinessRuleTask.bpmn2",
    "BPMN2-ErrorBoundaryEventOnBusinessRuleTask.drl");  // ← Drools rules file

kruntime.getProcessEventManager()
    .addEventListener(new RuleAwareProcessEventListener()); // ← Rule-aware listener

KogitoProcessInstance processInstance =
    kruntime.startProcess("BPMN2-ErrorBoundaryEventOnBusinessRuleTask");
```

### Why v7 Supports This
In v7, `createKogitoProcessRuntime()` can accept **both** `.bpmn2` and `.drl` files.
It loads them together into a `KieBase` (Drools knowledge base) that understands both
process definitions and business rules.

The `RuleAwareProcessEventListener` is a special listener that knows how to interact
with Drools rules during process execution.

### Why v9 Cannot Support This
The v9 code generator (`jbpm-tools-maven-plugin`) **only processes `.bpmn2` files**.
It does not know how to handle `.drl` files.

When we checked the generated sources directory:
```
target/generated-test-sources/jbpm/org/jbpm/bpmn2/error/
```

There was **no generated class** for `ErrorBoundaryEventOnBusinessRuleTask` or
`MultiErrorBoundaryEventsOnBusinessRuleTask`. The code generator silently skipped
these BPMN files because they contain Business Rule Tasks referencing `.drl` files
that it cannot process.

```
# Generated classes that DO exist (no DRL dependency):
✅ ErrorBoundaryEventOnTaskProcess.java
✅ EventSubprocessErrorProcess.java

# Generated classes that DO NOT exist (have DRL dependency):
❌ ErrorBoundaryEventOnBusinessRuleTaskProcess.java  (missing)
❌ MultiErrorBoundaryEventsOnBusinessRuleTaskProcess.java  (missing)
```

Without the generated class, there is no v9 equivalent to instantiate.

### What Would Be Needed to Fix This

The v9 code generator would need to be enhanced to support Business Rule Tasks
that reference `.drl` files — generating the appropriate Drools integration code
alongside the BPMN process classes.

---

## Limitation 3: Tests Using Unsupported BPMN Constructs

### Affected Test (ErrorEventTest.java)
- `testEventSubprocessErrorWithOutErrorCode`

### What Does This Test Do?

Tests that an event subprocess correctly catches an error **without an error code**
(catches any error regardless of type) and handles it.

### The v7 Code That Cannot Be Migrated

```java
// BPMN file is in a subfolder "subprocess/"
kruntime = createKogitoProcessRuntime(
    "subprocess/EventSubprocessErrorHandlingWithOutErrorCode.bpmn2");

KogitoProcessInstance processInstance =
    kruntime.startProcess("order-fulfillment-bpm.ccc");

assertProcessInstanceFinished(processInstance, kruntime);
assertNodeTriggered(processInstance.getStringId(),
    "start", "Script1", "starterror", "Script2", "end2", "eventsubprocess");
assertProcessVarValue(processInstance, "CapturedException",
    "java.lang.RuntimeException: XXX");
```

### Why v9 Cannot Support This

When we checked the generated sources:
```
# Generated classes that DO exist:
✅ EventSubprocessErrorProcess.java  (BPMN2-EventSubprocessError.bpmn2)
✅ EventSubprocessErrorHandlingWithErrorCodeProcess.java

# Generated classes that DO NOT exist:
❌ EventSubprocessErrorHandlingWithOutErrorCodeProcess.java  (missing)
```

The BPMN file `EventSubprocessErrorHandlingWithOutErrorCode.bpmn2` was **skipped
by the v9 code generator**. The likely reason is that this BPMN contains a construct
(error event subprocess without an error code) that the code generator does not
currently support.

Note: The process ID in this BPMN is `"order-fulfillment-bpm.ccc"` — the `.ccc`
extension in the process ID is unusual and may also be contributing to the generator
skipping this file.

### What Would Be Needed to Fix This

The v9 code generator would need to be enhanced to handle event subprocesses
that catch errors without a specific error code.

---

## Summary Table

| Test | File | Limitation Type | Root Cause |
|------|------|----------------|-----------|
| `testSLAonProcessViolatedExternalTracking` | `SLAComplianceTest` | Runtime Environment Config | `SLATimerMode=false` not accessible in v9 API |
| `testSLAonUserTaskViolatedExternalTracking` | `SLAComplianceTest` | Runtime Environment Config | `SLATimerMode=false` + node-level signal not accessible in v9 API |
| `testSLAonProcessViolatedNoTracking` | `SLAComplianceTest` | Runtime Environment Config | `SLATimerMode=false` not accessible in v9 API |
| `testErrorBoundaryEventOnBusinessRuleTask` | `ErrorEventTest` | Code Generator Limitation | Business Rule Task with `.drl` file not supported by v9 generator |
| `testMultiErrorBoundaryEventsOnBusinessRuleTask` | `ErrorEventTest` | Code Generator Limitation | Business Rule Task with `.drl` file not supported by v9 generator |
| `testEventSubprocessErrorWithOutErrorCode` | `ErrorEventTest` | Code Generator Limitation | BPMN construct not supported by v9 generator — no class generated |

---

## Quick Reference: What CAN Be Migrated to v9

A test **can** be migrated to v9 if ALL of the following are true:

| Check | Question |
|-------|----------|
| ✅ | Does the test use only `.bpmn2` files (no `.drl` files)? |
| ✅ | Does a generated `*Process.java` class exist in `target/generated-test-sources`? |
| ✅ | Does the test NOT use `kruntime.getKieRuntime().getEnvironment().set(...)`? |
| ✅ | Does the test NOT use `kruntime.getKieSession().insert(...)` (Drools facts)? |
| ✅ | Does the test NOT use `kruntime.signalEvent("slaViolation", ...)`? |
| ✅ | Does the test NOT use `RuleAwareProcessEventListener`? |

If any check fails → the test **cannot** be migrated to v9 at this time.

---

## How to Check if a Generated Class Exists

Before attempting migration, always verify the generated class exists:

```bash
# Replace "YourProcessName" with the process ID from the BPMN file
find jbpm/jbpm-tests/target/generated-test-sources -name "*YourProcessName*"

# Example:
find jbpm/jbpm-tests/target/generated-test-sources -name "*UserTaskWithSLA*"
# Output: UserTaskWithSLAProcess.java ✅ → can migrate

find jbpm/jbpm-tests/target/generated-test-sources -name "*BusinessRuleTask*"
# Output: (nothing) ❌ → cannot migrate
```
