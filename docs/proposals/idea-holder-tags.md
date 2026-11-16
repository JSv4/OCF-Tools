# Idea: tags on issuances and on aggregates (asserted and derived)

An idea to keep, not a plan. It is separate from the graph validator proposal (Open-Cap-Table-Coalition/OCF-Tools#225), though it would build on it.

## The idea

Let a cap table carry arbitrary tags, of two kinds:

- **Asserted.** A label attached directly to an issuance, for example "qualified holder". Someone declares it.
- **Derived.** A label that follows from a rule over aggregates, for example "anyone holding at least 12.5% of a class, fully diluted, is a qualified holder". It is computed, not declared.

## Notes

- A derived tag is a computation over one day's snapshot, so it has a timeline: a holder can gain or lose the tag as others are issued shares.
- It would run at the same end-of-day stage as the aggregate checks in the graph proposal, and inherit taint: a tag computed from untrusted numbers is itself untrusted.
- Comparing the two kinds is a check in its own right: a holder asserted as qualified who is below the threshold is a finding.
- Threshold rules are usually per holder (often with affiliates), not per issuance, so derived tags probably attach to a stakeholder within a class. Asserted tags can sit on either.
- "Fully diluted" is defined differently from document to document (as-converted or not, unissued pool included or not), so a rule has to name its basis.

## Open

- Where the rules live: in the package, in a sidecar, or in validator configuration.
- OCF has no tag or label field today (none in the vendored types or the schema), so asserted tags need either an OCF enhancement or a sidecar.
- Depends on class, plan and issuer aggregates being tracked, which the validator does not do yet.
