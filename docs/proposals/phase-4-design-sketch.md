# Phase 4 design sketch: the graph layer and the reducer

Companion to `phase-4-dependency-graph.md`. That file says what and why. This one sketches how, in pseudocode, and maps it onto the code as it stands at `46b8d82`.

The pseudocode is meant to be read by someone who doesn't write TypeScript. Words in CAPITALS are operations defined on this page.

## How the pieces fit

There are three layers. Each one only knows about the one below it.

```
  the machine      takes events in, hands each one to the reducer, keeps the result
       │
  the reducer      decides what one transaction does: which checks pass, what gets written
       │
  the graph        holds the state, and remembers who wrote each piece and whether to trust it
```

The graph and the reducer are plain functions with no XState in them. The machine stays the front door, so `ocfValidator()` and `createActor(ocfMachine)` keep working.

## The graph layer

### What it holds

```
A FIELD is one fact about a node.
    value         what we believe it is
    written by    the transaction that last set it
    taint         empty, or the root transactions that make it untrustworthy

A NODE is one thing in the cap table that has state.
    id
    kind          security, stock class, stock plan or issuer
    fields        a security has:   lifecycle, quantity, holder, class, came from
                  a stock class:    authorized, outstanding, ratio
                  a stock plan:     reserved, available
                  the issuer:       authorized

The GRAPH is every node, looked up by id.
    It is never edited in place.
    Every operation hands back a new graph that shares whatever it did not touch.
```

### What you can do with it

```
LOOK UP (graph, node, field)
    the node is not there           → MISSING
    the field is tainted            → TAINTED, with its roots
    otherwise                       → the value

SET (graph, node, field, value, by transaction)
    create the node if it is not there
    store the value and who wrote it
    clear any taint on that field
    → a new graph

ADJUST (graph, node, field, amount, by transaction)
    add the amount to the value, record who wrote it
    keep any taint that was there
    → a new graph

TAINT (graph, node, field, roots)
    create the node if it is not there, with its value unknown
    add the roots to that field's taint
    → a new graph
```

SET clears taint and ADJUST keeps it. That one difference is rule 4 of the proposal: a clean new value can be trusted, a running total built on a bad number cannot.

### How transactions and checks plug in

Every transaction type declares what it touches. Two examples:

```
WARRANT EXERCISE
    reads     the warrant's lifecycle
              each resulting stock security: that it exists, its date, its holder
    writes    SET the warrant's lifecycle to removed
              SET "came from" on each resulting security

STOCK ISSUANCE
    reads     the stakeholder and the stock class
    writes    SET a new security: lifecycle, quantity, holder, class
              ADJUST the class's outstanding upward                    (cut 2)
```

Every check declares what it reads:

```
CHECK issuance-exists
    reads     this security's lifecycle
    passes    when the lifecycle is neither MISSING nor removed
```

The reducer uses these declarations. A check never looks at taint itself.

### The lifecycle of a security

One chart per family. They differ only in which transactions the family has.

```
    not issued ── issuance ─────────────────────────────────▶ outstanding

    outstanding ── acceptance, vesting, repricing,
                   partial exercise, partial release ───────▶ outstanding

    outstanding ── retraction, cancellation, repurchase,
                   transfer, conversion, warrant exercise,
                   reissuance, consolidation ───────────────▶ removed

    removed ── return to pool ──────────────────────────────▶ removed

    Anything not drawn here is not allowed.
```

"Is this transaction allowed in this state?" is a check like any other. "Which state comes next?" is one of the transaction's writes.

## The reducer

### The whole run

```
VALIDATE (package)
    graph      ← empty
    findings   ← none
    pictures   ← none

    graph, findings ← CHECK REFERENCE DATA (package)
    cures           ← for each ratification, the transactions it names        (cut 3)

    for each DAY in the package, earliest first
        for each TRANSACTION in ORDER OF (day)
            graph, new findings ← STEP (graph, transaction, cures)
            add the new findings
        graph, new findings ← END OF DAY (graph, day, cures)                  (cut 2)
        add the new findings
        add PICTURE (graph, day) to pictures

    → findings, pictures, SUMMARY (findings)
```

### Before the first day

```
CHECK REFERENCE DATA (package)
    make a node for the issuer, each stock class and each stock plan,
        with the starting values the package gives them
    check each stakeholder, vesting terms object and stock legend
    for each one that is invalid
        record a FAILED finding
        remember it, so that any check that reads it comes back BLOCKED
    → graph, findings
```

### The order inside one day

```
ORDER OF (day)
    A goes before B when
        A issues a security and B is anything else on that security
        A issues a security that B names as its result
        B removes a security and A is anything else on that security

    sort the day's transactions by those rules
    where the rules don't decide, sort by id
    if the rules go round in a circle
        every transaction in the circle FAILS with "circular reference"
```

### One transaction

```
STEP (graph, transaction, cures)

    if the transaction's type is unknown
        record one FAILED finding
        TAINT the security it names, if it names one
        → graph, that finding

    for each CHECK on this transaction's type, lifecycle check included
        tainted ← the fields this check reads that are TAINTED
        if there are any
            the check is BLOCKED, pointing at their roots
        otherwise
            problems ← run the check against the graph
            no problems                                         → PASSED
            problems, but the check is only a warning           → WARNED
            problems, the check is ratifiable,
                and a ratification names this transaction       → CURED
            otherwise                                           → FAILED

    if nothing FAILED and nothing was BLOCKED
        graph ← do the transaction's writes, with SET and ADJUST
    otherwise
        roots ← this transaction, if anything FAILED
                the roots that blocked it, if it was only BLOCKED
        graph ← TAINT every field the transaction writes, with those roots

    → graph, one finding for each check that did not simply pass
```

Two things to notice. A failed issuance still leaves a node behind, because TAINT creates it; that is what lets a later acceptance come back BLOCKED and not "no such security". And a CURED check counts as passing, so the writes are applied and nothing downstream is blocked.

### The end of a day

```
END OF DAY (graph, day, cures)                                                (cut 2)
    for each class, each plan and the issuer, if anything touched it today
        for each aggregate check on it
            (outstanding is no more than authorized, granted is no more than reserved)
            same outcomes as in STEP: BLOCKED, PASSED, WARNED, CURED or FAILED
    → graph, findings
```

### The picture and the summary

```
PICTURE (graph, day)
    every security whose lifecycle is outstanding, with its values
    each one marked trusted, tainted (and by which roots) or cured
    the class, plan and issuer totals, marked the same way                    (cut 2)

SUMMARY (findings)
    nothing failed                  "appears valid", with the count of warnings and cures
    otherwise                       "N errors, M checks blocked by them, K cured, W warnings"
```

### How the machine calls it

```
on START                     graph, findings ← CHECK REFERENCE DATA; build cures
on any transaction event     graph, findings ← STEP (graph, the transaction, cures)
on RUN_EOD                   END OF DAY, then add a PICTURE
on RUN_END                   result ← SUMMARY
```

One transition per event, no guard, and no state that means "stopped on an error".

## A worked example

Three warrants. Warrant A is issued to a stakeholder who isn't in the package.

```
day 1   issue A        stakeholder-exists FAILED          A is created, tainted, root = "issue A"
day 1   issue B        all checks PASSED                  B is outstanding
day 2   accept A       reads A's lifecycle: TAINTED       BLOCKED by "issue A"
day 2   accept B       PASSED
day 3   transfer A     reads A's lifecycle: TAINTED       BLOCKED by "issue A"
                                                          the security it results in is tainted too
day 3   cancel B       PASSED                             B is removed
day 4   accept B       lifecycle check FAILED             "B was removed by cancel B on day 3"

summary: 2 errors, 2 checks blocked by them
```

Today the run stops at the first line and reports one error. Warrant B's mistake on day 4 is never seen.

## What has to change

| Where | Today | Becomes |
|---|---|---|
| `types/validator.ts`, `OcfMachineContext` | Four issuance arrays, `removals`, `report`, `lastErrorFindings` | One `graph`, plus `cures`. `findings`, `snapshots` and `result` stay |
| `types/finding.ts` | `severity`, `check`, `message`, `subject` | Adds an outcome (failed, blocked, cured) and the roots or the ratification it points at |
| `validators/checkKit.ts`, `CheckObject` | `id`, `severity`, `description`, `run` | Adds `reads`, and later `ratifiable` |
| `validators/checkKit.ts`, `toValidate` | Runs every check, returns findings | Looks up each check's reads first; a tainted read gives BLOCKED without running it |
| `validators/checkKit.ts`, `Spec` | `effect` and `collection` | `reads` and `writes` |
| `validators/checks.ts`, `issuance-exists` | Searches the family array, then scans the log to explain a miss | Looks up the security's lifecycle |
| `validators/checks.ts`, `no-other-transactions` | Scans the log for other transactions on the security | Becomes the lifecycle check |
| `validators/checks.ts`, `resulting-stock-*` | Scan the log for the resulting issuance | Look up the resulting security, which ORDER OF guarantees is already there |
| `ocfMachine.ts`, transaction handlers | Guard, then one of two branches | One transition that calls STEP |
| `ocfMachine.ts`, unknown type (`*`) | Goes to `validationError` and stops | A FAILED finding inside STEP; the run continues |
| `ocfMachine.ts`, `RUN_END` | Writes "appears valid" | Writes SUMMARY |
| `ocf_validator/index.ts` | Sorts by date, then by a ranking of transaction types | Groups by day and calls ORDER OF |
| `ocf_validator/eod.ts` | Copies the four arrays | Builds PICTURE from the graph |
| `types/snapshot.ts` | Entries have no trust marker | Each entry says trusted, tainted or cured |
| The 8 legacy stock validators | Return a boolean or a report object | Cut 1: wrapped, so a failure becomes one FAILED finding. Cut 2: rewritten as checks (#159) |

The 20 graded validator files (`tx_warrant_exercise.ts` and the like) barely change. They list checks by name, and the reads live on the shared checks. Each file swaps `effect` and `collection` for `reads` and `writes`.

## What we need to add

- **`ocf_validator/graph.ts`**: the node and field types, and LOOK UP, SET, ADJUST and TAINT.
- **`ocf_validator/lifecycle.ts`**: one chart per family, used as a lookup. On the pinned XState this is `resolveState`, `can` and `getNextSnapshot`.
- **`ocf_validator/reducer.ts`**: STEP, ORDER OF, END OF DAY, PICTURE and SUMMARY.
- **A reference data pre-pass**, and the checks that say what makes a stakeholder, vesting terms object or legend invalid. Those definitions don't exist yet.
- **Reads and writes for all 45 transaction types.** For the 17 passthrough types this is the first time anything is said about them. In cut 1 they need only their effect on a security's lifecycle.
- **Tests.** Unit tests for each graph operation and for STEP, one outcome at a time. Small packages run through `ocfValidator` for the chains (the worked example above is one). A test that fails when a check reads something it didn't declare. A test that shuffles a package and expects the same output.

## What we can remove

Removed in cut 1:

- **The four-array bookkeeping:** `appendToCollection`, `removeFromCollection`, `collectionUpdate`, `CollectionKey`, `CollectionElement`, and the `EXPECTED_COLLECTION` list in `tests/ocfMachine.test.ts`.
- **The removals index and everything hung on it:** `earliestRemovals`, `context.removals`, the `noEarlierRemoval` check, `notRemoved`, `afterRemoval`, the `followsRemoval` guard, `recordRemoval` and `setRemovalError`. "Was this removed earlier" is now the lifecycle check.
- **The halt machinery:** the `isValidTx` guard, `isValidOutcome`, the two-branch transitions, `setValidationError`, `failureResult`, `lastErrorFindings`, and `validationError` as the place a failed transaction goes.
- **The same-day type ranking** in `index.ts`. `constants.transaction_types` no longer decides order.
- **All three log scans** in `validators/checks.ts`.

Removed when stock moves to findings (#159):

- `legacyValidate`, the legacy arms of the dispatch helpers, `context.report`, and the cut 1 wrapper.

## What stays as it is

- `read_ocf_package`, and the malformed-transaction and thrown-error handling in `index.ts`.
- `defineValidator`, `asGap`, `text`, and the check metadata the coverage report reads.
- Every check's id, severity and description, so findings keep their names.
- `ocfSnapshot`, which only picks a snapshot by date.

## An order to land cut 1

Each step is one pull request that leaves the suite green.

1. Add `graph.ts` with its tests. Nothing uses it yet.
2. Add the outcome field to `Finding`. Every existing finding is "failed". No behaviour changes.
3. Fill the graph alongside the four arrays, and move the shared checks over to LOOK UP. Then delete the arrays.
4. Add the lifecycle charts. Replace `no-earlier-removal` and `no-other-transactions` with the lifecycle check, and delete the removals index.
5. Add `reads`, STEP, taint and the blocked outcome, and stop halting. These go together: blocked only appears once the run continues past a failure.
6. Replace the type ranking with ORDER OF.
7. Add the reference data pre-pass.

Step 5 is the one that changes what users see. It rewrites `result` and the characterization snapshot.

## Not settled by this sketch

- **The public type changes.** `OcfMachineContext` is exported and is what `ocfValidator()` returns, so dropping the four arrays is a breaking change. The choice is a major version, or keeping the arrays for one release as read-only views built from the graph.
- **`issuance-exists` loses a diagnostic.** Today it scans ahead to say "this security is issued later, on this date". A reducer that never looks ahead can only say "not issued as of this date". Keeping the richer message needs an index built at START.
- **Wrapping the legacy stock validators** for cut 1 is untested.
- **Which fields a security node carries** beyond lifecycle. Cut 1 needs only what today's checks read: holder, date and class.
