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

## Limitation 4: Tests That Validate Runtime BPMN Parsing Errors

### Affected Tests (FlowTest.java)
- `testExclusiveSplitWithNoConditions`
- `testMultipleInOutgoingSequenceFlowsDisable`

### What Do These Tests Do?

These tests verify that the engine **rejects invalid BPMN at load time** — i.e., when
you try to load a structurally invalid process, the engine throws a descriptive exception.

**Real-world analogy:**
> Like a compiler that rejects code with syntax errors before running it.
> These tests check that the "compiler" produces the right error message.

#### `testExclusiveSplitWithNoConditions`
```java
// Loads a BPMN with an XOR gateway that has NO conditions on its outgoing paths
createKogitoProcessRuntime(
    "org/jbpm/bpmn2/flow/BPMN2-ExclusiveGatewayWithNoConditionsDefined.bpmn2");
fail("Should fail as XOR gateway does not have conditions defined");
// expects: "does not have a constraint for Connection"
```
The BPMN has an exclusive gateway (XOR split) but the outgoing sequence flows have
no conditions defined. In v7, this is caught when the process is loaded and validated
at runtime.

#### `testMultipleInOutgoingSequenceFlowsDisable`
```java
// Loads a BPMN where a Script Task has 2 outgoing + 2 incoming connections
// (invalid unless jbpm.enable.multi.con=true is set)
createKogitoProcessRuntime("BPMN2-MultipleInOutgoingSequenceFlows.bpmn2");
fail("Should fail as multiple outgoing and incoming connections are disabled by default");
// expects: "This type of node [ScriptTask_1, Script Task] cannot have more than
//           one outgoing connection!"
```
The BPMN has a `ScriptTask` with two outgoing connections and two incoming connections.
By default (`jbpm.enable.multi.con` not set), this is invalid. In v7, the exception
is thrown during BPMN parsing when connections are wired.

### Why v7 Supports This

In v7, `createKogitoProcessRuntime()` parses and validates the BPMN file **at test
runtime**. The validation chain for `testMultipleInOutgoingSequenceFlowsDisable` is:

1. **[`RuleFlowProcess` constructor](jbpm/jbpm-flow/src/main/java/org/jbpm/ruleflow/core/RuleFlowProcess.java)** — reads `System.getProperty("jbpm.enable.multi.con")` and stores it in process metadata
2. **[`ActionNode.validateAddOutgoingConnection()`](jbpm/jbpm-flow/src/main/java/org/jbpm/workflow/core/node/ActionNode.java)** — during BPMN XML parsing, as connections are wired up, checks `WORKFLOW_PARAM_MULTIPLE_CONNECTIONS.get(getProcess())`. No property set → `null` → `false` → throws `IllegalArgumentException`
3. `createKogitoProcessRuntime()` propagates that exception up to the test

For `testExclusiveSplitWithNoConditions`, a similar validation is triggered when the
XOR gateway's connections are verified and no constraint/condition is found.

### Why v9 Cannot Support This

**In v9, this entire path does not exist at test runtime.** The code generator runs
at **build time** (Maven compile phase). By the time the test runs, the BPMN has
already been processed and the Java class is compiled.

There is no v9 API that re-parses a `.bpmn2` file at test runtime and throws a
validation exception. `ProcessTestHelper.newApplication()` boots a pre-compiled
application — it does not load BPMN files from disk.

Additionally, `BPMN2-MultipleInOutgoingSequenceFlows.bpmn2` lives at:
```
src/test/resources/BPMN2-MultipleInOutgoingSequenceFlows.bpmn2
```
This is a **flat path outside the package directory** that the code generator scans
(`src/test/bpmn/org/jbpm/bpmn2/`). No generated class exists for it.

Even if the file were moved into the generator's scan path, the **build itself would
fail** — the same `validateAddOutgoingConnection()` logic runs during codegen, so
the Maven build would error out. The test cannot be expressed as a build-time failure
inside a JUnit test method.

### What Would Be Needed to Fix This

These tests verify behaviour that is now a **build-time concern**, not a runtime
concern. The conceptual equivalent in v9 would be:
1. A build-time test that asserts the Maven code generator rejects the invalid BPMN
2. Or the tests could be removed/disabled, since the validation still happens — just
   earlier in the pipeline (at build time instead of test time)

---

## Limitation 5: Tests Requiring `jbpm.enable.multi.con=true` System Property

### Affected Tests (FlowTest.java)
- `testMultipleInOutgoingSequenceFlows`
- `testMultipleIncomingFlowToEndNode`

### What Do These Tests Do?

These tests verify that when the `jbpm.enable.multi.con=true` system property is set,
the engine **allows** a node to have multiple incoming or outgoing connections — a
non-standard BPMN construct.

**Real-world analogy:**
> Standard road rules say each intersection has one exit lane. These tests verify
> a special "multi-lane mode" where one intersection can have multiple exit lanes.

```java
// testMultipleInOutgoingSequenceFlows
System.setProperty("jbpm.enable.multi.con", "true");
kruntime = createKogitoProcessRuntime("BPMN2-MultipleInOutgoingSequenceFlows.bpmn2");
// ... starts the process and verifies it runs with multi-connection enabled

// testMultipleIncomingFlowToEndNode
System.setProperty("jbpm.enable.multi.con", "true");
kruntime = createKogitoProcessRuntime("BPMN2-MultipleFlowEndNode.bpmn2");
KogitoProcessInstance processInstance = kruntime.startProcess("MultipleFlowEndNode");
assertProcessInstanceCompleted(processInstance);
```

### Why v9 Cannot Support This

**Two compounding reasons:**

#### Reason 1 — BPMN files are outside the code generator's scan path

Both BPMN files live under `src/test/resources/` (flat directory, no package):
```
src/test/resources/BPMN2-MultipleInOutgoingSequenceFlows.bpmn2
src/test/resources/BPMN2-MultipleFlowEndNode.bpmn2
```

The v9 code generator scans `src/test/bpmn/org/jbpm/bpmn2/`. These files are
**not in that directory**, so no generated class exists for either.

#### Reason 2 — The `jbpm.enable.multi.con` flag is a build-time concern in v9

In v7, `jbpm.enable.multi.con` is read at runtime by
[`RuleFlowProcess`](jbpm/jbpm-flow/src/main/java/org/jbpm/ruleflow/core/RuleFlowProcess.java):
```java
// RuleFlowProcess constructor — reads system property at BPMN parse time
setMetaData("jbpm.enable.multi.con", System.getProperty("jbpm.enable.multi.con"));
```

In v9, BPMN parsing happens during the **Maven build**, not during the test. Setting
`System.setProperty("jbpm.enable.multi.con", "true")` inside a test method has no
effect because the code generator has already run (with or without the flag) hours
before the test executes.

Even if the BPMN files were moved into the scan path, the codegen would need the
flag set in the Maven build configuration — not in a JUnit test method.

### What Would Be Needed to Fix This

1. Move the BPMN files into the generator's scan path (`src/test/bpmn/org/jbpm/bpmn2/flow/`)
2. Configure `jbpm.enable.multi.con=true` as a Maven system property during the
   code generation phase (in `pom.xml`)
3. Rewrite the test using the generated class and v9 API

---

## Limitation 6: Tests Relying on Swimlane Actor Propagation via Work Item Parameters

### Affected Test (FlowTest.java)
- `testLane`

### What Does This Test Do?

Tests that **swimlane actor propagation** works correctly: when the first human task
in a swimlane is completed by "mary" (overriding the default "john"), the second
human task in the same swimlane automatically inherits "mary" as the actor.

**Real-world analogy:**
> In a bank, if Mary handles the first step of a loan application, all subsequent
> steps in that "lane" are automatically assigned to Mary as well — unless explicitly
> reassigned.

### The v7 Code That Cannot Be Migrated

```java
kruntime = createKogitoProcessRuntime("org/jbpm/bpmn2/flow/BPMN2-Lane.bpmn2");
TestUserTaskWorkItemHandler workItemHandler = new TestUserTaskWorkItemHandler();
kruntime.getKogitoWorkItemManager().registerWorkItemHandler("Human Task", workItemHandler);
KogitoProcessInstance processInstance = kruntime.startProcess("Lane");

KogitoWorkItem workItem = workItemHandler.getWorkItem();
assertThat(workItem.getParameter("ActorId")).isEqualTo("john");

Map<String, Object> results = new HashMap<>();
// ↓ This mutates the work item's "parameters" map directly before completion
((KogitoWorkItemImpl) workItem).setParameter("ActorId", "mary");
kruntime.getKogitoWorkItemManager().completeWorkItem(workItem.getStringId(), results);

// The second task should now have "mary" as the SwimlaneActorId
KogitoWorkItem workItem2 = workItemHandler.getWorkItem();
assertThat(workItem2.getParameter("SwimlaneActorId")).isEqualTo("mary");  // ← key assertion
kruntime.getKogitoWorkItemManager().completeWorkItem(workItem2.getStringId(), null);
assertProcessInstanceFinished(processInstance, kruntime);
```

The critical line is:
```java
((KogitoWorkItemImpl) workItem).setParameter("ActorId", "mary");
```
This writes directly into the work item's **parameters** map before calling
`completeWorkItem`. The swimlane propagation reads from that parameters map.

### Why v7 Supports This

In v7's swimlane propagation, [`HumanTaskNodeInstance.triggerCompleted()`](jbpm/jbpm-flow/src/main/java/org/jbpm/workflow/instance/node/HumanTaskNodeInstance.java)
reads the `ActorId` from the work item's **parameters** map:

```java
// HumanTaskNodeInstance.java (simplified)
String actorId = (String) workItem.getParameter(ACTOR_ID);
if (actorId != null) {
    // Update the swimlane context so next task inherits this actor
    swimlaneContextInstance.setActorId(swimlaneName, actorId);
}
```

By mutating the work item parameters **before** calling `completeWorkItem`, v7 ensures
that `triggerCompleted()` sees the new actor ("mary") and updates the swimlane.

### Why v9 Cannot Support This

In v9, `processInstance.completeWorkItem(id, resultsMap)` writes the completion data
into the **results** map, not the parameters map. The `triggerCompleted()` internal
method still reads `workItem.getParameter(ACTOR_ID)` — but since `completeWorkItem`
never updates parameters, it always sees the original value ("john") and the swimlane
is never updated.

There is no clean v9 public API to:
1. Mutate work item parameters before completion (requires casting to `KogitoWorkItemImpl` — an internal class)
2. Update the swimlane actor directly

```java
// v9 approach — results map only, parameters not updated:
processInstance.completeWorkItem(workItemId, Map.of("ActorId", "mary")); // ❌ won't propagate
// triggerCompleted() reads workItem.getParameter("ActorId") — still "john"
```

### What Would Be Needed to Fix This

Either:
1. Modify [`HumanTaskNodeInstance.triggerCompleted()`](jbpm/jbpm-flow/src/main/java/org/jbpm/workflow/instance/node/HumanTaskNodeInstance.java)
   to read `ActorId` from the **results** map (in addition to or instead of parameters)
2. Add a v9 API method to update work item parameters before completion
3. Add a v9 API method to set the swimlane actor explicitly

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
| `testExclusiveSplitWithNoConditions` | `FlowTest` | Runtime Validation Test | Validates BPMN parse-time exception — v9 has no runtime parsing |
| `testMultipleInOutgoingSequenceFlowsDisable` | `FlowTest` | Runtime Validation Test | Validates build-time exception — BPMN file also outside scan path |
| `testMultipleInOutgoingSequenceFlows` | `FlowTest` | Multi-Connection Flag + File Path | `jbpm.enable.multi.con` is build-time in v9; BPMN file outside scan path |
| `testMultipleIncomingFlowToEndNode` | `FlowTest` | Multi-Connection Flag + File Path | `jbpm.enable.multi.con` is build-time in v9; BPMN file outside scan path |
| `testLane` | `FlowTest` | Swimlane Actor Propagation | `completeWorkItem` puts data in results map; engine reads from parameters map |

---

## Quick Reference: What CAN Be Migrated to v9

A test **can** be migrated to v9 if ALL of the following are true:

| Check | Question |
|-------|----------|
| ✅ | Does the test use only `.bpmn2` files (no `.drl` files)? |
| ✅ | Does a generated `*Process.java` class exist in `target/generated-test-sources`? |
| ✅ | Does the BPMN file live under `src/test/bpmn/org/jbpm/bpmn2/`? |
| ✅ | Does the test NOT use `kruntime.getKieRuntime().getEnvironment().set(...)`? |
| ✅ | Does the test NOT use `kruntime.getKieSession().insert(...)` (Drools facts)? |
| ✅ | Does the test NOT use `kruntime.signalEvent("slaViolation", ...)`? |
| ✅ | Does the test NOT use `RuleAwareProcessEventListener`? |
| ✅ | Does the test NOT rely on testing a BPMN load/parse exception? |
| ✅ | Does the test NOT set `System.setProperty("jbpm.enable.multi.con", "true")`? |
| ✅ | Does the test NOT mutate work item parameters via `KogitoWorkItemImpl.setParameter()`? |

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
