---
layout: default
---
# Winthorpe Architecture

## Deterministically Bounded Financial Execution for AI

Winthorpe separates **AI reasoning** from **financial execution**.

AI is used to interpret objectives, reason about financial processes, and determine what should happen next. Execution, however, occurs only through predefined, tested capabilities inside a deterministic runtime.

This separation allows AI to retain process-level autonomy without receiving unrestricted authority over financial systems.

> **AI can compose functionality. AI cannot create functionality.**

At a high level:

```text
OBJECTIVE / INSTRUCTION
        ↓
   AI REASONING
        ↓
 STRUCTURED REQUEST
        ↓
 WINTHORPE RUNTIME
        ↓
ISOLATED EXECUTION
        ↓
 VERIFIED STATE
        ↓
   AI REASONING
        ↓
       ...
```

The cycle continues until the financial process completes, fails, or requires escalation.

# 1. Architecture Model

Winthorpe has three distinct architectural layers:

### AI Reasoning

The AI interprets objectives, evaluates available information and verified execution state, and determines what should happen next.

Depending on the process, the AI may follow a predefined sequence or dynamically compose a sequence of available capabilities.

### Execution Flows

A flow is an ordered composition of executable capabilities used to accomplish a financial process.

For example:

```text
Retrieve Invoice
      ↓
Validate Vendor
      ↓
Request Approval
      ↓
Execute Payment
      ↓
Confirm Settlement
```

Flows may be either:

**Forced** — the company defines the sequence in advance.

**AI-assembled** — the AI determines the appropriate sequence at runtime from the capabilities made available to it.

In either case, the executable capabilities themselves remain fixed.

### Execution Blocks

An execution block is a predefined, tested unit of executable functionality.

Examples include:

- Retrieve invoice
- Validate vendor
- Request human approval
- Initiate ACH payment
- Check payment status
- Issue refund
- Reverse transaction
- Update ledger

Each block defines its own:

- structured inputs;
- available operations;
- required permissions;
- validation rules;
- execution behavior;
- failure and retry behavior;
- structured outputs; and
- logging requirements.

Blocks exist before execution begins. AI cannot create a new block, write arbitrary execution logic, or grant itself a new financial capability during a process.

This creates the fundamental boundary in Winthorpe:

```text
AI REASONING
Flexible and probabilistic
        ↓
STRUCTURED EXECUTION BOUNDARY
        ↓
FINANCIAL EXECUTION
Predefined and deterministically constrained
```

# 2. Deterministic Execution Boundary

AI does not directly invoke arbitrary financial APIs through Winthorpe.

Instead, it requests execution through known structured interfaces.

For example:

```text
Action:   PAY_VENDOR
Vendor:   vendor_546
Invoice:  invoice_123
Amount:   34.87
Currency: USD
Rail:     ACH
```

Before anything executes, the Winthorpe runtime can deterministically evaluate the request.

Among other things, it can verify:

- whether `PAY_VENDOR` is an existing capability;
- whether required parameters are present;
- whether parameter types are valid;
- whether the requested combination is permitted;
- whether the capability is currently available;
- whether prerequisite state exists; and
- whether the required permissions can be provided.

A malformed, unavailable, or impermissible request does not partially execute. It fails validation and returns control to the AI.

The AI can then correct the request, select another available capability, or escalate.

## Architectural Invariants

Regardless of whether a flow is predefined or dynamically assembled, several properties remain invariant.

### Executable authority comes from blocks, not prompts

Natural-language instructions can influence AI reasoning. They cannot manufacture new executable authority.

An instruction such as:

```text
If necessary, transfer funds from another account.
```

does not enable a transfer unless an appropriate transfer capability exists and is available to the process.

### Permissions are scoped to execution

The AI does not receive persistent access to every financial system that might be required by a process.

When a block executes, Winthorpe makes available the capabilities and permissions required for that execution. Those permissions do not imply access to unrelated operations or future steps.

### Runtime state is authoritative

The model's context is not the system of record for execution.

Winthorpe maintains authoritative runtime state including:

- requested blocks;
- executed blocks;
- actual execution results;
- current process position;
- available permissions;
- failures and retries; and
- completed actions.

The AI may reason about that state. It cannot replace it with its own assertion of what occurred.

# 3. Sandboxed Sequential Isolation

Financial processes are not executed as one continuously privileged AI session.

Each execution step is isolated and constrained by the current verified state of the process.

Consider:

```text
1. Retrieve Invoice
2. Validate Invoice
3. Obtain Approval
4. Execute Payment
5. Confirm Settlement
```

The existence of `Execute Payment` later in the process does **not** mean the AI currently possesses the ability to execute a payment.

At the beginning:

```text
AVAILABLE

Retrieve Invoice


UNAVAILABLE

Validate Invoice
Obtain Approval
Execute Payment
Confirm Settlement
```

When `Retrieve Invoice` executes successfully, its actual output is persisted and verified.

That verified result changes the runtime state and allows the next permitted capability to become available.

Conceptually:

```text
[Retrieve Invoice]
        ↓
  verified invoice
        ↓
[Validate Invoice]
        ↓
 validated invoice
        ↓
 [Obtain Approval]
        ↓
  approval record
        ↓
 [Execute Payment]
        ↓
   transaction ID
        ↓
[Confirm Settlement]
```

The sequence is therefore not enforced merely through an instruction such as:

```text
Do not pay until approval has been obtained.
```

It is enforced through **capability availability**.

> **The model cannot unlock a capability by claiming that a prerequisite occurred. The runtime requires verified state.**

This produces two important architectural properties.

### Injection Resistance

Instructions encountered by the AI cannot create executable capabilities.

A malicious or injected instruction may attempt to alter the AI's reasoning, but execution remains constrained to predefined capabilities available in the current runtime state.

A step can act only through the operations and permissions exposed to that step.

### Misstep Resistance

Execution state advances only through verified results.

If the AI requests a capability that is unavailable, attempts to operate out of sequence, or supplies a request inconsistent with current state, execution does not proceed.

The AI can reason flexibly about the process.

The execution environment does not become flexible merely because the AI is.

# 4. Autonomy Within Deterministic Boundaries

Deterministic execution does not require every financial process to be hard-coded in advance.

Winthorpe separates **what the system is capable of executing** from **how those capabilities are composed**.

A company may require a fixed flow:

```text
A → B → C → D
```

For another process, it may give AI an objective and allow the AI to determine the appropriate sequence from available blocks.

For example:

```text
Resolve this failed vendor payment.
```

The AI might initially choose:

```text
Retrieve Payment
      ↓
Check Failure Reason
      ↓
Retrieve Vendor Payment Method
      ↓
Retry Payment
      ↓
Confirm Status
```

But execution does not require the AI to determine the complete flow in advance.

Suppose `Check Failure Reason` establishes that the vendor's payment details are invalid.

The AI might instead determine:

```text
Retrieve Payment
      ↓
Check Failure Reason
      ↓
Request Updated Vendor Details
      ↓
STOP / WAIT
```

The AI can therefore reason after each verified result and determine the next appropriate action.

What remains constant is the execution boundary:

```text
Dynamic flow creation       YES
Dynamic block creation      NO
Arbitrary execution code    NO
Direct financial API access NO
Self-granted permissions    NO
```

This provides **process-level flexibility with action-level constraints**.

Instructions and contextual information—including instructions supplied through MCP—may influence the objective, business rules, reasoning, block selection, and composition.

They do not override Winthorpe's execution boundary or create financial capabilities that do not already exist.

The division of responsibility is therefore:

> **AI determines what should happen next.  
> The runtime determines what can actually execute.**

# 5. Failure Semantics, Recovery, and Auditability

Financial execution requires stronger failure semantics than ordinary software actions.

A failed API call does not necessarily mean a financial operation failed.

An operation may be:

- definitely failed;
- successfully completed;
- pending;
- of unknown outcome;
- retryable; or
- permanently failed.

The distinction is critical.

If the outcome of a payment request is unknown, blindly repeating the request may create a duplicate payment.

Execution blocks can therefore encode the mechanics required to handle their financial operation correctly, including:

- idempotency;
- status verification;
- retry behavior;
- expected failure modes; and
- structured failure outputs.

The AI reasons about the **process-level response** to the result.

The block governs the **execution-level mechanics** of the financial operation.

For example:

```text
Execute Payment
      ↓
Outcome Unknown
      ↓
Check Payment Status
      ↓
 ┌───────────────┬────────────────┐
 ↓               ↓                ↓
SETTLED        FAILED           PENDING
 ↓               ↓                ↓
continue       AI reasons       wait/check
```

Recovery operations are subject to the same execution model.

A reversal, refund, or corrective transaction is an explicit capability with its own inputs, permissions, validation, execution behavior, and outputs.

AI does not receive unrestricted authority to "undo" arbitrary financial actions.

## Inherent Auditability

Auditability follows directly from the execution architecture.

Each financial process consists of discrete execution steps operating against authoritative runtime state. Each step produces an actual result that is persisted before execution proceeds.

The resulting record can establish, step by step:

```text
REQUESTED
    ↓
VALIDATED
    ↓
AUTHORIZED
    ↓
EXECUTED
    ↓
RESULT PERSISTED
    ↓
STATE VERIFIED
```

Because execution is isolated and stateful, the audit trail is an artifact of the execution itself rather than a reconstruction of the AI's reasoning after the fact.

This same property provides a deterministic point from which failures can be investigated, retried, escalated, or—where an explicit capability exists—reversed.

# Architectural Principle

Winthorpe is designed to preserve the part of AI that makes autonomous systems useful—reasoning, adaptation, and composition—while placing financial execution behind deterministic boundaries.

In practical terms:

> **AI reasons.**  
> **Winthorpe constrains what can execute.**  
> **Verified state determines what becomes possible next.**

The result is an execution architecture in which AI can dynamically determine how to complete a financial process without being given unrestricted authority over the financial systems that perform it.
