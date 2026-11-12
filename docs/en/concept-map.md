# Rules Mapped to Established Concepts

> This concept map records the first version. The [second version](coordination-structure.md) supersedes its coordination, file ownership, and manager-capacity rules.

[中文](../concept-map.md)

Most rules in this skill have well-known counterparts in distributed systems, operations, and engineering management. We hit the problems first, wrote the rules, and only afterwards matched them to these names. Listing them serves three purposes:

1. Readers who know these concepts can quickly see what each rule is doing.
2. Anyone turning the rules into programs can borrow from the existing implementations in these fields.
3. It makes explicit **where multi-agent dispatch differs** from the traditional setting, which is the main thing this repository tries to add.

A mapping means "similar mechanism", not equivalence. The established practices listed here are not implemented in this repository.

## Delivery and follow-up

| Rule in this skill | Similar established concept | What is specific to multi-agent work |
| --- | --- | --- |
| Executor saves the delivery first, then notifies the manager | Transactional outbox; persist before publish | Agents tend to stop right after writing a summary; a sent notification does not necessarily wake the recipient |
| Acknowledge a delivery only after the next step is in place; "got it" does not count | Message acknowledgment, at-least-once delivery, ack after processing | The consumer is a model and can acknowledge in words without starting anything, so evidence of the follow-up must be tied to the ack |
| Only one manager consumes deliveries for a given task | Single active consumer, exclusive consumer | When a new conversation or model takes over, the old dispatcher may still be alive; handover must be explicit |
| A ledger entry saying "running" does not prove it is still running | Heartbeats and leases; stale-state detection | Nobody writes "exited" when an agent exits, so records and real state often diverge |
| The watcher reports only evidence-backed gaps, deduplicates, and stays quiet when healthy | Reconciliation loops (e.g. Kubernetes controllers); alert deduplication and suppression | The hard part is notification ownership: which conversation should receive the reminder. Scheduling the check is easy |
| A handoff keeps one current summary listing unconsumed deliveries and resume triggers | Shift handover checklists | The successor is often a fresh conversation with no context and can only rely on records |

## Ownership and resources

| Rule in this skill | Similar established concept | What is specific to multi-agent work |
| --- | --- | --- |
| Each file has only one writer at a time | Single-writer principle; code ownership (CODEOWNERS) | Separate worktrees make it look as if parallel writes to one file are fine |
| Resources are allocated centrally, counting tasks dispatched but not yet started | Scheduler reservations (e.g. kube-scheduler assuming a Pod in its cache before binding) | Several agents reading the same "machine is free" message rush to it together: a thundering herd |
| Agent slots, machine load, file write access, and desktop focus are accounted separately | Multi-dimensional resource scheduling; per-resource locking | Desktop GUI focus is an exclusive resource that must be serialized, which traditional CI rarely faces |
| The user talks only to the lead dispatcher and explains new information once | Incident command: incident commander and single point of contact | Executors share no context; if the dispatcher does not relay information, it is lost |

## Scale and process

| Rule in this skill | Similar established concept | What is specific to multi-agent work |
| --- | --- | --- |
| Lead dispatcher → managers → executors, grouped by backlog, not by department | Span of control; Erlang/OTP supervision trees | The bottleneck is the dispatcher's context and per-turn processing bandwidth, not the number of executors |
| A manager's capacity counts queued, awaiting-verification, and externally blocked items | Kanban WIP limits; Little's law | The illusion that "handing in source frees a slot" is especially common with agents |
| Status levels: source / build / platform operation / integration / release / user acceptance | Definition of Done; deployment pipeline stages in continuous delivery | Models tend to describe a lower-level pass as "done", so each status word must map to evidence |
| Save, then reopen in a new process and read back | Round-trip persistence tests | — |
| Integration split into lanes by disjoint file sets; only the final lane writes shared files | Merge queues; ownership-partitioned merging | Once parallel execution speeds up, a single integration writer becomes the new bottleneck |
| After release is authorized, freeze scope; new feedback goes to the next version | Code freeze, release branches, release trains | — |
| Self-created test resources are cleaned up and read back after last use | Test fixture teardown | Agents tend to "test and leave", leaving resources in real accounts |

## Defect remediation

| Rule in this skill | Similar established concept | What is specific to multi-agent work |
| --- | --- | --- |
| Stop-the-bleeding line and class remediation line run in parallel | Correction vs corrective action (CAPA) in quality management; ITIL incident vs problem management | With few people, only single-point fixes were affordable; parallel agents make class-wide fixes cost-effective |
| Roll out once the pilot passes; finish only when the three-state inventory is closed | Incremental migration with the strangler fig pattern and a migration inventory | Agents report "first batch cleared" as "class done", or expand scope without end, so the end point must be checkable |
| One lightweight ratchet per class: freeze existing locations, allow only removal | Baseline files in static analysis (e.g. PHPStan or detekt baselines); ratcheting quality gates | Compare by location identity, not by count, so removing one occurrence cannot pay for adding another |
| Look for mature libraries first; proposals go to the user | Build vs buy; architecture decision records (ADRs) | Under a "minimal change" constraint, agents default to writing it themselves |

## What this table does not cover

In traditional systems the following are guaranteed by programs. In multi-agent work they are often enforced only by written rules, and they are what we most want to discuss:

- **Whether an acknowledgment is real.** A traditional consumer's ack is emitted by code; a model can *say* it handled something without acting.
- **What triggers an execution turn.** A message can be delivered while an idle agent never starts a new turn.
- **Status inflation.** Models tend to describe stage results with more optimistic status words.
- **Broken context.** The successor is often a new conversation that can only recover from records and tool state.

A skill can state these requirements clearly, but write success, delivery retries, and automatic wake-ups still have to be implemented and verified by the host. See the [host adaptation notes](../../skills/reliable-agent-dispatch/references/host-adaptation.md) (Chinese).
