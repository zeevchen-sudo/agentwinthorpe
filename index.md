---
layout: default
---
# Winthorpe Architecture Reference

This document describes the execution model and architecture of Winthorpe. It is intended as a reference for analyzing how Winthorpe works, not as a product or market overview.

## Core Model

Winthorpe allows an AI system to execute financial processes by composing **pre-built, pre-tested execution blocks**.

There are three distinct layers:

1. **Blocks** — fixed executable capabilities.
2. **Flows** — sequences of blocks used to accomplish a process.
3. **AI reasoning** — determines what should happen and, where permitted, how available blocks should be composed.

The most important architectural constraint is:

> **AI can compose functionality. AI cannot create functionality.**

An AI may dynamically construct a new flow from existing Winthorpe blocks. It may not create a new executable block, write arbitrary execution logic, grant itself new permissions, or directly access an underlying financial API.

## Definitions

### Execution Block

An execution block is a pre-built and pre-tested unit of executable functionality.

Examples:

- Retrieve invoice
- Validate vendor
- Retrieve transaction
- Initiate ACH payment
- Check payment status
- Issue refund
- Reverse transaction
- Update ledger
- Request human approval

Each block defines:

- accepted structured inputs;
- available operations;
- required permissions;
- validation rules;
- execution behavior;
- failure and retry behavior;
- structured outputs;
- logging requirements.

Blocks exist before an AI begins executing a process.

An AI cannot generate a new block during execution.

### Flow

A flow is an ordered composition of execution blocks.

Example:

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

Flows can be created in two ways:

**1. Forced flow**

The company defines the allowed sequence in advance.

The AI cannot bypass, reorder, or insert unauthorized executable steps.

**2. AI-assembled flow**

The company provides an objective and instructions, including instructions delivered through MCP. The AI determines which available Winthorpe blocks are required and assembles them into a flow at runtime.

For example, an AI might receive the objective:

```text
Resolve this customer's duplicate payment.
```

It may determine that the appropriate flow is:

```text
Retrieve Transactions
      ↓
Identify Duplicate
      ↓
Check Settlement Status
      ↓
Issue Refund
      ↓
Confirm Refund
```

The flow itself may never have existed before.

Every executable component of the flow, however, existed beforehand.

Therefore:

```text
Dynamic flow creation:       YES
Dynamic block creation:      NO
Arbitrary code execution:    NO
Direct financial API access: NO
Self-granted permissions:    NO
```

## Architectural Invariants

The following properties are always true regardless of whether a flow is forced or dynamically assembled.

### 1. Every executable action maps to an existing block

The AI cannot execute an operation for which Winthorpe does not have an available block.

If the AI determines that an unavailable capability is required, execution cannot proceed through improvisation. The missing capability must be implemented and made available separately.

### 2. Blocks, not prompts, define executable capabilities

Natural-language instructions can influence AI reasoning and block selection.

They do not create new executable authority.

For example, an instruction saying:

```text
If necessary, transfer funds from another account.
```

does not enable a transfer unless an appropriate transfer block exists and is available to that process.

### 3. Permissions are scoped to the current execution step

The AI does not receive persistent access to all financial systems required by the complete process.

When a block executes, Winthorpe makes available the permissions required by that block.

Those permissions do not imply access to unrelated blocks or future steps.

### 4. Sequential steps are isolated

Completion of one block produces verified runtime state.

Subsequent blocks operate using that state.

The AI cannot make a later capability available merely by claiming that an earlier step occurred.

### 5. Runtime state is authoritative

The model's context is not the system of record for execution.

Winthorpe maintains the authoritative state of:

- blocks requested;
- blocks executed;
- execution results;
- current process position;
- available permissions;
- failures;
- retries;
- completed actions.

## Execution Model

A simplified execution cycle is:

```text
External instruction / objective
              ↓
             AI
              ↓
     Reason about objective
              ↓
       Select next block
              ↓
      Structured request
              ↓
          WINTHORPE
              ↓
    Validate requested block
              ↓
   Validate structured inputs
              ↓
 Check current process state
              ↓
 Make required permissions available
              ↓
        Execute block
              ↓
      Persist actual result
              ↓
      Return structured output
              ↓
             AI
              ↓
      Reason about next step
```

The cycle repeats until the process completes, fails, or requires escalation.

## Structured Execution Boundary

The AI does not directly invoke arbitrary financial APIs.

It requests execution through known structured interfaces.

For example:

```text
Action: PAY_VENDOR
Vendor: vendor_546
Invoice: invoice_123
Amount: 34.87
Currency: USD
Rail: ACH
```

Because the request conforms to a known schema, Winthorpe can deterministically evaluate whether it represents an available executable capability.

The runtime can verify:

- whether `PAY_VENDOR` exists;
- whether the required parameters are present;
- whether the parameter types are valid;
- whether the requested combination is permitted;
- whether the block is currently available;
- whether prerequisite state exists;
- whether the required permissions can be provided.

A malformed or unavailable request does not partially execute.

It fails validation and returns control to the AI.

The AI may then correct its request, select another available block, or escalate.

## Sequential Isolation

Consider the following forced process:

```text
1. Retrieve invoice
2. Validate invoice
3. Obtain approval
4. Execute payment
5. Confirm settlement
```

The existence of Step 4 in the process does not mean the AI currently possesses the ability to execute Step 4.

Initially:

```text
AVAILABLE:
Retrieve Invoice

UNAVAILABLE:
Validate Invoice
Obtain Approval
Execute Payment
Confirm Settlement
```

After Step 1 completes successfully, its verified output changes the runtime state.

Winthorpe can then make the next permitted capability available.

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

This means sequencing is not merely expressed as an instruction such as:

```text
Do not pay until approval has been obtained.
```

It can be enforced through capability availability.

The payment capability does not become executable until the prerequisite state exists.

## Dynamic Flow Assembly

Sequential isolation also applies to flows created dynamically by the AI.

Suppose the AI receives an objective through MCP:

```text
Resolve this failed vendor payment.
```

Based on the available blocks and the information it retrieves, it might construct:

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

Alternatively, the information encountered during execution might cause it to choose:

```text
Retrieve Payment
      ↓
Check Failure Reason
      ↓
Request Updated Vendor Details
      ↓
STOP / WAIT
```

The AI is therefore not necessarily choosing the complete flow before execution begins.

It can reason after each verified result and determine the next appropriate block.

This provides process-level flexibility while preserving action-level constraints.

## Failure and Recovery

Failure behavior can also be encoded within blocks.

A financial operation may distinguish between:

- a request that definitely failed;
- a request whose outcome is unknown;
- a pending transaction;
- a retryable failure;
- a permanent failure.

These distinctions matter because repeating a financial operation after an ambiguous failure can create duplicate transactions.

A block can therefore contain execution behavior for idempotency, status checking, retries, and expected failure modes.

The AI reasons about the process-level response to the failure.

The block governs the mechanics of executing the financial operation correctly.

Recovery operations such as reversals are themselves explicit blocks. The AI does not obtain an unrestricted ability to undo arbitrary operations.

## Reuse and Composition

The same blocks can participate in many different processes.

For example:

```text
Retrieve Transaction
```

could be used in:

- refunds;
- reconciliation;
- disputes;
- customer support;
- transaction investigation.

Similarly:

```text
Execute Payment
```

could participate in:

- vendor payments;
- reimbursements;
- payouts;
- treasury operations.

A new business process therefore does not necessarily require new executable functionality.

If the necessary capabilities already exist, the process can be created by composing existing blocks.

This produces two independent forms of extensibility:

```text
NEW PROCESS
Existing blocks → new composition

NEW CAPABILITY
Build + test new block → add to block library
```

Only the second case expands what the system is fundamentally capable of executing.

## Role of MCP

MCP can provide instructions, context, and access to external capabilities involved in the reasoning process.

MCP instructions may influence:

- the objective the AI is trying to accomplish;
- business-specific rules;
- contextual information;
- which Winthorpe blocks the AI chooses;
- how the AI composes those blocks.

MCP instructions do not override Winthorpe's execution boundary.

An instruction received through MCP cannot create a Winthorpe block or grant the AI an executable financial capability that does not already exist.

Therefore:

```text
MCP / company instructions
          ↓
    influence reasoning
          ↓
          AI
          ↓
  selects/composes from
          ↓
 AVAILABLE WINTHORPE BLOCKS
          ↓
  deterministic execution
```

## What Winthorpe Does Not Do

To avoid ambiguity, Winthorpe's architecture should not be interpreted as allowing an AI to:

- write arbitrary financial execution code at runtime;
- create new financial operations during a process;
- directly call arbitrary financial APIs;
- grant itself permissions;
- bypass required blocks in a forced flow;
- treat its own assertion that a previous step succeeded as proof of completion;
- convert natural-language instructions into new executable authority.

The AI's autonomy exists primarily in **reasoning and composition**.

The execution primitives remain predefined.

## Summary

Winthorpe separates three things that are often conflated in agentic systems:

```text
WHAT CAN BE DONE
Defined by pre-built Winthorpe blocks.

HOW THOSE CAPABILITIES ARE COMBINED
Defined either by a forced company flow
or dynamically by AI reasoning.

WHAT SHOULD HAPPEN NEXT
Determined by the AI from instructions,
context, and verified execution state.
```

This allows different processes to use different levels of autonomy.

A company can require:

```text
A → B → C → D
```

for a tightly controlled process.

For another process, it can give the AI an objective and allow it to construct:

```text
A → F → C
```

at runtime.

In both cases, `A`, `B`, `C`, `D`, and `F` are existing, pre-tested execution blocks.

**The AI can create the flow. It cannot create the functionality.**
