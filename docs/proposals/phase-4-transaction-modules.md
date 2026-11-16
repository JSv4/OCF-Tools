# Phase 4: one module per transaction type

Companion to `phase-4-design-sketch.md`. That file defines the graph and the reducer. This one defines how a transaction type plugs into them, so that support can be partial, honest about what it skipped, and grown one type at a time.

The numbers here come from the descriptor table at `46b8d82`.

## The idea

Each transaction type is described by one module. A module can be nearly empty or complete. The reducer treats every module the same way and never needs to know how complete it is.

What makes partial support safe is one rule:

> If a module knows a transaction changes a field but doesn't yet know how, the reducer marks that field **unsupported** from that point on. A check that reads an unsupported field is reported as **not checked**. It is never reported as passed.

So support is decided per package, at run time, field by field. A package with no stock split gets every quantity check, even though splits aren't implemented. A package with a split gets everything except the checks that depend on what the split changed, and is told which ones and why.

## What a module is

```
A MODULE describes one transaction type. Every part is optional.

    names         which nodes the transaction points at, and through which of its fields
    lifecycle     the move it makes on a security's chart, if any
    checks        the checks to run on it; each is written, or declared as a gap
    writes        the fields it changes; each one says HOW, or says "not yet"
    stands for    another type whose module to use (for deprecated names)
```

Four examples, from nearly empty to complete.

```
MODULE stock class split                                       recognized only
    names         the stock class, through stock_class_id
    writes        the class's ratio                            not yet
                  the quantity of every security in the class  not yet

MODULE plan security issuance                                  borrowed
    stands for    equity compensation issuance

MODULE stock class authorized shares adjustment                complete, and tiny
    names         the stock class, through stock_class_id
    checks        the class exists
    writes        the class's authorized                       SET to new_shares_authorized

MODULE warrant exercise                                        complete
    names         the warrant, through security_id
                  each resulting security, through resulting_security_ids
    lifecycle     outstanding → removed
    checks        issuance-exists, resulting-security-named, resulting-stock-exists,
                  resulting-stock-dated, resulting-stock-stakeholder
    writes        the warrant's lifecycle                      SET to removed
                  "came from" on each resulting security       SET to this warrant
```

## What the reducer does with it

This replaces STEP in the design sketch. The only new parts are the "not yet" branch and the "not checked" outcome.

```
STEP (graph, transaction)
    module ← the module for this transaction's type, following "stands for"

    if there is no module                         (a type OCF itself doesn't define)
        record one FAILED finding
        MARK UNSUPPORTED every field of the security it names, if it names one
        → graph, that finding

    for each CHECK in the module, plus the lifecycle check if it has a lifecycle
        if the check is a declared gap            → nothing; the coverage report lists it
        look at the fields the check reads
            any tainted by an error               → BLOCKED, pointing at the roots
            any marked unsupported                → NOT CHECKED, naming the transaction that did it
            otherwise run it                      → PASSED, WARNED, CURED or FAILED

    if anything FAILED or was BLOCKED
        graph ← TAINT every field the module writes
    otherwise, for each WRITE in the module
        it says HOW                               → do it, with SET or ADJUST
        it says "not yet"                         → MARK UNSUPPORTED that field

    → graph, one finding for each check that did not simply pass
```

```
MARK UNSUPPORTED (graph, node, field, by transaction)
    the same as TAINT, but the reason recorded is "not supported yet", not "error"
    a later SET on that field clears it, exactly as it clears taint
```

Blocked and not checked are kept apart on purpose. Blocked means the package has an error upstream. Not checked means the validator has a limit. The first is the user's to fix and the second is ours.

The summary says so:

```
"Appears valid as far as it could be checked. 6 checks were not run, because the
 package contains TX_STOCK_CLASS_SPLIT, which the validator does not apply yet."
```

## The ladder

Every module is somewhere on these four rungs. The rung is worked out from the module, not declared.

| Rung | The module has | What the validator can then do for that type |
|---|---|---|
| 1. Recognized | `names`, and `writes` all "not yet" | Mark what it touched as unsupported. Nothing is silently skipped |
| 2. Lifecycle | a `lifecycle` move | Catch it happening to a security that isn't there or was already removed |
| 3. Checked | written `checks` | Report its own errors |
| 4. Applied | every write says HOW | Keep the fields it touches trustworthy, so checks elsewhere stay live |

Each rung is one small pull request with a fixed shape of test:

1. **Recognized.** No new test. The registry-wide tests below cover it.
2. **Lifecycle.** One allowed case and one not-allowed case, generated from the chart.
3. **Checked.** For each check, one package that passes and one that fails, as the family test files do now.
4. **Applied.** The graph before and after one transaction, plus a package showing a check that used to come back "not checked" now running.

## The current 45 types as modules

On the day the reducer lands, with no new validator work:

| Group | Types | Today | Rung as a module |
|---|---|---|---|
| Convertible | 6 | Graded. 15 of 18 declared checks run | 3 |
| Warrant | 6 | Graded. 15 of 16 run | 3 |
| Equity compensation | 8 | Graded. 12 of 22 run | 3, with exercise and release missing their quantity writes |
| Stock | 8 | Legacy validators | 3, through an adapter (below) |
| Plan security | 7 | Passthrough | Borrowed from equity compensation with `stands for`, if their fields match |
| Vesting | 3 | Passthrough | 2. They require the security to be outstanding; vesting state is "not yet" |
| Authorized, ratio and pool adjustments | 4 | Passthrough | 1, and each is a one-line SET away from 4 |
| Stock class split | 1 | Passthrough | 1 |
| Consolidation | 1 | Passthrough | 2. It removes several securities and results in one |
| Return to pool | 1 | Passthrough | 2. It requires the security to be removed; plan available is "not yet" |

The legacy stock validators become modules without being rewritten:

```
LEGACY MODULE (the old validator, its lifecycle move)
    names         the security, through security_id
    lifecycle     as given
    checks        one check that runs the old validator and FAILS if it says no
    writes        the security's lifecycle
```

That adapter is deleted when stock is rewritten as checks (#159).

### The 14 declared gaps, and what each is waiting on

Of the 56 checks the graded validators declare, 42 run and 14 are gaps. Seen through the graph, most are gaps because a field they need isn't being written yet.

| Gap | Where | Waiting on |
|---|---|---|
| `no-other-transactions` | Convertible, warrant and equity compensation transfer; equity compensation exercise | Nothing new. The lifecycle check should cover it once the charts exist |
| `quantity-outstanding` | Equity compensation exercise, release, cancellation, transfer | Exercise and release writing the security's quantity |
| `quantity-reconciles` | Equity compensation exercise, convertible conversion | Quantity on the securities involved |
| `balance-security-exists` | Convertible conversion | The check being written; the reads are already there |
| `resulting-stock-exists`, `-dated`, `-stakeholder` | Equity compensation exercise | The checks being attached; they already run for warrant exercise |

## What to build next, and what each step unlocks

This table can be generated from the modules: for any field, list the modules that still say "not yet", and the checks that read it.

| Step | Modules | Unlocks |
|---|---|---|
| 1 | Plan security, as `stands for` | 7 types validated by checks that already exist |
| 2 | The four adjustments, as SET | Authorized, ratio and reserved become known values |
| 3 | Quantity on equity compensation exercise and release | `quantity-outstanding` in four places |
| 4 | Class outstanding on stock issuance and the stock removals | The authorized-shares check, which is new |
| 5 | Plan available on equity compensation issuance, cancellation and return to pool | The pool check, which is new |
| 6 | Stock class split | Quantities stay trusted across a split |
| 7 | Consolidation writes | Quantity on the resulting security |
| 8 | Vesting state | Waits on #165 |

Steps 1 and 2 are small and need no new concepts. Steps 4 and 5 are where ratification first has something to cure.

## Tests that cover every module automatically

These run over the whole registry, so a new module is tested the moment it is added.

- **Every OCF transaction type has a module.** This is already enforced at compile time by `satisfies DescriptorTable`.
- **No undeclared reads.** Run each check against a recording graph and fail if it looks up a field it didn't declare.
- **No undeclared writes.** Apply each module's writes to a fixture and fail if any field changed that the module didn't list.
- **"Not yet" never leaves a field trusted.** After a module with a "not yet" write runs, that field must be marked unsupported.
- **Order doesn't matter.** Shuffle a package's transactions and expect the same findings and snapshots.
- **The coverage report matches the registry.** The report of rungs, gaps and "not yet" writes is generated from the modules, so it can't drift from them.

## Where this goes in the code

- **`defineValidator` grows into the module definition.** Its `Spec` keeps `transaction` and `checks`, drops `effect` and `collection`, and gains `names`, `lifecycle`, `writes` and `standsFor`.
- **`asGap` gets a sibling for writes.** A check with no `run` is already a declared gap that the coverage report lists. A write marked "not yet" is the same idea applied to writes.
- **`TX_DESCRIPTORS` stays the registry.** The `passthrough` effect goes away. Those 17 entries become real modules at rung 1 or 2.
- **The `afterRemoval` special case goes away.** Today the machine runs `no-earlier-removal` by hand for passthrough types and exempts return to pool. With a `lifecycle` on those modules, the chart says the same thing.
- **`Finding` gains "not checked"** next to blocked and cured, carrying the transaction that marked the field.
- **`validators/checks.ts`** drops its "uses the deprecated type, which the validator doesn't process" message once plan security borrows the equity compensation modules.

## Not settled

- **Whether plan security can borrow outright.** OCF says `TX_PLAN_SECURITY_ISSUANCE` is kept only to avoid a breaking change before v2.0.0, which suggests the shapes match. That needs checking field by field for all seven.
- **How loud "not checked" should be.** A split marks every quantity in its class. Nothing reads quantity today, so nothing is reported. Once quantity checks exist, one split could produce many "not checked" lines, and they may need grouping by cause.
- **Whether an unsupported write should also block the transaction's own later checks**, or only checks on other transactions. The pseudocode above only affects later transactions.
