# How Winthorpe Works

Winthorpe is a runtime for letting AI execute financial processes autonomously while keeping each individual action constrained and verifiable.

The core idea is to separate **reasoning about what to do** from **the mechanics of doing it correctly**.

Rather than giving an AI agent broad access to financial systems and asking it to follow a long set of instructions, Winthorpe provides a library of pre-built execution blocks. Each block represents a narrow financial capability—such as retrieving an invoice, validating a vendor, initiating an ACH payment, checking transaction status, issuing a refund, or reversing a transaction.

A process is assembled from these blocks. The AI can reason about which blocks it needs and how to move through them, but each block defines exactly what can happen when it is invoked.

The result is a system where the AI retains flexibility at the process level while execution remains tightly bounded at the action level.

## 1. Pre-built execution blocks

The basic unit in Winthorpe is an **execution block**.

A block represents a discrete financial operation and packages together everything required to execute that operation correctly.

Conceptually, a block contains:

- **An action** — what the block is capable of doing.
- **A schema** — the structured inputs required to perform it.
- **Execution logic** — how the action should be carried out, including handling expected failures, retries, idempotency, and other operational behavior.
- **Permissions** — the systems, APIs, accounts, and operations that the block is allowed to access.
- **Validation rules** — conditions that must be satisfied before execution.
- **Outputs** — structured results that can be verified and passed to subsequent blocks.
- **Logging** — a deterministic record of what was requested, what was executed, and what happened.

For example, a `Pay Vendor` block might require a vendor ID, invoice ID, amount, currency, and payment rail. Its execution environment might allow it to retrieve the relevant payment method and initiate an ACH transfer, but nothing else.

A `Refund Transaction` block would expose a different schema and receive a different set of permissions.

These blocks are deliberately narrow. A block does not receive general access to the company's financial infrastructure simply because it is part of a financial workflow.

This is the Lego-block model: **build reliable financial primitives once, then reuse and combine them across many processes.**

## 2. Processes are assembled from blocks

Most real financial processes require multiple actions.

Paying an invoice, for example, might involve:

**Retrieve invoice → identify vendor → validate payment details → initiate payment → confirm settlement → update records**

A refund might require:

**Retrieve transaction → determine refundable amount → execute refund → confirm result → update customer record**

Winthorpe assembles these processes from the same library of reusable blocks.

This is different from hard-coding every complete workflow as a separate deterministic graph.

The same `Retrieve Transaction` block might appear in refunds, reconciliation, disputes, customer support, and treasury operations. The same `Send Payment` block might participate in vendor payments, payouts, reimbursements, or internal treasury movements.

The blocks define **how individual operations can safely occur**. The process defines **how those operations are combined for a particular objective**.

This allows Winthorpe to support many financial processes without independently implementing every possible process from beginning to end.

## 3. The AI operates through structured requests

The model does not directly manipulate the underlying financial APIs.

Instead, it interacts with Winthorpe through structured requests corresponding to the available blocks.

For example, rather than the model effectively saying:

> “Pay this invoice.”

and receiving general payment-system access, it must produce something conceptually equivalent to:

```text
Action: PAY_VENDOR
Vendor: vendor_546
Invoice: invoice_123
Amount: 34.87
Currency: USD
Rail: ACH
```

Winthorpe can deterministically validate this request because the expected structure is known.

The runtime can verify that:

- `PAY_VENDOR` is a valid action;
- the requested parameters exist;
- those parameters are valid for that action;
- the required fields are present;
- the combination of parameters is permitted;
- and the execution request corresponds to a real Winthorpe block.

If the model produces an invalid request, it does not become an ambiguous instruction that downstream software attempts to interpret.

It simply does not execute.

The model can then correct the request, choose another block, or escalate when it cannot proceed.

Natural-language reasoning therefore remains probabilistic, while the boundary between reasoning and execution is machine-verifiable.

## 4. Permissions are attached to execution, not to the agent

An important part of the architecture is that the AI agent does not need to hold all of the permissions required for the complete process.

Permissions are provided at the level of the individual execution block.

If a process contains five steps, Winthorpe does not expose the permissions required for all five steps when the process begins.

Instead, the runtime provides only the permissions required for the block currently being executed.

Conceptually:

```text
Step 1
Reason → invoke Block A → receive A permissions → execute → revoke

Step 2
Reason → invoke Block B → receive B permissions → execute → revoke

Step 3
Reason → invoke Block C → receive C permissions → execute → revoke
```

This creates a much smaller execution surface.

A model performing an invoice-validation step does not need access to the capability that moves money. A model executing a payment does not automatically receive the capabilities required to issue refunds or modify unrelated accounts.

The capability exists in the Winthorpe runtime, but it is not available to the agent until the corresponding execution block is legitimately reached.

## 5. Sequential step isolation

This becomes particularly important in multi-step processes.

Winthorpe treats sequential steps as **isolated execution environments**, rather than exposing an entire workflow to the model at once.

Suppose a process contains:

```text
1. Retrieve invoice
2. Validate invoice
3. Pay vendor
4. Confirm payment
5. Update ledger
```

Step 3 should not be executable merely because the overall process contains a payment somewhere in its future.

Winthorpe first executes and records Step 1. Its verified output becomes an input into Step 2. Only after Step 2 has completed successfully can the execution context required for Step 3 become available.

In simplified form:

```text
[ Step 1 ]
Permissions: A
      ↓
Verified output
      ↓
[ Step 2 ]
Permissions: B
      ↓
Verified output
      ↓
[ Step 3 ]
Permissions: C
```

Each step therefore has its own bounded context, inputs, outputs, and permissions.

The AI can reason across the broader objective, but **execution authority advances one step at a time**.

If the model attempts to jump directly to a later action, the necessary capability is simply unavailable.

This is different from telling the model, in natural language, “Do not pay the vendor until you have validated the invoice.” Winthorpe can make that sequencing an execution property rather than an instruction the model is expected to obey.

## 6. Outputs become inputs to subsequent blocks

Isolation does not mean that steps are disconnected.

Each block produces a structured output that can be persisted and used by subsequent blocks.

For example:

```text
Retrieve Invoice
        ↓
invoice_123

Validate Invoice
        ↓
validated_invoice_123

Authorize Payment
        ↓
payment_authorization_789

Execute Payment
        ↓
transaction_456

Confirm Settlement
        ↓
settled_transaction_456
```

The important distinction is that later steps operate on **verified state produced by earlier execution**, rather than solely on the model's representation of what happened.

The model cannot simply claim that a payment was authorized and proceed as though authorization occurred. The runtime has the actual output of the authorization block.

This creates a chain of machine-verifiable state through the process.

## 7. Failure, retries, and recovery are part of the blocks

Financial processes rarely follow a perfect happy path.

Payments fail. APIs time out. Transactions remain pending. Vendors have outdated details. A request may be valid but temporarily impossible to execute.

Winthorpe therefore treats failure handling as part of the execution primitive rather than leaving all recovery behavior to ad hoc model reasoning.

A block can define what constitutes a retryable failure, when an operation should be checked rather than repeated, what information is required before retrying, and which recovery operations are available.

This is particularly important for money movement, where blindly repeating an operation can be much worse than failing to perform it.

The model can still reason about the broader situation—for example, deciding whether an alternative payment method is appropriate—but the mechanics of safely retrying an ACH initiation do not need to be reinvented every time an agent encounters a timeout.

The same principle applies to reversals and compensating actions. They are explicit execution capabilities with their own schemas and permissions, rather than arbitrary actions available to the model.

## 8. The runtime maintains the authoritative state

Throughout the process, Winthorpe—not the model's context window—maintains the authoritative execution state.

The runtime knows:

```text
What process is running
        ↓
Which blocks have completed
        ↓
What each block actually returned
        ↓
Which block is currently executable
        ↓
Which permissions that block receives
        ↓
What actions were attempted
        ↓
What ultimately happened
```

The model uses this state to reason about what to do next, but it does not define the state simply by asserting it.

This distinction is important because models can misunderstand prior events, lose information from context, or produce an incorrect representation of what happened.

Winthorpe's execution state is based on actual execution results.

## 9. Building new processes becomes composition

Once a useful library of blocks exists, supporting a new financial process increasingly becomes an exercise in composition rather than building new execution infrastructure.

If Winthorpe already knows how to:

```text
Retrieve invoice
Validate vendor
Check balance
Move money
Check transaction
Reverse transaction
Update ledger
Request approval
```

then many new processes can be constructed by combining those capabilities in different ways.

Some processes may require a new block. Others may require only a new arrangement of existing blocks.

This is where the Lego analogy becomes most useful.

The blocks are not complete applications. They are reusable financial execution primitives with known interfaces and known behavior. Different processes assemble those primitives differently.

Over time, Winthorpe's library can cover increasingly complex financial operations without the number of underlying execution capabilities growing at the same rate as the number of processes it supports.

## 10. Where the AI fits

Winthorpe does not attempt to turn financial operations into fully deterministic workflows.

That would remove much of the reason to use AI in the first place.

The model remains responsible for the parts where reasoning is valuable: understanding intent, interpreting context, dealing with ambiguity, choosing appropriate actions, responding to unexpected situations, and deciding how to proceed when several valid paths exist.

Winthorpe handles the boundary where that reasoning turns into real-world execution.

The division is roughly:

```text
AI
Understand → Reason → Decide → Request action
                              ↓
                         WINTHORPE
                   Validate structured request
                              ↓
                   Load execution block
                              ↓
                   Grant narrow permissions
                              ↓
                         Execute
                              ↓
                   Verify + persist result
                              ↓
                              AI
                       Reason about next step
```

The AI is therefore not reduced to a component inside a rigid workflow, but neither is it given unrestricted access and trusted to execute a financial process correctly based solely on instructions.

It can reason freely.

Execution remains constrained by the blocks available to it, the structure of those blocks, the current state of the process, and the permissions Winthorpe makes available at that specific step.

That separation—**flexible reasoning above a deterministic execution boundary**—is the basic mechanism behind Winthorpe.
