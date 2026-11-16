# Put Rules into Scripts; Let Machines Recover Locally: Multi-Agent Dispatch, Third Version

The second version turned managers into shared services and gave each problem to one executor through completion. After another five days, the goal changed: from working while the user watched to continuous operation while the user slept.

As we worked toward that goal, we found something more uncomfortable: most of what slowed us down was not product defects, but rules we had added ourselves.

This article covers four things:

1. Where the bottlenecks came from.
2. How to turn rules into scripts.
3. How to allocate machines without wasting them.
4. How to let the system recover when things go wrong.

## Conclusions first

- **Restrictive rules must trace to the user's words.** Remove gates that neither trace to those words nor protect a hard boundary—release, cloud deployment, customer or public data, credentials, permissions, permanent deletion, or shutdown—or make them diagnostic only. A new gate needs separate user approval, just like a new blocking check in the product.
- **Put rules that can be checked into scripts; do not rely on memory.** Written instructions trigger inconsistently in long conversations. Command checks either reject incomplete input or flag missing work. Much of the managers' work has been replaced this way too.
- **Admit work based on live machine pressure, not reservation arithmetic.** Start the task, then correct its reservation from measurements. Under real overload, return the most recently admitted work to the pool.
- **We cannot stop a model from writing a bad script, but we can improve recovery.** Each machine's local watchdog should stop runaway processes without depending on a remote connection or a person.
- **Record every source of friction.** Needing the user, waiting with work available, or repeating a manual step all count. Whoever encounters it records a row that turn, then looks for a script that can remove it.

## 1. Where the bottlenecks came from

The user said there seemed to be bottlenecks everywhere, more than before the restrictions, and that many rules had been added by the agents themselves through excessive defensive thinking rather than requested by the user.

We classified a little over a day's conversation records: 117 lanes and 3,383 messages.

| Bottleneck | Lanes | Messages |
| --- | ---: | ---: |
| Taking turns with test accounts: one account exclusively per session, then a named handoff to the next user | 63 | 724 |
| Waiting for builds | 65 | 660 |
| Queuing for machine reservations | 65 | 432 |
| Authorization chains, old queues, and registration receipts | 73 | 277 |
| Review and rechecking, including independent review for a two-line status-metadata change | 48 | 238 |
| UI locks and foreground checks | 29 | 131 |
| **Real product defects—the actual work** | 66 | 322 |

Real product defects ranked only fourth. The largest category, exclusive test-account ownership, was invented by agents: nobody had tried logging into the same account from two places, so they assumed it could not work and created a turn-taking and handoff process. When we finally tried it, the same test account worked on two machines simultaneously without interference. We removed the rule that day.

Another example came from the AI writing the rules. The user had asked for more work on the small Mac, but this became a ban on builds and tests on the main Mac. The user corrected it: the request had been to use all four machines, not to exclude the main one.

### Removing gates

We asked two questions about each gate: does it trace to the user's actual words, and does it protect a hard boundary? If neither applied, we removed it or made it diagnostic only without asking about each one. Unclear cases would go to the user, one per line with a recommendation.

We removed 12 gates, downgraded 3, and kept 16. None needed a further user decision.

### The lesson

- A request for more work on one machine can easily become a ban on work elsewhere. Restrictive rules must trace to what the user actually said.
- Adding a gate costs an agent almost nothing; removing it needs someone to investigate. New gates therefore require user approval, and existing gates should periodically face the same two questions.

## 2. Put rules into scripts

Most second-version rules were prose, with execution depending on agents remembering them. Those details are among the first to disappear during conversation compaction. Now anything checkable belongs in the command itself.

- **Reject missing information.** A state-changing command refuses incomplete fields: Needs your decision requires a recommendation, Done requires an outcome, and a released fix requires a release receipt containing that version.
- **Flag omitted work.** Scheduled checks exit unsuccessfully when a notification is more than 5 minutes late, hard approval has no answer, or a lane waits while a machine is idle. Lead dispatch sees that result when it wakes.
- **When the written rule and its script conflict, the script is authoritative.** Change the script and tests first when changing the rule.

Scripts have also taken over managers' duties:

| Previously done by a manager | Now |
| --- | --- |
| Identify work to start and whom to resume | A work-pickup script lists all ready items grouped by original owner, each with an assignment containing only the user's words, original-material locations, and the item identifier. The dispatch assistant passes it on unchanged, without rewriting or paraphrasing |
| Watch for stalled work | Every 15 minutes, a stall script incrementally reads session records and identifies promised next steps with no follow-up for 2 hours, 3 consecutive failures, multiple lanes blocked on the same class of issue, or performance work with no measurements for 4 hours. Patrol reads only this report hourly and judges the cause and who should act |
| Allocate machines | An allocation script, described in section 3 |
| Handle known faults | A recovery runbook, described in section 4 |

Only one management layer remains: a dispatch assistant. It does not talk to the user, review changes, or read execution logs. Lead dispatch remains the user's sole contact.

Before adding a new role, ask whether its work can be scripted.

## 3. How we allocate machines

We have four machines: a main Mac, a small Mac with 16G of memory, a Windows desktop, and a Windows laptop. The Windows machines are also the user's everyday computers.

### The small Mac was supposedly always short of memory, yet mostly idle

One night's records showed:

- Of 25 requests entering the waitlist, 24 requested Mac. Many were Rust tests, frontend builds, or Python unit tests that did not depend on Mac; requesting it was simply a habit.
- The small Mac had normal memory pressure, 9–11G actually available, and CPU use mostly around 1%. Yet reservation accounting showed only about 2G remaining, so even three small tests requesting 0.25–0.5G could not start.
- Reservations deducted estimated peaks even when unused. In measurements, SketchUp validation reserved 3G but peaked at 1.71G across 249 samples; Python tests reserved 0.25G but used 0.02–0.05G; frontend builds reserved 1G but used 0.03–0.28G. Rust builds peaked at 7–8G, but only briefly during linking.

### A queue deadlock

One night, the small Mac had about eighty percent of its memory free while 14 items waited. Three things combined:

1. An agent-created rule held a whole machine free for a large task after it had waited more than 20 minutes.
2. That task requested 9G, although the measured p90 for similar work was about 2.6G. The small Mac had only 16G and a 7G release reservation that actually used less than 1G and would not shrink. The task could never fit, and nobody else could enter.
3. After the dispatch session restarted, wait entries from the old session were awakened every 15 minutes. Nobody claimed them, and they never left the list.

### The current rules

The user proposed a greedy approach: start the work, and if the machine genuinely suffers, return that task to the pool. Until it runs, we cannot know. The user also pointed out that macOS compresses memory and that more than a dozen SketchUp sessions had run without lag, so routine queuing was unnecessary.

- **Admit on live pressure.** On Mac, admit while memory pressure is not critical: macOS can compress memory, and a program using more while the machine is idle does not prove contention. On Windows, admit below 95% committed memory. Commitment has a hard limit; allocations can fail when it is exhausted, so that boundary stays.
- **Normally, do not queue.** Allocate immediately if any online machine qualifies. Queue only when all four cross the thresholds. Do not hold a whole machine for a waiting item.
- **Correct reservations from measurements.** After 10 minutes, use 1.3 times the peak over the latest 10 minutes, with a 0.5G minimum, adjusting upward too. Apply the same correction to release reservations, retaining only their priority in ordering and return-to-pool decisions.
- **Return work only after sustained severe overload.** If Mac pressure is critical or Windows commitment reaches at least 97% for 2 minutes, pause background work first, then return one most recently admitted ordinary build or test at a time, preserving its original queue time. Return SketchUp validation last and save its scene first.
- **Request any platform for platform-independent work.** If the small Mac cannot take it, use Windows or the main Mac.
- **Remove entries unclaimed after two wake-ups.** A stopped session should not keep occupying a place.

There is a counterpart in [Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/): scheduling by requests, with limits and [node-pressure eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/). We recognized the connection only after these mistakes.

## 4. When things go wrong, recover on the machine

After a PowerShell process made a machine unusable, the user put it this way: we cannot prevent a model from causing these failures, but we can improve our own recovery.

### What happened

- A PowerShell script started remotely over SSH on the laptop grew to 56.6G of private memory and pushed commitment to 100%. CPU use was only 7%, yet the machine could not accept any new task for about 1 hour and 40 minutes.
- A few minutes after stopping it, the desktop started another similar process, which reached 11.3G in 8 minutes.
- Looking back over 30 days, the desktop had run out of memory 8 times, all from our own PowerShell processes, each consuming 40–53G.

### The recovery runbook

Scripts fix known faults directly as part of the machine check every 15 minutes. Current entries include:

- Windows logon-session leaks or memory pressure → restart automatically. Wait for our running builds or validation to finish, do not restart machines registered as in use by the user, and show an on-screen warning 5 minutes beforehand.
- Reused remote connection fails → close the old connection and reopen at the next sample.
- Expired UI lock is not returned → reclaim it.
- Scheduled-check results are stale → rerun the scheduled task.
- A PowerShell process we started remotely exceeds 8G of private memory → verify that it is still the same process, stop it, and notify the owner. Do not restart the machine in this pass.

Each repair must say how to recognize the fault, how to repair it, and how to verify recovery. If the same problem survives 3 repairs within 6 hours, escalate to lead dispatch to investigate the cause.

Lead dispatch or patrol can add a rule for a newly diagnosed fault. A new rule records observations without acting for 24 hours, then replays both incident and normal states. It must match the incident and avoid matching normal operation before activation. Validation rejects deletion, shutdown or logout, stopping others' processes, external writes, credentials, and privilege escalation.

### Remote recovery is not enough: use a local watchdog

A remote check every 15 minutes has two gaps:

- **It is too slow.** A process growing by 1.4G per minute can fill a 32G machine in little more than ten minutes.
- **It may be unavailable when needed most.** At full commitment, even a new SSH session may fail to start.

The next step is a resident local watchdog on each machine, checking every 15 seconds and recognizing only our processes: those started through SSH, our scheduled tasks, or the allocation script's registry. It stops them locally when they exceed a limit and never touches the user's applications. Windows also provides [Job Objects](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects), which can apply memory limits to a whole process tree through the operating system. This was still being built when this section was written; there were no results yet.

## 5. The friction ledger

Continuous unattended operation gives friction a concrete definition:

- The user has to intervene.
- Work is available but waits idle for more than 15 minutes.
- The same manual step happens a second time.
- A process invalidates existing output.

Whoever encounters it records a row that turn: symptom, cost, cause, how to remove it, and status. Prefer a script check or automatic handling, leaving only a pointer in the written rules. If it recurs, change the existing status back to Partial rather than adding a duplicate row.

Almost all the changes above came from this ledger.

## 6. Results and what remains unverified

- Gate removal, allocation, and recovery rules were introduced only in the past few days. At writing, we did not yet have a full day's comparison. After a day, we plan to recount bottlenecks using the same categories and measure concurrent tasks, average queue time, and returns caused by overload.
- One idle-capacity survey found average CPU use of only 7%–22% over more than 6 hours across four machines. The machines were far from full; more work was blocked elsewhere. We therefore want to fix rules and allocation before deciding whether to add machines.
- All figures come from local conversation and resource records. They are observations, not a controlled experiment.

## 7. What changed from the second version

| Second version, November 13 | Now |
| --- | --- |
| Rules were mostly prose that agents had to remember | Put checkable rules into commands: reject omissions and flag undone work; the script prevails over conflicting prose |
| Lane coordinators pick up work, resume threads, and relay decisions | Dispatch assistants follow generated assignments; scripts count stalls and patrol judges them; only one management layer remains |
| Scripts read live load but deduct reservations from capacity | Admit on live pressure without routine queues, correct reservations from measurements, and return work only under sustained severe overload |
| No dedicated self-healing | Observe new recovery rules before activation; stop runaway processes locally |
| Add gates when thought necessary | New gates require user approval; remove those without a basis in the user's words or a hard boundary |
| Act after 15 minutes stalled on a critical path | Retain this and extend it into the friction ledger: record intervention, idle waiting, repeated manual steps, and invalidated output |

## Appendix: before adding a gate

```text
What does this gate block:
Which exact user statement supports it; quote it:
Which hard boundary does it protect: release / cloud deployment / customer or public data / credentials / permissions / permanent deletion / shutdown or logout:
Cost of false blocking: how many lanes wait and for how long:
Could this be diagnostic only, without blocking:
User approval record:
```

If the first two lines have no answer, do not add it.
