# The Ledger Must Match Reality: Multi-Agent Dispatch, Fourth Generation

The third generation concluded: "any rule that can be a script should be a script." We followed that for three days, and scripts did take over a lot of the work. But three new problems showed up:

1. The ledger inside the scripts drifted from what had actually happened, and the statistics started lying.
2. Gates moved from written rules into code, where they were harder to see, and what they blocked were normal cases.
3. Once releases ran through a pipeline, they cost four times as many tokens as manual releases.

## The short version

- **The ledger must match reality.** "Shipped" means the code is in a release, not that someone recorded it as shipped. "Resumed" means the agent actually started a new turn, not that a message was sent.
- **One false block and the check goes.** If a check ever blocks a normal case, delete it or downgrade it to log-only. Don't wrap another layer of fixes around it. Only four release checks can't be skipped.
- **Scripts should replace agents, not give agents a script to watch.** One agent owns a release from integration to close-out; once the pipeline is running, it waits for the result instead of polling.
- **The patroller becomes a pusher.** When something is stuck, first ask "is there another way?" If there is, push it through on the spot; only if there isn't, ask a human.
- **The user has to be able to see the project.** Reconcile lanes every 15 minutes, reconcile the ledger every hour, and write a one-page plain-language status every hour.

## 1. The statistics were lying

The user asked an agent to audit how many fixes were done but not yet shipped. The ledger said 41 were waiting to ship. In reality, 27 of them had already gone out with official releases.

The cause was mundane:
- When a lane finishes a fix, it records its branch and commit hash.
- At release time, the patch is re-applied into the integration source, which changes the commit hash.
- At release close-out, nobody went back to mark those entries as shipped.

So the backlog kept growing and looked like "lots of fixes, few releases", and the team started pushing in the wrong direction.

**Now we judge by the code:**
1. If the fix commit is an ancestor of a release, and the change still reverse-applies on that release (it wasn't reverted along the way), it counts as shipped.
2. Otherwise, take the product-code lines the lane added (tests and docs excluded) and look for them in the same files of the release source. 90% or more found: shipped. 50%–90%: the original owner checks. Below 50%: still waiting.
3. Backtest against those 41 entries, each checked by hand: **none of the 10 unshipped fixes was misjudged as shipped.** Of the 27 shipped ones, 23 were recognized automatically, 3 were flagged "probably shipped" for a human to check, and 1 was missed (it stays "waiting to ship", so nothing gets closed by mistake).
4. Entries that are definitely shipped get marked automatically by the scheduled check, without bothering anyone.

The same problem has other forms. "Resumed the worker" used to count as soon as the message went out; now it only counts once the worker has actually started a new execution turn.

> A ledger records what someone said happened. Reality is code, processes and sessions. Once the two drift apart, the statistics lie, and the team works against fake numbers.

## 2. Gates moved from text into code

The third generation said: any restrictive rule must trace back to something the user actually said. But once rules became scripts, gates grew somewhere else.

**The release glue layer.**
- The user asked for "something lightweight, just stitch it together": chain the existing release scripts.
- What got written was about 3,300 lines in two days, with a check around every step: identity, receipts, bindings, sealing eligibility, signing closure.
- Around the freeze, it blocked one release four times in a row, every time on a normal case: preflight was all green but got voided for "source or environment changed"; it couldn't read a build receipt from another machine; a home-made transfer step timed out; a manifest contained Mac paths, so the encryption tool on Windows couldn't find the files.
- That same night, the old one-click script never failed once.

**We did it too.** The AI assistant that writes these rules once added, on its own, "the main Mac's screen belongs to the user; don't run live tests on it." The user never said that. The main Mac sat idle while all live tests queued on one small Mac.

**The rules now:**
- **One false block and the check goes.** Any check that has blocked a normal case (path formats differing between platforms, a directory that moved, a rerun in a new folder) gets deleted or downgraded to log-only. No patch layer on top.
- **Only four release checks can't be skipped:**
  1. The release package comes from the frozen source, and its identity and hashes match;
  2. This release's changes directly cause data loss, privilege overreach or wrong production output (if that item can be pulled out of the release, pull it);
  3. The public release must read back correctly;
  4. Credentials stay safe.

  Every other identity, receipt, binding or provenance check is handled by the previous rule.
- **New rejection points must be registered.** Any new `raise`, error return or whole-batch drop in a lane needs a stated reason and a test for the normal case, or it doesn't get merged. This gate itself was approved by the user.

## 3. The pipeline made releases more expensive

| | Manual releases (6 in 4 days) | Pipeline releases (3) |
|---|---|---|
| Tokens per release | ~0.6B | ~2.5B |
| Release-related calls | 2,000–4,000 a day | 10,000+ a day |

The cost per call didn't change; it's roughly the size of the context. What grew was **the number of calls**.
- Around the pipeline hung five or six standing agents: integrator, release owner, preflight, Windows pre-freeze, entry-point auditor, bundle owner.
- Every time one of them woke up, it re-read its own long context.
- One of those owners made over 7,000 calls in three days: 1,291 of them sending messages, 929 waiting.

The script didn't replace the agents. It gave them an extra job: watching the script.

**What we do now:**
- **One release, one agent.** Integration, build, preflight, smoke test, freeze, encryption, publishing, read-back and close-out are all the same executor.
- **Parallelism belongs to pipeline processes, not agents.** The pipeline runs on several machines at once; the executor starts it and waits for the result. On failure it reads the raw output itself: tool problems get fixed on the spot and rerun, product defects become a new item for someone else.
- **Nothing gets added after the freeze.** Once integration starts, the scope is fixed. Anything finished later goes into the next release.
- **Ship whenever there's an all-green candidate; no fixed release slots.** Once frozen, it must ship, whatever the hour.
- **Releases must keep up with fixes.** If fewer than 70% of the fixes from the last 24 hours were shipped, raise an alert.
- **A major production incident gets a hotfix.** Take the current public release, add only that fix, and every batch release in flight waits its turn.
- **Alert when release tokens exceed budget.** The budget is the manual-release average, about 0.6B per release.

## 4. The patroller becomes a pusher

The old patroller did one thing: read the report and pass it on. Some examples:
- A Windows machine's C: drive filled up, and automatic cleanup ran three times without freeing enough space. The D: drive had 100 GB free, and nobody thought to switch drives.
- One release slipped from "ship this first" in the afternoon to the next morning. Every lane was busy the whole time, so the stall detector never fired once.

Both were eventually spotted by the user.

**As a pusher:**
- Every hour it reads the stall report, goal progress and the last few messages from any lane that was named.
- When a lane looks stuck (it announced its next step and then went silent for two hours, or it's been busy for two hours without reaching the next step, or its estimate slipped twice), it first asks itself "is there another way?" If so, it pushes on the spot: wake the original executor and hand it the new approach, switch machine or entry point, or pull that item out of the release.
- Lanes already switching approach and making progress are left alone.
- Only when there's truly no other way does it go to the main dispatcher to ask the user.
- A fix that worked twice goes into the self-healing playbook, so next time the script handles it.

## 5. The user has to be able to see the project

What the user said, roughly: besides polishing the production system, I want to stay closer to the project itself.

So the scheduled tasks were split into three cadences:

| Cadence | One job only |
|---|---|
| Every 15 minutes | Reconcile lanes: dispatch everything that can start; the original executor resumes, and only if it can't be woken is a new one started |
| Every hour | Reconcile the ledger: compare every entry's state with reality and correct mismatches on the spot |
| Every hour | Write a one-page status for the user |

The status page has four parts:
1. **This hour:** what got fixed and what got shipped, each as "problem → result".
2. **In progress:** grouped by area, with counts, plus the few most important items and "where it stands now".
3. **Stuck:** for anything waiting on the user, exactly what the user needs to do; for anything stuck elsewhere, who it's waiting on and for what.
4. **Next release:** how many items are waiting, when it's expected, and what's holding it.

Every number is computed item by item, and no internal codenames are used.

## 6. Not done yet (and just fixed)

**Too many alerts; real problems got buried.** While writing this, one scheduled check raised 74 alerts:
- About 50 were "a notification to the user hasn't been delivered yet", mostly records of the watchdog killing a task, which the user doesn't need to see one by one;
- The ones that really needed someone to act (idle machines, several lanes stuck on the same kind of problem, a disk nearly full) were only a dozen or so.

"Alert if undelivered" was meant to keep notifications from getting lost; it had turned into noise. We fixed it the same day: record-only notifications meant for the original owner no longer raise alerts and are only counted; notifications of the same kind are merged into one line; and alerts that need action come first. On the same data, alerts went from 74 to 26.

**Lots of detectors, few fixers.** About ten detectors can only raise alerts, while the self-healing playbook has just two rules, both still in observation. Going from "raise an alert" to "the script fixes it" is where this system actually saves people. Auto-marking shipped fixes, auto-cleaning full disks and auto-restarting Windows machines that leak login sessions were all built that way; the frequent alerts need to be converted the same way, one kind at a time.

**Two ledgers haven't been merged.** The old dispatch queue and the new state machine both still exist, and reconciliation has to read both.

## 7. This week's numbers

| Item | Number |
|---|---|
| Period | 7 days |
| Public releases | 11, with a three-day gap followed by five releases in two days |
| Shipped fixes | 109 |
| Fixed, waiting to ship | 20 |
| Answered "not a defect" / won't do / merged into another item | 106 / 57 / 64 |
| Waiting on a user decision | 0 |
| Blocked on the user | 1 |
| Average CPU use (four machines) | 10%–22% |
| Machine requests granted immediately | 97% |

The last two rows say it: **the bottleneck is process and releasing, not compute.** So we're not buying machines for now; the plan is a fast external drive for the main Mac to hold lane scratch files.

## 8. Compared with the third generation

| Third generation | Fourth generation |
|---|---|
| Rules become scripts | The scripts' ledger must match reality (code, processes, sessions), not bookkeeping |
| Restrictive rules must trace to the user's words | Plus: one false block and the check goes. Only four release checks can't be skipped |
| Releases go through a pipeline | One agent per release; pipeline processes run in parallel; no agent polling; nothing added after freeze; ship whenever all green |
| Patroller judges hourly and advises | Pusher: if there's another way, push it through on the spot |
| User reads the progress board | Lanes every 15 minutes, ledger every hour, a one-page plain-language status every hour |

## Appendix: before you write a check script

1. What real incident does it block? Has that incident been reproduced?
2. Will it block normal cases? Try paths on both platforms, a moved directory and a rerun.
3. After it raises an alert, who acts? Could that action be written straight into the script?
4. If it blocks a normal case once, are you willing to delete it?

If you can't answer the first two, don't add it. If you can answer the third, write a fix script, not an alert.
