# How This Multi-Machine, Multi-Agent Workflow Took Shape

> English translation of [the Chinese original](../workflow-evolution.md). If the two differ, the Chinese version is authoritative.

> Reading guide: for the multi-manager architecture itself, jump to [section 8](#8-from-reminding-the-dispatcher-to-sharing-the-dispatchers-bandwidth) or read [Multi-Manager Dispatch Architecture](manager-architecture.md). Sections 1–7 cover the problems we hit before adding the layer.

In APlus development, several AI agents split the work of changing code, compiling, and driving SketchUp on Mac and Windows to verify the results. The project owner wanted to talk to a single lead dispatcher day to day: raise product problems, add facts from the field, and make the decisions that only they can make. The dispatcher organizes everything else.

At first Claude was mostly the lead dispatcher, and the owner found the collaboration fairly smooth. Later that setup was interrupted and Codex had to take over. The migration exposed coordination that had only been held together by conversational habit. An executor would deliver, and nobody picked up the next step. The owner would discuss a new idea, and the dispatcher would answer and stop. Machines sat idle while tasks were not routed to them. After long silences the owner had to come back and check whether anything was still running.

The rules in this repository came out of repeated use, questioning, and correction. The sections below follow how the problems evolved; in practice several of the fixes overlapped. Tool names are given for context, and every experience described here comes from this one team.

## 1. From "the executor finished" to "someone picks up the delivery"

Once work ran in parallel, every execution line reported progress, but a lot still lay between a partial delivery and the end of the user's problem. After source changes and tests pass, there may still be a new build, an operation on the target machine, a save-and-reopen, inclusion in a combined build, and finally acceptance by the user repeating the original operation.

The owner kept having to ask: this one was handed in, so why hasn't anything moved? Judged by each agent's completion notice, everyone had a result. The actual user problem could still be stuck between two stages.

So we made task state explicit: who is executing, who has handed in a result, and what is waiting for the dispatcher. The current queue has the states *queued*, *running*, *awaiting dispatcher*, *waiting on external input*, and *closed*. Once an executor saves a delivery, the task moves to *awaiting dispatcher*. "Delivered" means the executor handed in one stage of results and the next responsibility now belongs to the dispatcher.

A delivery carries the actual result, evidence, remaining work, and a concrete next step. After checking it, the dispatcher must actually resume the original owner, hand it to a named successor, or state the specific queue position or external gap, and only then acknowledge the delivery as handled. Replying "got it" does not count as arranging the next step.

This design now has program support. A local queue stores registration, delivery, and handling records separately, and current state is rebuilt from those records. Executors write only their own deliveries; the dispatcher assigns and acknowledges. A check tool reports unacknowledged deliveries and cases where an executor has exited while the queue still says *running*. When a task resumes, it reads these records first so that stages that already passed are not redone.

The queue implementation is not included in this public repository. The information needed for delivery, acknowledgment, and recovery is kept in the skill and templates so other teams can map it onto their own execution systems.

## 2. A pre-exit check still leaves a timing gap

The next step was to require the dispatcher to check, before ending every reply: are there new deliveries, which executors have finished, and is any runnable work left unconnected? After the owner asks about progress or switches topic, the dispatcher also has to return to the original task records.

Then a new scenario appeared in real use. A sub-agent was still running while the owner and the dispatcher discussed a new product idea. The dispatcher finished writing up the discussion and ended its turn. Later the sub-agent completed, its result still went unhandled, and only after a while did the owner notice that work had stalled again.

Two cases have to be told apart here. If the delivery arrived before the dispatcher's reply ended and was not handled, the pre-exit requirement was not followed. If the executor was still working at check time and delivered after the reply ended, the check was entirely correct, and something else is still needed to make the dispatcher run again.

State records keep the work from being lost. When the dispatcher reads those records and how it continues are separate problems. That distinction shaped the design of the watcher.

## 3. A standalone scheduled job first, then a watcher inside the dispatcher's own agent tree

To catch post-reply misses, the owner set up a scheduled job running every ten minutes. It was meant to read the task records and send a reminder whenever a delivery was waiting.

In the owner's retrospective, the key difficulty was notification ownership. The standalone job could not tell which dispatcher conversation should receive the reminder. The check ran on time, but its result did not reliably reach the dispatcher responsible for that batch of work.

The owner then wrote the watcher into the skill: whenever a conversation takes over as lead dispatcher for a batch, it creates a low-cost watcher sub-agent for that batch. It can use a low-cost model such as Luna, or a reasoning depth suited to the task. The watcher is tied to the current dispatcher, reads the same task records, and reports anomalies through the existing agent messaging channel.

This change put three things together: which tasks the watcher looks at, whom it reports to, and which channel delivers the report. The owner observed that missed deliveries were returned to the right dispatcher more effectively.

The watcher's duties later needed tightening. It checks only for dispatch gaps backed by evidence: a delivery left unhandled for a long time, an executor that has exited while the record still says running, or runnable work with no owner. It does not take over the queue or reassign work, so that two dispatchers never direct the same batch.

Reminders must be based on the latest state. The dispatcher may already have handled an old event, and an executor's historical completion text may differ from its current run state. A watcher reading only old records would re-report handled items as missed. So we added requirements to read current agent state, check the most recent receipts, and avoid duplicate reminders. A healthy state does not produce repeated all-clear messages; after the dispatcher acts, it reports back what it actually did.

This watcher depends on the delivery and run capabilities of the tools in use. Delivery of reminders into an active dispatcher can be recorded directly; whether things recover after stopping, closing, or restarting the application has to be verified separately. Ten minutes was this team's choice, and the interval also has to account for the agent slot it occupies and the reasoning cost of each round.

## 4. When sub-agents picked their own machines: local congestion, global idleness

Early in multi-machine development, sub-agents chose which computer to run on. Each agent saw only its own task and part of the resources, and could not know which machine the other executors were about to use.

When the owner needed a computer, they told the dispatcher it was temporarily unavailable. Several sub-agents might then switch to the same other computer at once. When the owner was done and said the machine could take work again, the dispatcher sometimes just wrote that into a handoff document and suggested routing tasks there. Several executors read the same "available" message and rushed together onto the newly freed machine.

So despite several Macs and Windows machines, tasks concentrated on a few of them and the rest of the capacity went unused. The owner had to keep watching which machine was too busy and which was idle, and come back to ask for adjustments.

The owner went looking for multi-machine resource monitoring and put real-time queries into the allocation flow. Our records keep a trial of Glances. After that combination reported an abnormal total CPU usage of zero, the current query entry switched to system counters and existing capabilities such as psutil. Choosing a tool also means checking whether its data can be trusted.

The more important change was in responsibility. The dispatcher queries the current resources of all approved machines in one place and, combining task fit with work already assigned, names the machine for each step. Executors work where they are assigned, and when an assigned action fails they report it; the dispatcher decides whether to move. When a machine is paused or resumed, the dispatcher adjusts tasks one by one instead of broadcasting a message that makes every executor migrate at once.

The dispatcher also has to count tasks that were just dispatched but have not started. Looking only at each query's idle snapshot, consecutive dispatches can still put several heavy tasks on the same momentarily idle computer.

Over time we separated several kinds of constraint: agent slots, each machine's own memory and load, write access to shared files, and the desktop focus that truly needs a mouse and keyboard. When one machine's foreground is occupied, only the conflicting steps wait; other machines and work that does not need the foreground continue. Actual capacity changes with load and isolation; one historical concurrency figure is not a general limit for every machine.

## 5. The dispatcher also has to be someone people want to keep working with

After moving to Codex, the owner also ran into communication problems. Sometimes the dispatcher went quiet for a long time. Sometimes replies were full of internal IDs and process detail, and the owner still could not tell how far the product problem had progressed. Sometimes it carefully answered the latest discussion and then failed to keep holding the parallel work already in flight.

So the owner went back to Claude and asked it to summarize how a dispatcher can work more smoothly with a person. Those summaries were folded into the project's dispatch skill and feedback workflow. The change history shows additions on prompt responses, communication during long tasks, and "follow up first, report in full afterward".

The first rule is plain: **reply with a sentence first, then work.** On a new message, respond first: say what is already known and what still needs checking, then read state, call tools, and dispatch. The owner should not have to guess, during a silence, whether the request was received. After the short reply, investigation and execution continue; the reply itself is not a delivery.

The second rule is staying visible while working. In the original practice, the owner wanted no more than about one minute without a response during continuous work. New results, changes of direction, or blockers get a short note; while waiting, say what you are waiting on and when the next check is. This cadence is a collaboration preference between the two parties. Its purpose is to keep the owner informed of changes without repeating the same status when there is no new fact.

The third rule is reporting in terms of the user's problem. The first sentence states what the user can see and the current gap. Multiple items are listed as "problem, current result, what is still missing"; branches, processes, and detailed logs go into evidence. The owner can then judge directly what is usable and what still needs verification, without translating internal codes first.

The fourth rule is that status words must mean something. *Dispatched*, *running*, *source candidate*, *passed a real operation*, *released*, and *accepted by the user* are reported separately. Calling a tool in preparation is not yet *running*; passing tests does not replace acceptance by the owner or the person who reported the issue. Time estimates need a basis; when no estimate is possible, give the next checkpoint.

The fifth rule is that the owner explains additional information only once. When the user gives the dispatcher a new parameter, model, or decision, the dispatcher decides which task it belongs to, passes it to the original owner of that task, and updates the record. When an executor needs material or a decision, the dispatcher handles that communication centrally, so the owner does not have to repeat it across sessions.

The sixth rule is keeping discussion and execution going at the same time. When the owner discusses a new idea, the idea can be explored fully; if it is only authorized for discussion, it stays a discussion. Previously authorized tasks continue until the owner explicitly pauses, cancels, or changes the goal. After answering the latest message, the dispatcher returns to the existing deliveries and follow-ups, so one discussion does not leave the whole batch unattended.

Finally, routine technical arrangements and machine allocation are the dispatcher's call; existing authorization is not turned into repeated approvals. Decisions that would really change the business outcome are brought to the owner together. On receiving a delivery, the dispatcher checks the necessary evidence and sets up the next step first, then reports in full. The owner should not end up chasing every item one by one just because the dispatcher wrote a summary.

These rules are distilled into the [skill entry point](../../skills/reliable-agent-dispatch/SKILL.md) and the [communication notes](../../skills/reliable-agent-dispatch/references/communication.md). They come from specific friction in use; other teams can adjust the cadence and format to their own habits.

## 6. Desktop software adds real conditions to "done"

APlus tasks have to reach a real SketchUp scene. Beyond code and unit tests, we must confirm that the running program contains the current candidate, that the operation used the right input, and that the target platform can actually perform the action. Where persistence is involved, the result is only checked after saving and reopening in a new process.

So we record source, build, required platform operations, integration, release, and user acceptance separately. Each item requires only the levels relevant to the user outcome. If a platform cannot be verified, state the specific gap and continue with everything else that can run. Self-built scenes help understand the mechanism, but when they cannot reproduce the original situation, acceptance on the original case remains an open item.

These distinctions also shape dispatch. When an executor hands in a source candidate, the dispatcher needs to know whether what is missing next is an artifact, a target platform, real input, or user confirmation. Leaving a named owner and next step at the end of each stage is what makes the follow-up possible.

## 7. What we want to discuss by sharing this process

Looking at the design and public prototype of DeepSeek Super-agent, we found many similar problems around persistent task ownership, independent execution, delivery, and reducing manual chasing, as well as differences in how responsibilities are divided. See the [prototype comparison](../superagent-comparison.md) (Chinese).

We especially want to discuss: which rules a skill can help a model follow, and which need program guarantees; how watchers and deliveries can return to the right dispatcher when several human–agent conversations exist at once; how agent run capacity and real machine resources should be coordinated; and how to judge real progress while keeping communication with the product owner smooth.

These improvements are still happening as we use them. We publish them so others can see the reason behind each change, reuse what works, and point out problems we have not yet seen.

## 8. From reminding the dispatcher to sharing the dispatcher's bandwidth

By November the owner observed a new bottleneck: with a dozen or so agents and about ten execution lines, work would already start to fall apart midway. The dispatcher could not keep up with receiving deliveries, reviewing results, and arranging next steps. Executors had finished and machines might be free, but the next stage did not start in time.

The obvious idea was another polling sub-agent to remind the dispatcher promptly. Reminders can detect misses, but they do not increase how fast the dispatcher processes deliveries. Every delivery still returns to the same entry point to be read, judged, and arranged one by one; more timely reminders only reveal the backlog earlier. The watcher solved whether messages arrive. This time the problem was who has the capacity to handle them once they arrive.

The owner recalled an interview with an Anthropic engineer that discussed an architecture where several project managers organize the work. That inspired him to try hierarchical dispatch in practice. This is only the author's recollection of the inspiration; the specific interview has not been verified, and no name, link, or quote is added here. The owner first guessed that one sub-agent could reliably look after perhaps five or six execution lines; later he used a maximum of ten open lines per manager as this round's tested configuration. The former was a hypothesis and the latter a working configuration; neither is a general limit of the models.

The new division is *lead dispatcher, manager, executor*. The lead dispatcher still holds the original user goals, global priorities, cross-group conflicts, and is the only contact for the user. A manager directly organizes the executors in its group: it receives deliveries, does the necessary review, arranges builds and real acceptance, and follows the same user action to the end. The number and grouping of managers change with the work and are not fixed departments. Items waiting for verification, queued, or waiting externally still take management attention, so they all count as open; handing in source does not free a slot.

This also changed what the watcher reports to. The watcher first sends a group's anomalies to that group's manager; only if the manager cannot continue, or the problem crosses groups, does it go to the lead dispatcher. Routine deliveries no longer queue at the lead dispatcher before they can proceed. The lead dispatcher still checks at key points "what did the user originally want, and what is being delivered now", so that the hierarchy is not read as permission to narrow the goal at each layer.

Resource allocation follows the same principle. One existing manager doubles as resource coordinator. It looks in one place at the real-time memory, CPU load, disk, and in-flight work of approved machines, and gives each group its current share. Business managers then name the exact execution location within that share. Tasks just dispatched but not yet started are counted; executors cannot each choose machines from the same idle snapshot. Compilation, tests of already-built programs, desktop focus, and lightweight source work are judged separately; when one step is constrained, only the affected step waits. A new machine joins the pool only after the user registers and enables it and its connection and tool capabilities have been checked. A capability statement is not proof that work can run there.

The owner's latest observation is that the scale of effective collaboration grew from a dozen or so lines to about fifty, and that with about fifty agents running concurrently the collaboration still stays quite efficient. He stresses that the change goes beyond his earlier "throughput doubled" description. What can be compared is a roughly fivefold collaboration scale in actual use; it cannot be converted directly into five times the work completed per unit time.

Another example that impressed him: one large refactor compressed what he had estimated as several weeks of work for a traditional team into a little over three hours. He sees it as validating the throughput of organizing work in parallel. The workload and time are the author's estimates for the selected first round. They are not an independent benchmark or a controlled cross-team comparison, and they do not show that the repository-wide rollout of all seven classes is finished.

Relating this to the public DeepSeek Super-agent case mentioned earlier, the owner believes many people are already exploring similar architectures on their own. What he is trying here is to scale effective collaboration through hierarchical management. That is the purpose of sharing the process; it is not a claim to be first in the industry, nor a judgment on whether other systems currently have managers.

Whether the hierarchy stays effective still depends on whether the interval from delivery to follow-up shrinks, whether backlog and duplicate execution decrease, and whether real acceptance keeps up. The public skill defines only responsibilities and checkpoints. Whether managers can actually be resumed, whether they keep running while the lead dispatcher is talking to the user, and how messages and wake-ups behave after a conversation closes all have to be verified in the tools you use. Detailed duties are in the [manager and integration notes](../../skills/reliable-agent-dispatch/references/manager-hierarchy.md) (Chinese).

## 9. Execution can run in parallel; so must integration

Once business execution was grouped, dozens of lines could deliver at the same time, and another funnel appeared: every candidate still said "hand to the single integration owner". One person had to register, merge, handle shared files, and prepare the combined program. Business managers followed up faster, and candidates formed a long queue at the integration entry.

The original rule that each file has only one writer exists to prevent overwrites. The rule is valuable, but we had stretched it into "the whole integration stage has only one writer". Frontend, desktop shell, and business kernel were waiting on the same person even though their files did not overlap.

So integration was also divided into lines by actual write scope, organized by an integration manager. There can be one line for the frontend, one for the desktop shell, and the kernel can be split further by current modules. Each line maintains its own sub-combination and runs the tests relevant to its layer. The split is based on non-overlapping files and whether dependencies can be handled independently, not on headcount, and certainly not one person per file.

Shared registries, dependency lock files, and generated embedded artifacts are still handled by the final integration line. The final line merges the sub-combinations, updates or regenerates shared files once, builds in batch, and runs combined acceptance. Direct reviews and real-operation evidence already completed on business lines are reused rather than redone item by item; interactions that can only appear after combination are still checked in the final artifact.

For entry points migrated with the same pattern, review is also sampled by class and batch, focusing on representative entries; new business logic or new contracts are still reviewed independently one by one. Sampling does not remove the direct tests, real-entry acceptance, or static guards each item needs.

This change separated the independent parts of what used to be a single write entry; shared files and combined acceptance remain explicit merge points. It has only just entered this round's rules. We cannot yet claim it has removed all integration waiting, and we have not independently measured the speed-up. We still need to check whether it actually reduces candidate waiting time and whether it introduces new combination regressions before adjusting the grouping.

## 10. After the pilot, how to continue, and where the rollout ends

Large class-wide refactoring ran into three problems pulling in opposite directions. The first plan said that rolling out after a pilot needed separate approval, so the dispatcher stopped right after finishing the pilot. The owner wanted already-authorized class-wide work to keep moving, so we changed it: pick one real scenario per class, and once the pilot passes, roll it out to the other entry points of that class without waiting for other classes.

Then two more problems appeared. Sometimes finishing one batch of registered entry points was reported as the whole class, or even the overall goal, being done. After misses were found, people kept adding entry points and checks with no visible point at which to close. Closing the list for the selected first batch was once mistaken for the end of the whole goal, and that conclusion was later withdrawn. The tests and real-operation evidence from the first batch remain valid; they just prove less than was claimed.

The cause was that "start the rollout" was clearly defined but "the full set and the end point of the rollout" were not. Now each class first inventories every old entry point within the authorized scope and marks each with one of three outcomes: *migrated*, *existing capability is sufficient*, or *not doing for now*. The latter two need actual evidence; *not doing for now* also needs a reason and a condition for reopening. Items waiting for an execution slot, not yet investigated, or not yet accepted cannot be renamed into *not doing for now*.

The inventory is not enough on its own. The class also needs one registered, lightweight static ratchet. It scans for the old pattern and records existing exceptions in an allowlist that may only shrink; new code that takes the old path fails. The rule is one ratchet per class, each finishing within one second, with no compilation and no desktop software, run when a candidate is delivered. The allowlist keeps checkable identities for the exceptions, so deleting one occurrence does not permit adding another old-style occurrence elsewhere.

The end point thus becomes two checkable conditions: every old entry point in the class has an explicit disposition, and the static ratchet stops new entry points from going back to the old pattern. Once both hold, rollout of that class stops; nobody keeps hunting for out-of-scope work to raise coverage. The ratchet only guards an explicitly defined pattern. It does not replace business tests, real desktop operations, or verification by reopening in a new process after saving.

This definition lets "continue once the pilot passes" and "stop at the end point" both hold, and makes premature closure something that can be pointed out concretely. As of this update, the repository-wide rollout of the seven classes has not reached its end point for all of them; the completed first batch and the remaining rollout are recorded separately. The full method is in [class-wide remediation and rollout end points](../../skills/reliable-agent-dispatch/references/bug-by-class.md) (Chinese).

## 11. Beyond the minimal fix, look for mature solutions first

Looking back at the first refactoring round, we noticed a habit: many problems were general ones, yet we ended up writing almost all of it ourselves. The plan said "borrow ideas", the rules forbade adding frameworks without approval, and the frontend had no ready dependency pipeline. On top of "minimal change" and "fewer dependencies", it was easy to treat not adopting a mature implementation as the default.

The owner pointed out that collaboration also means handing part of our abstraction work to mature implementations. When managers fix defects, build features, or judge root causes, they should actively ask: does the industry already have a lightweight, mature library, or does a large project have a stable approach? If one might exist, search official documentation and project material without waiting for the owner to prompt. General problems such as query state, field contracts, and form descriptions should all go through this check.

When a candidate is found, the manager writes a short proposal: which piece of our own logic it can replace, how mature and well maintained it is, and its licence, size, and compatibility costs. The lead dispatcher then discusses it with the user. Once the user approves, it is integrated and verified along the normal execution line, and the duplicate implementation it replaces is removed rather than keeping two versions long-term. If the check concludes that existing capability is better, the specific reason is recorded too.

New dependencies still have limits: pin versions; prefer directly adopting implementations under permissive licences such as MIT or Apache-2.0, and only borrow ideas from GPL or LGPL projects; verify runtime compatibility and the packaging pipeline for embedded desktop pages. Adopting a mature library does not automatically authorize changing persistent data formats or breaking existing compatibility commitments.

The minimal fix for the current defect does not wait for this evaluation. Stopping the bleeding can happen first, and mature libraries are for governing the whole class; the user does not wait for a library discussion to get their operation back. This is now a working rule, but approving a dependency, a successful trial integration, and all entry points formally consuming it are different stages; picking a library is not the same as finishing the remediation. The evaluation contents and decision boundaries are in the [mature-library proposal notes](../../skills/reliable-agent-dispatch/references/mature-libraries.md) (Chinese).

## 12. One defect: fix the immediate operation and close off the whole class

The owner had long wanted to fix a whole class of problems whenever one defect was found. With few people and slow development, changing a whole area would hold up the immediate problem, so usually only the single point got fixed. Now that dozens of agents can work in parallel, he wanted to bring class-wide fixing back; the first large refactoring round showed him this organization could work.

We first looked for recurring sources in the history of fixes and formed seven classes: parameter source, instance identity and preserved deltas, field contracts, query state, coordinate spaces, form descriptions, and scene facts. The classification helps find entry points with the same pattern. It does not mean every problem can be classified mechanically, nor that all classes have been remediated.

A rough split of the author's historical fix titles puts changes related to preserving user edits at about 34%, parameter source and override order at about 19%, and field propagation at about 13%. These approximations are for finding high-frequency patterns. Categories can overlap, and a time window of titles is not the same as independent defects, so the numbers cannot be summed into a share of failures, let alone promise that fixing a class will remove that proportion of problems.

The new practice is to consider two lines for every clear defect. The **stop-the-bleeding line** keeps doing the minimal fix with end-to-end verification and follows the normal release; the user does not wait for the class conclusion. The **class remediation line** determines the class, inventories other entry points with the same pattern, looks for mature approaches, adds the class's static ratchet, and merges batch by batch as each matures. Several defects of the same class join the existing remediation line, so a new feedback item does not spawn yet another group.

The two lines do not wait on each other, and two executors still never modify the same file at the same time. Managers make write ownership and dependencies explicit; a shared root cause is handled by its owner, and everything that can proceed independently continues. A new class is first piloted on a real scenario, then rolled out as authorized, and closed by the inventory-and-ratchet conditions of the previous section.

A few problems get only a single-point fix: pure copy, styling, and layout; customer data configuration or environment problems; and isolated logic errors where investigation really found no second occurrence of the pattern. The last case must state which entry points were checked and why none matched; skipping classification on instinct is not allowed. Proposals for new features or changes to existing product behaviour cannot be implemented automatically under "class remediation" either; the user still decides whether to adopt them.

Reports therefore answer two questions: "Is the user's immediate problem fixed?" and "Is this class of old entry points closed off?" Passing the first does not end the second, and closing the inventory for the second does not replace real user acceptance. The manager checks that the delivery matches the original problem; the lead dispatcher reviews at key points and explains to the user. Source, build, real operation, integration, release, and user acceptance are still reported separately.

This round connected grouping, integration, rollout end points, and mature implementations, but organizational rules do not create execution capability by themselves. Messaging, queues, recovery, machine operations, and automatic wake-ups are still provided by the host. Each time you switch tools, first verify with a small batch of real tasks that deliveries reach a person, the next step can start, and status does not pretend to be a result; only then scale up.
