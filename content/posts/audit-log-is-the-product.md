---
title: 'Audit Log Is the Product'
description: "In a regulated environment, the audit trail has the same status as the feature. The durable execution engine you pick settles where your evidence commits — before you write a line of compliance code."
date: 2026-06-24
draft: false
tags: ["compliance", "durable-execution", "architecture", "temporal", "dbos", "regulation"]
cover:
  image: "images/og-audit-log.png"
  alt: "Diagram comparing Temporal (two separate commits with a crash window) and DBOS (one atomic commit in the same database) as durable execution engines for regulated AI workflows"
  hiddenInSingle: true
---
# The Audit Log Is the Product

*Where your workflow engine keeps its history decides whether your AI system survives an audit. Most teams make that decision by accident.*

---

There's a question that reshapes how you architect AI systems in a bank, and it doesn't come from your users. It comes from an auditor, eighteen months after go-live:

*"Show me what the system did in this case — and prove that what it recorded is what actually happened."*

The first half is easy. Everyone has traces now. The second half hides two questions people conflate: were the record and reality *consistent when written*, and has the record *stayed unaltered since*? This piece is about the first — the engine you pick settles it on day one, before you write a line of compliance code, probably without your noticing you decided. The second is a layer you add on top, and I'll come back to it.

I spent ten years building regulatory reporting systems before I started building AI platforms. The habit that carried over: I read every architecture through the eyes of the person who will eventually have to defend it to a regulator. And by that reading, the most consequential property of an agent system isn't its accuracy, its latency, or its framework. It's where the evidence lives.

## Agents are long-running transactions

Strip the vocabulary away and a production agent workflow is a multi-step process: fetch context, call a model, validate the output, call a tool, wait for a human approval, write a result. It runs for minutes, sometimes days. Any step can fail. The process must survive a pod restart without double-charging anyone or losing its place.

We've had a name for this problem for decades — it's a long-running transaction — and the industry has converged on a primitive for it: durable execution. Checkpoint every step, replay on crash, resume exactly where you stopped. Temporal defined the category; DBOS, Restate, and others crowded into it. I'll spend this piece on the two poles — Temporal as the cluster, DBOS as the library — because the axis I care about is sharpest between them; the others sit on the spectrum it defines. The comparisons you'll find online argue about operational cost and developer experience, and they mostly reach the same conclusion: the library is cheaper to run than the cluster.

That's the wrong axis for my world. In a regulated environment, the interesting property of durable execution isn't that your workflow survives a crash. It's that checkpointing every step produces, as a side effect, a complete record of what the system did.

In most companies, that record is observability exhaust. In a bank, it's a deliverable.

## The trace is not exhaust

This isn't a stylistic preference. It's written down.

The EU AI Act requires high-risk AI systems to automatically record events over their lifetime (Article 19) — and Article 26(6) puts the retention obligation on *deployers*, not just providers: keep the logs the system generates, "for a period appropriate to the intended purpose ... of at least six months." If you run an AI system inside a bank, that's you, regardless of whose model you call. And the same clause has a sting in its tail for my world specifically: deployers that are financial institutions must maintain those logs *as part of the documentation they already keep under Union financial-services law*. The regulator isn't asking you to invent a new audit trail — it's telling you the AI system's trace now lives under the same regime as the rest of your books. BCBS 239 adds that the risk data such a workflow touches must be traceable to its source — lineage as a property of the data, not a diagram in Confluence. DORA makes every ICT service supporting a critical function a managed third-party risk, with a register, a contract review, and an exit strategy.

Read together, these say something uncomfortable: for a regulated AI workflow, the audit trail has the same status as the feature. The agent that processes a payment document and the record proving how it processed it are the same deliverable. If you can ship the first without the second, you haven't shipped.

Once you accept that, the question for choosing a durable execution engine changes. Not "which is cheaper to operate" — but:

**Can the trace and the system of record disagree?**

## One question, two architectures

Temporal's answer is: yes, briefly, and you reconcile.

Temporal keeps a complete event history for every workflow — that history *is* its replay mechanism, and it's genuinely complete. But it lives in Temporal's own persistence, behind its own cluster, in its own trust boundary. Your activity writes a payment row to your Postgres; Temporal records "activity completed" in its history store. Two systems, two commits, and a window between them. Crash inside that window and the history can say *completed* while your database says nothing — which is why every Temporal tutorial teaches idempotency keys before it teaches anything else. The discipline works. But notice what it is: a *procedure* that keeps two systems in agreement. The trace is complete, the data is correct, and their agreement is something you maintain, audit, and occasionally repair.

I'd call this traceable-in-retrospect: you can always reconstruct what happened, provided your reconciliation held.

DBOS's answer is: for the writes that land in your own database, no — by construction.

DBOS is a library, not a cluster. Your workflow state checkpoints into the same Postgres your application already runs on. And for steps marked as transactions, the business write and the durability checkpoint commit *atomically*:

```python
@DBOS.transaction()
def write_payment(record: dict) -> str:
    DBOS.sql_session.execute(
        sa.text("INSERT INTO payments (...) VALUES (...)"), record
    )
    # The checkpoint recording that this step completed commits
    # in this same transaction. One commit. There is no state in
    # which the trace and the payment table disagree.
    return record["payment_id"]
```

For that step there is no window and nothing to reconcile. But the guarantee is narrower than "the trace and reality can never disagree," and I'd rather mark the boundary myself than let an auditor find it: the atomic commit covers **side effects that are writes to the same database.** External calls — the model invocation, the tool call, the API POST — are HTTP, not SQL. They can't join your transaction, so DBOS records their outcome *after* the call returns, and a crash in between re-runs the step. That's at-least-once, the same idempotency discipline Temporal needs — and on an async stack (FastAPI, asyncpg) it's the *only* mode available, because `@DBOS.transaction` is synchronous; the async path uses at-least-once steps throughout.

So be plain about it: **on external effects, the two engines are the same.** Both record after the fact; both lean on idempotency. The atomic commit only pulls ahead for the writes that land in your own Postgres — the payment row, the decision, the state transition — where the evidence and the record commit together. The model call that *informed* the decision is recorded at-least-once like everyone's; but that effect was never in your system of record to begin with.

Which means the real difference isn't atomicity. It's **locality** — and that is the whole argument. The evidence isn't kept *about* the system of record; it's kept *in* it: same instance, same backup regime, same access controls your DBA already answers for. That's structural — true regardless of how you serialize, how you query, or whether anyone ever reads it. (It is also not the same as proving nobody edited the record afterward; atomicity is consistency-at-write-time, not tamper-evidence. That's a separate layer, and the subject of the next piece.)

Locality is not the same as **legibility**, and I want to keep them apart because only one is free. When the auditor asks her question, the answer is a SQL join:

```sql
SELECT w.workflow_uuid, w.status, w.inputs, o.function_name, o.output
FROM dbos.workflow_status w
JOIN dbos.operation_outputs o USING (workflow_uuid)
WHERE w.workflow_uuid = 'task-2026-06-0042'
ORDER BY o.function_id;
```

Whether that join returns something an auditor can *read* is a design choice, not a guarantee. DBOS's default serializer stores base64-encoded pickle — the join returns opaque blobs. To get legible rows you swap in a JSON serializer (and, in my case, write one that handles the types the stock one rejects), and you write your steps to return structured results rather than log sentences. Do that, and the trace of a real workflow looks like this — every decision and effect as a queryable row, in execution order:

```
fn  step                      output
 1  classify                  "phase_2"
 2  already_migrated          false
 3  create_external_session   "2d65cdb7-…"
 4  create_internal_session   {…"created":true}
 5  initialize_session_fields  {…"coverage_score":0.0,"field_states_synced":27}
 6  start_session             {…"context_messages":38}
 7  complete_session          {…"session_status":"completed"}
 8  update_preapproval_date   {…"updated":false}
 9  is_preapproved            false
```

This is not a reconstructed timeline. It's the workflow's own decision record: the classification (1), the precondition it checked (2), the external session it created (3), and — step 9 — the branch condition that determined it would *not* proceed to phase 3. Every one a row, in order, in the database I already govern. And the trace is honest about which is which: step 3 is an external call, recorded after the fact (at-least-once); steps 4–8 are local writes.

So: **locality is by construction; legibility is by configuration.** The first is the argument. The second is the part you have to build — and worth building, but don't sell it as something the engine hands you for free.

And if you've read my piece on building agents as finite state machines instead of prompt loops, you'll recognize the axis underneath both — it's the same one. In regulated systems, control and evidence aren't qualities you bolt onto an architecture afterwards. They're properties the architecture either has or doesn't. Locality is one of them. The engine that writes its evidence into the boundary you already govern has it; the one that writes elsewhere doesn't — and replication or log-shipping mitigates that for reads, but it doesn't undo where the system of record commits.

## The question that isn't on any comparison chart

There's a second compliance angle, and it's specific to *managed* Temporal.

Temporal Cloud is excellent operationally. But think about what it holds: the complete execution history of your regulated process — every step, every input, every decision — in vendor-operated infrastructure. Under DORA, that's not an architecture detail. That's an ICT third-party arrangement supporting a potentially critical function: register entry, contractual provisions, concentration risk assessment, exit strategy. The trace of your process living outside your boundary is precisely the kind of thing the regulation was written about.

You can self-host Temporal and the question goes away — replaced by a different one: you now operate a distributed system (server, workers, persistence, visibility store) whose primary job, from a compliance perspective, is keeping evidence about your *other* system. That's a real cost, and it should be priced as one.

With the Postgres-resident approach, the evidence sits inside a boundary you've already defended to the regulator a hundred times. Banks know how to govern Postgres. That institutional muscle memory is worth more than any feature.

## Where this argument breaks

Honesty section, because this argument has a boundary and I'd rather draw it myself.

Temporal wins — clearly — on very high fan-out, multi-tenant and multi-region orchestration, polyglot teams, and organizations that already have a platform team running it. Those companies process volumes where "one Postgres" stops being elegant and starts being the bottleneck. And at that scale my own argument inverts: the single database is now a single point of failure for the application *and* its evidence. Losing both in the same incident is not a story you want to tell a regulator either.

And one more honesty while I'm here: locality puts the evidence in your boundary, but it does not enforce your *retention* policy for you. The EU AI Act's six months is an obligation you operate on the checkpoint schema — and guard against premature cleanup — not something the engine inherits because the rows happen to live in your database. (The tamper-evidence gap from earlier is the other half of this; both are layers you add, and locality's contribution is only that you add them somewhere you already control.)

My claim is narrower than "DBOS is better." It's this: if your workflow's side effects land mostly in your own Postgres, your volumes are bank-workflow volumes rather than Uber volumes, and you carry audit obligations — then the engine whose evidence commits inside your compliance boundary is the structurally correct choice, and the burden of proof sits with the cluster, not the library. If you genuinely outgrow it, you'll know, and you'll migrate with evidence intact. The reverse migration — cluster to library — is the one nobody plans for, because nobody admits they bought the cluster for workloads they never had.

## The decision rule

Stop choosing a workflow engine and then asking how to make it auditable. Choose where your evidence must live — inside which boundary, under whose governance, joined to which data — and treat the engine as an implementation detail of the audit log.

For most regulated AI workflows, that reasoning lands you on the database you already have. Which is a strange place for an article about cutting-edge agent infrastructure to end: the most defensible component of your AI platform was installed before your AI platform existed.

The audit log is the product. Build accordingly.

---

*Next in this series: when an agent generates the workflow plan itself, the plan becomes part of the trace — and the approval of that plan becomes an artifact you can hash, bind, and defend. That changes what "human oversight" means in practice.*
