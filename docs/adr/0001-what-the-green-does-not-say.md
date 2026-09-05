# ADR-0001 — What the green does not say

- status: accepted
- date: 2026-09-05
- measured against: `main 3e0a0fd`
- related: `adr-2800010300-cloud-itonami-souji-west-registration` (superproject
  `90-docs/adr/`, accepted — the source of truth for this repo that **exists**)

## Context

`clojure -M:test` returns `Ran 7 tests containing 16 assertions. 0 failures,
0 errors.` [docs/operator-quickstart.md](../operator-quickstart.md) shows an
operator how to run that and how to watch the refusals happen.

This ADR records the other half: **what that green does not check.** It is
written from probes, not from reading the source, because the failure mode the
superproject CLAUDE.md keeps naming — *a check that could not run returns the
same value as a check that ran and found nothing* — is invisible from reading.

Three of the four items below are open. The ADR does not fix them; recording
them and fixing them are different iterations.

## Decision

### 1. `blueprint.edn` drift is unchecked, and the probe stays green

`blueprint.edn` and `src/souji/design/operation.clj` both declare the same five
ops. **Nothing compares them.** `operation_test.clj` holds the op set as a
literal (`#{:design/propose :design/validate :safety/gate :sim/run
:bom/observe}`) and never reads `blueprint.edn`.

Measured twice — once against `af0b67d`, once against `3e0a0fd` after the
quickstart landed. Adding to `blueprint.edn`:

```clojure
{:op/name :robot/command
 :op/effect :execute
 :op/note "FABRICATED drift probe"}
```

and running `clojure -M:test` gives **`Ran 7 tests containing 16 assertions.
0 failures, 0 errors.`**, exit 0.

So the one thing this repo exists to make impossible — an op with an
`:execute` effect that drives hardware — can be written into the document the
fleet reads to understand this repo, and **nothing turns red.** The
`actuator-paths-are-absent` test looks at `op-names`, which comes from `src/`;
`blueprint.edn` is outside its reach.

The quickstart's closing rule ("do not add an op that actuates") is enforced on
one of the two files that declare ops. This is the next `axis-test` iteration:
a test that reads `blueprint.edn` and asserts the two op sets are equal, and
that no `:op/effect` outside `#{:propose :observe}` appears.

### 2. The mutation runner's exit 0 means "nothing registered"

From the superproject:

```bash
nbb scripts/maturity-loop/run.cljs --only cloud-itonami-souji
```

returns `maturity-loop: 0 suite / policy maturity-loop/mutation/v1`,
`噛む=0 噛まない=0 エラー=0 skip=0`, **exit 0**.

`scripts/maturity-loop/mutations.edn` has **no entry** for this repo. "No
breakage was registered" and "every registered breakage was caught" print the
same summary and exit the same. Do not read this green as evidence about test
quality.

### 3. Maturity is scored on `:default` weights

`manifest/repo-taxonomy.edn` has no `:repo/kind` row for this repo, and the
fleet maturity tick warns about it every round. The score is therefore computed
with `:default` weights, so this repo's `M_own` **is not comparable** to repos
whose kind is known.

A design-only governor ⊣ advisor looks like `actor`, but that is a judgement,
not a measurement, and the taxonomy is owned by the superproject. Not decided
here.

### 4. The README named a source of truth that does not exist (fixed here)

Until this commit the README said "ADR-2609011520 が正本". Querying the
superproject:

```bash
git ls-tree -r --name-only origin/main -- 90-docs/adr | grep 2609011520   # 0 results
```

That ADR does not exist. The neighbouring numbers do
(`2609011400-an-agent-has-no-gesture.edn`,
`2609011842-awai-japanese-models-openrouter-provider-application.edn`), so the
number band is real and this one was simply never landed. The accepted ADR that
does cover this repo is
`2800010300-cloud-itonami-souji-west-registration.edn`, and the README now
points there.

**The sibling repo `cloud-itonami-carrier` carried the identical defect** — its
own `docs/adr/0001` is titled "the source of truth this readme names does not
exist". Same scaffold, same broken reference. When the next repo in this family
is created, check that the ADR its README names is resolvable before landing.

## Consequences

- An operator reaching this repo through the quickstart can find the three open
  gaps instead of reading `0 failures` as "checked".
- Item 4 is fixed in this commit. **Items 1–3 are open**, and this ADR does not
  claim otherwise: 1 belongs to a test iteration, 2 to a `mutations.edn`
  registration, 3 to the superproject taxonomy.
- `.cpcache/` was committed with absolute paths baked in until `2a1bd35`, which
  removed it and added `.gitignore`. Independently measured here before that
  commit was seen: the CLI regenerates rather than consumes a corrupted cache,
  so it broke nobody; but running under a different `HOME` emits
  `.cpcache/1079720815.*` beside the committed `.cpcache/4246712940.*` and
  leaves the tree dirty — and a dirty west checkout is refused by `west update`
  and skipped by the fleet maturity scan. Recorded because the *reason* it went
  unnoticed is that on the machine that generated it the regenerated bytes match.
