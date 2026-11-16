# Track security, class, plan and issuer state as a dependency graph and propagate errors along it

Revised body for Open-Cap-Table-Coalition/OCF-Tools issue 225. Everything below the line is the issue text, ready to copy.

---

Proposed as the Phase 4 tracker on the roadmap (#145), superseding #164. Adopts the reducer mechanism from discussion #197 and widens the goal. Relates to #186, #198, #214 and #224.

@MattCantor this reframes #164 and #197, both yours, so it needs your read before anything moves. If you agree with the direction, the housekeeping I'd propose is: close #164 as superseded by this issue, resolve #197 with a reply pointing here, and rename the Phase 4 milestone to drop "XState actors", since #197 and this issue both land on a reducer. None of that is done yet. It waits on your review.

**Revised 2026-11-16.** What changed since the first version:

- Ratification: a new edge type, a cured outcome and a section of its own.
- Two new rules: a failed issuance still creates its security, and a valid absolute write clears taint.
- A removed security stays in the map, and every field records its last writer.
- A picture of the graph over time.
- More detail on where XState sits.
- Open decision 2 is narrowed and three are added.
- A section on what this design keeps open for validating a live event stream later. Streaming itself is out of scope.
- "Today" now counts what is validated and what isn't, and "Sequencing" splits the phase into three cuts as a result.

## Summary

The validator enforces a real dependency structure between transactions, but nothing writes it down. Each check re-derives what it needs by scanning the transaction log, and the run halts at the first failure. Halting is the crudest propagation rule there is: it propagates along time, not along dependencies, so an error in one security's chain stops validation of every unrelated security dated after it.

This issue proposes making the structure explicit: a small set of stateful nodes (securities, stock classes, stock plans, the issuer), transactions as reads and writes on those nodes, and a taint that propagates along the edges. A failing transaction taints what it writes. A check that reads a tainted node is blocked, with a pointer to the root cause, and the rest of the package keeps validating.

Some failures are later cured by a ratification. A failing check that a ratification covers is recorded as cured: it taints nothing, and the checks downstream of it run.

## Today

- The machine runs in date order and enters the final `validationError` state on the first failing transaction. Snapshots stop at the last day validated in full.
- #198 made the offender scans look only backward, and #214 added `no-earlier-removal` on top of a `removals` index. Both are special cases of the graph. A removal is one edge type, and the index holds one security per transaction, which is why #224 left consolidation's `security_ids` and plan-security removals out.
- Checks re-derive what they need by scanning. The legacy stock validators loop over `context.stockIssuances`, and two places in the shared checks in `validators/checks.ts` still scan `ocfPackageContent.transactions`. That is #164's complaint, now narrower than when it was filed.
- Of the 45 transaction types, 20 return findings (convertible, warrant, equity compensation), 8 are legacy (stock), and 17 are passthrough: not validated and not applied. The passthrough types are most of what writes to a class, a plan or the issuer: every adjustment, split, consolidation, return to pool, plan securities and vesting.
- No aggregate check exists. The authorized-shares check is listed under "MISSING CHECKS" in `tx_stock_issuance.ts`, and nothing tracks outstanding, reserved or available.
- Same-day order comes from a fixed ranking of transaction types (the `transactionTypes.indexOf` tiebreaker in `ocf_validator/index.ts`), not from what each transaction depends on.
- The roadmap's goal of reporting all findings at once instead of bailing (#145, #186) has no answer yet for how to keep one root error from spraying cascading findings through everything downstream of it.
- A package has no way to say that a defect was ratified, so an over-issuance that was cured reads the same as one that wasn't.

## The model

### Nodes

Four node kinds carry state and can be tainted.

| Node | State | Written by |
|---|---|---|
| Security | lifecycle (per #197), quantity and exercised tally, vesting state, provenance | its family's transactions, vesting transactions, consolidation, split |
| Stock class | authorized, outstanding, conversion ratio | every issuance and removal in the class, split, both class adjustments |
| Stock plan | reserved, available | pool adjustment, plan security issuances, return to pool |
| Issuer | authorized | issuer adjustment |

The security row is close to what the code tracks today. The class, plan and issuer rows are not: most of their writers are passthrough, so that state and the checks on it are new work (see Sequencing).

State and taint are per field. A class whose `outstanding` is tainted still has a trustworthy `authorized` and conversion ratio. Each field holds its value, the id of the transaction that last wrote it, and its taint if any.

A removed security stays in the map with its lifecycle set to removed. Return to pool reads a removed security, and a later transaction that names one should be told it was removed and by what, not that it never existed.

Stakeholders, vesting terms and stock legends are reference data. They don't change over the run, but they can be invalid. They're validated once before the run, and an invalid one taints what depends on it: a broken vesting terms object taints the vesting state of every security that uses it. Without this pre-pass, a bad reference object is a hole no transaction-level check can catch.

Transactions aren't nodes. Two of the transaction-to-transaction references (`split_transaction_id` on reissuance and on return to pool) are covered by what the split writes. The third is ratification, which names the transactions it cures and is described below. The transaction graph is derived rather than stored: every field carries the id of its last writer, and following those pointers is the graph. That holds for untainted fields too, so the same pointers answer "why is this blocked" and "where did this number come from".

### The graph over time

Unrolled along the date axis, each node is a thread and each write is a new point on it. A transaction is a vertical connector joining the threads it reads and writes on one date.

```
           day 1        day 2         day 3           day 4
START ●
 ├ issuer ─────────────────────────────────────────────────▶
 ├ class A ──┬────────────────────────┬──────────────┬─────▶
 ├ plan ─────┼────────────────────────┼──────────────┼─────▶
 │           │                        │              │
 └ sec-1     ◉ issue ───◉ accept ─────╳ transfer     │
                                      ├◉ sec-2 ──────╳ cancel
                                      └◉ sec-3 ────────────▶
           ┆            ┆             ┆              ┆
          snapshot     snapshot      snapshot       snapshot
```

- The single starting point is `START`, where the package is loaded and the reference data is checked.
- Class, plan and issuer threads never end. Security threads end at a removal.
- Most removals are branch points: a transfer ends one thread and starts its resulting and balance securities. Only retraction, and cancellation or repurchase with no balance, end a thread with nothing after it.
- A vertical cut at a day boundary is that day's snapshot.
- Order inside a day is not given by the data. It is computed from the edges (rule 5).
- Taint flows to the right along a thread and crosses to other threads at each connector. It never flows left.

### Edges

Every transaction reads some nodes and writes some. From the reference fields on each transaction type in the OCF schema, grouped by shape:

- **Issuances** read stakeholder, class, plan, vesting terms and issuer. They write a new security, class outstanding and plan available.
- **Lifecycle transactions** (acceptance, retraction, repricing, vesting start, event and acceleration) read and write one security.
- **Removals with a balance** (cancellation, repurchase) read one security. They write it removed, the balance security and class outstanding.
- **Removals with resulting securities** (transfer, conversion, exercise, release, reissuance) read the source security and the resulting issuances. They write the source removed, provenance on the resulting securities, the balance, and outstanding on the source and destination classes.
- **Consolidation** reads many securities and writes one.
- **Split** reads a class and writes its ratio plus every security in it. One failure fans out to the whole class.
- **Class, plan and issuer adjustments** read and write their own node. Class authorized also reads the issuer.
- **Return to pool** reads a removed security and writes plan available.
- **Ratification** names earlier transactions and writes no node. It changes the outcome of their ratifiable checks from failed to cured.

### Rules

1. **Checks declare what they read, per field.** A check whose read is tainted comes back blocked, carrying the root cause, and the transaction's other checks still run. This is the decision that makes class and plan taint tolerable. Most cap tables have one or two classes, so if a tainted class blocked whole transactions, one bad issuance would block every later issuance in the class, and for the common case that is halt-on-first-error with extra steps. Blocking only the authorized-shares check keeps stakeholder, lifecycle and vesting checks running on everything else.
2. **A failed or blocked transaction taints everything it writes.** Over-tainting is safe, since a blocked check is recoverable once the root is fixed. Under-tainting is a hole, since a false pass isn't. Start conservative and narrow specific write edges only where real packages show them to be noisy.
3. **A failed issuance still creates its security.** The node exists, tainted, with its values unknown. Otherwise a later acceptance of that security fails with "no such security", which is a second root error for one mistake. With the node in place the acceptance is blocked and points at the issuance.
4. **A valid absolute write clears taint on the field it sets.** The authorized-shares adjustments, the pool adjustment and the conversion ratio adjustment each set a new value without reading the old one, so a valid one makes that field trustworthy again from that point on. Fields that accumulate (class outstanding, plan available) are never cleared this way. Without this rule, one early bad write to `authorized` blocks every authorized-shares check for the rest of the run, even after a clean adjustment replaces the value.
5. **Aggregate checks run at end of day over the net.** A same-day transfer momentarily over-issues the class if its resulting issuance is processed before the source removal. Authorized-shares and pool checks instead run against the end-of-day state the machine already builds for snapshots. Same-day order then matters only for lifecycle dependencies, and the resulting-security edges give that order explicitly, replacing the transaction-type ranking used today. Transactions on the same day with no edge between them are ordered by id, so the result never depends on the order of the input. A cycle among the edges is a root error of its own.
6. **A failing check covered by a ratification is cured.** Its writes are applied, nothing is tainted, and downstream checks run as if it had passed. The finding stays in the output, marked cured, with the id of the ratifying transaction.

### Ratification

The board, the stockholders or both can ratify an act that was not validly authorized when it happened. There are legal limits on what can be ratified, and they vary by jurisdiction and by the kind of defect.

**It is not a healing write.** Rule 4 clears taint from the write forward. A ratification relates back: the act is treated as valid from its own date. The cure lands at the root, on a transaction the run has already passed. So ratifications are indexed at `START`, before the run, the way `removals` is built today. When a check fails, the machine looks the transaction up in that index.

**Whether a check can be cured is a property of the check.** Each check declares whether it is ratifiable, and by whom (board, stockholders, both). The dividing line:

- **Authority defects.** The record is coherent but the act lacked approval or capacity, such as an issuance over the authorized share count or over the plan pool. Ratifiable.
- **Record defects.** The data itself is incoherent, such as a transfer of a security that was already cancelled, or a reference to a stakeholder that doesn't exist. No vote cures these, and a ratification that names such a transaction changes nothing.

**A ratification can name either side of the defect.**

- *The defective acts*, such as the issuances made over the authorized count. The end-of-day aggregate check then has to tie its failure to specific transactions, which is open decision 6.
- *The authorizing act that should have preceded them*, such as an authorized-shares adjustment. That adjustment is recorded at the date it takes effect, the ratification cures its late approval, and the aggregate check never fails. This case needs nothing beyond rule 4.

**What the validator checks.** That the ratification names transactions that exist, that the approving body matches what the cured check requires, and that it is dated on or after the act. It does not decide whether the ratification is legally effective. The outcome means "the package claims this was cured", not "this is valid".

**OCF has no ratification object.** Approval exists only as `board_approval_date` and `stockholder_approval_date` fields on the approved object. An enhancement request for a `TX_RATIFICATION` transaction is being raised on Open-Cap-Format-OCF (link to follow). How the validator receives ratifications before that lands is open decision 5.

## What changes in the output

- `findings` gains a blocked outcome: a check that couldn't run because of a tainted read, carrying the check id, the subject and the root transaction.
- `findings` gains a cured outcome: a check that failed and is covered by a ratification, carrying the id of the ratifying transaction.
- `result` separates root errors from their consequences and from cured defects: "1 error, 14 checks blocked by it, 2 cured" rather than "17 errors". Users fix roots, not consequences, and fixing a root either clears its blocked checks or exposes the next root.
- Snapshots carry the taint. A day's snapshot is trustworthy for untainted entries and says which ones aren't.
- Snapshots carry the cure. An entry dated between a defective act and its ratification is marked "defective until this date, ratified by this transaction", so one snapshot answers both "what was known then" and "what is true now".
- The run no longer halts mid-package. `validationError` stops being a final state reached on the first failure.

## Where it goes in the code

- **Reads** are declared on each check object in `checkKit`, next to `id`, `severity`, `description` and `run`. The machine resolves them against the node map before calling `run`, and returns blocked instead if any is tainted.
- **Ratifiable** is declared in the same place, with the approving body the check requires.
- **Writes** go on the descriptor table, which already classifies each transaction's `effect` (`append`, `remove`, `none`, `passthrough`) and its collection. It grows to name the aggregates each transaction moves, and whether each write sets a value or adjusts one (rule 4).
- **State**: the context's four flat issuance arrays and the `removals` index become one node map keyed by id, as #197 proposed, with class, plan and issuer entries alongside securities. `remove` stops filtering the security out and sets its lifecycle instead. The map is immutable: a write produces a new map that shares everything it didn't change, so keeping one per day for snapshots is cheap.
- **Indexes built at `START`**: the ratification index, keyed by ratified transaction id. The `removals` index goes away, since "was this security removed earlier" becomes a lifecycle question.
- **Lifecycle**: the per-security state chart from #197, as a static machine definition used as a lookup. The node stores only the state value. "Is this transaction legal in this state" is the lifecycle check, and "what state follows" is the write. No actor is spawned per security. #197's argument against spawned actors stands: a transfer is one atomic write across several nodes, and separate actors would turn that into message ordering. On the pinned `5.18.2` this is `machine.resolveState`, `snapshot.can` and `getNextSnapshot`, all of which work there.
- **The outer machine** becomes a day cycle: order the day's transactions, apply them, run the end-of-day aggregate checks, take the snapshot. The `isValidTx` guard exists only to choose between `capTable` and `validationError`. Once nothing halts it goes away, and each validator runs once per transaction instead of twice (today the guard runs it, then `recordOutcome` runs it again).
- **Tests**: every path through the lifecycle chart is a valid single-security history, and every state and transaction pair the chart doesn't allow is an expected finding. Both can be generated from the chart (XState's path utilities are not exported at `5.18.2`, so this means a newer XState, `@xstate/graph`, or walking the definition directly), alongside hand-written packages for the cross-node cases (transfer, consolidation, split, ratification). Two tests guard the model itself. One records what each check actually reads and fails if it reads something it didn't declare, since an undeclared read is a check that won't be blocked when it should be. The other shuffles a package's transactions and expects identical findings and snapshots.

## Holes this closes that exist today

- Consolidation's `security_ids` and plan-security removals, both left out of #224's index.
- A failed split, which today halts everything, and under a per-security-only model would taint nothing.
- Invalid reference objects.
- The "two findings for one story" residual noted on #202: one root finding and one blocked marker per dependent, instead of overlapping errors.
- #186's unknown transaction types: an unknown type becomes a root finding, and if it names a `security_id` that security is tainted, since we don't know what the transaction did to it. The rest of the package keeps validating.
- Same-day order that depends on a ranking of transaction types and not on what each transaction references.
- A cured defect that reads the same as an uncured one.

## Kept open for streaming

Out of scope here, but the direction is known: validating a long-running stream of transactions that can arrive late, out of order or more than once, with no guarantee of when. This issue does not build that. It avoids choices that would have to be undone to get there.

What a stream needs, and what this design already gives it:

- **The log, ordered by transaction date, is the truth.** State is the reducer folded over the sorted log. A late transaction is inserted at its date and what follows is recomputed. That only works if the reducer is pure.
- **Checkpoints to recompute from.** The per-day snapshots are those checkpoints, because the node map is immutable.
- **Recomputing only what a late transaction touches.** The declared reads and writes say which nodes are affected. The edges that carry taint are the same edges that carry invalidation.
- **Results that don't depend on arrival order.** Rule 5 orders each day from the edges and breaks ties by id.

Ratification is already a small case of this: a later fact that changes the outcome of an earlier transaction. The index built at `START` is the batch form. In a stream it is a late transaction that invalidates from the date of the act it names.

Five constraints this issue holds to for that reason. All are cheap now and expensive to retrofit:

1. The reducer is pure, and no check scans forward in the log. #198 already went this way.
2. The node map is immutable.
3. A finding has a stable identity (check id and subject), so a later run can replace or withdraw it. `findings` stops being a list that only grows.
4. A check that fails because a referenced object is absent says so in a field a program can read, not only in its message. In a batch run that is still an error. In a stream it may only mean the object hasn't arrived.
5. A test shuffles a package's transactions and expects the same findings and snapshots.

What changes later, and is not decided here:

- Nothing is final in a stream with no latency guarantee. The most the validator can say is "valid as of what has been received". Closing a period needs something outside the stream, such as the issuer attesting that the record is complete through a date.
- A transaction received twice is a no-op. The same id with different content is a conflict to report.
- The actor question from #197 comes back one level up: one actor per cap table stream, owning the connection, the log and the checkpoints, with this reducer running inside it. That is a separate proposal.

## Open decisions

1. **Blocked and cured as more `Severity` values, or a separate field on `Finding`?** A separate field keeps `severity` meaning how bad the problem is and lets a blocked warning-level check stay a warning once unblocked. With cured added there are now three outcomes (failed, blocked, cured), which makes the case for a separate field stronger. The absent-reference marker from the streaming section belongs in the same place. Leaning separate field.
2. **Are a failed transaction's values applied or withheld?** Rule 3 settles the node: it always exists. What remains is whether a failed transaction's other writes show their new values or the old ones. Downstream checks are blocked either way; it only changes what the snapshot shows. Leaning withhold, with the taint marker telling the reader why an entry is stale.
3. **How far to narrow rule 2.** Start with "taint all writes" and revisit with real packages.
4. **Does a blocked check carry one root or all of them?** A class's `outstanding` can be tainted by three bad issuances. Carrying only the first is simpler and matches "fix a root, expose the next". Carrying the set makes the count in `result` honest. Leaning the set.
5. **How does the validator receive ratifications before OCF has an object for them?** A sidecar input, or a custom transaction type. A custom type collides with the #186 rule above, which would flag it as unknown and taint what it names, so that rule would need an exemption.
6. **How is an end-of-day aggregate failure tied to transactions?** The check fails on a class and a date, but a ratification names issuances. One rule: the failure is cured if the ratified issuances in that class on or before that date cover the excess. Needs a real over-issuance package to test against.

## Sequencing

The engine is about half the work. The other half is validator work the model assumes and the code doesn't have. So the phase splits into three cuts, each useful on its own.

**Cut 1: securities only.** The node map, removed securities kept, no halting, the blocked outcome, taint, declared reads, the lifecycle chart, dependency order within a day, the reference data pre-pass, and the streaming constraints. It runs on the checks that exist today.

What it delivers:

- One run reports every root error in a package, not only the first.
- Each consequence is marked blocked and points at its root.
- Snapshots for every day to the end of the package, with untrusted entries marked.
- Lifecycle rules from one chart, covering consolidation and plan-security removals.
- Unknown transaction types no longer stop the run.

This cut may not need to wait on the stock family moving to findings (#159). It works on the 20 graded types as they are, and a failed legacy stock validator could be wrapped as one error finding that taints its security. That wrapping is untested.

**Cut 2: aggregates.** Class, plan and issuer state, behaviour for the passthrough types that write it, the authorized-shares and pool checks at end of day, rule 4, and the quantity arithmetic for splits and partial exercises. This is the largest cut, and most of it is new validator behaviour, not graph machinery. It needs #159 done, since stock is the family with class aggregates.

**Cut 3: ratification.** Small in code. It needs cut 2, because the ratifiable checks are the aggregate ones, and it needs open decision 5 answered.

Outside all three:

- Vesting state is read by equity compensation exercise and release. #165 (vestlang) is where that state gets computed; until then the graph treats it as reference data.
- Streaming, beyond the constraints cut 1 holds to.
