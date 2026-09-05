# Operator quickstart

Every command below was run against this repository at the commit that added
this file, and the output shown is what it printed. If a command here does not
do what it says, that is a defect in the repository, not in your setup.

What you can do with this repo today is narrow, and the narrowness is the
point: **there is no code path that drives a robot.** You can ask the design
actor for a proposal, and you can watch the governor refuse the proposals that
would be unsafe.

## Before you start

You need a JVM and the Clojure CLI. Nothing else — no credentials, no network
service, no hardware.

```
$ clojure --version
Clojure CLI version 1.12.x
```

The first command will download dependencies into `~/.m2` and `~/.gitlibs`.
That takes a minute; every command after it is fast.

## 1. Run the tests

```
$ clojure -M:test
```

```
Testing souji.design.operation-test

Ran 7 tests containing 16 assertions.
0 failures, 0 errors.
```

If this is not green, stop here. Everything below assumes it is.

## 2. See which ops exist

```
$ clojure -M -e "(require '[souji.design.operation :as op]) (prn (sort (map name op/op-names)))"
```

```
("gate" "observe" "propose" "run" "validate")
```

Five, and they are the whole surface: `:design/propose`, `:design/validate`,
`:safety/gate`, `:sim/run`, `:bom/observe`. There is no sixth one waiting
behind a flag.

## 3. Watch the safety gate refuse

This is the thing worth seeing first. A design proposal that does not carry
both mandatory safety gates is refused by the governor — not warned about,
refused.

```
$ clojure -M -e "
(require '[souji.design.operation :as op])
(try (op/govern (op/propose :design/propose {:brush :dual-roller}))
     (catch clojure.lang.ExceptionInfo e
       (println \"REFUSED:\" (.getMessage e))
       (prn (:missing (ex-data e)))))"
```

```
REFUSED: safety gates are mandatory: child-pet-contact-stop and
                       sharp-object-stop must be present in every design proposal
(:child-pet-contact-stop :sharp-object-stop)
```

The exception names which gates were missing. Add one of the two and it will
name the other.

## 4. A proposal that carries both gates

```
$ clojure -M -e "
(require '[souji.design.operation :as op] '[clojure.pprint :refer [pprint]])
(pprint (op/govern (op/propose :design/propose
                               {:brush :dual-roller
                                :safety/gates {:child-pet-contact-stop true
                                               :sharp-object-stop true}})))"
```

```
#:proposal{:op :design/propose,
           :effect :propose,
           :payload
           {:brush :dual-roller,
            :safety/gates
            {:child-pet-contact-stop true, :sharp-object-stop true}},
           :status :approved-for-review,
           :note "R0: propose-only; actuation path does not exist here"}
```

Read the status carefully. It is `:approved-for-review`, **not** `:done`. The
governor approved the proposal for a human to look at. Nothing was built,
ordered, or actuated, because at R0 there is no code that could.

`clojure.pprint` has to be required explicitly in a `-e` expression. It is in
the command above because the version without it fails with
`ClassNotFoundException`, which is how this line was found.

## 5. The one op that completes

`:bom/observe` is the only pure observation, so it is the only op that reaches
`:done` inside the governor. Everything else stops at review.

```
$ clojure -M -e "
(require '[souji.design.operation :as op])
(println (:proposal/status (op/govern (op/propose :bom/observe {:as-of \"2026-09-05\"}))))
(println (:proposal/status (op/govern (op/propose :sim/run {:floor-m2 42}))))"
```

```
:done
:approved-for-review
```

`:sim/run` looks like it should just compute something, and at R1 it will. At
R0 it is declared and gated but the simulation itself is not written, so it
returns for review like every other `:propose` op.

## 6. Confirm there is no way to drive hardware

```
$ clojure -M -e "
(require '[souji.design.operation :as op])
(doseq [o [:motor/drive :robot/start :actuator/command]]
  (println o \"admitted?\" (op/admitted-op? o)))
(try (op/propose :motor/drive {}) (catch clojure.lang.ExceptionInfo e (println \"REFUSED:\" (.getMessage e))))"
```

```
:motor/drive admitted? false
:robot/start admitted? false
:actuator/command admitted? false
REFUSED: op not admitted
```

These names are refused because they are not in `ops`, and `ops` is the only
place an op can come from. That is the whole mechanism: absence, not a policy
check that could be configured off.

## What this repo does not do yet

The README's stage table is accurate. At R0 there is a blueprint, an actor
skeleton, and its tests. In particular:

- **`:sim/run` runs no simulation.** Floor area, obstacle count and battery
  are accepted as payload and passed through. R1 is where that becomes real.
- **`:design/validate` validates nothing beyond op admission.** The
  cleaning-rate and obstacle-avoidance invariants the README names are stated,
  not computed.
- **There is no BOM.** `:bom/observe` completes because observing is defined as
  completing, not because there is a bill of materials to read.
- **Every number in `blueprint.edn` is modelled**, and the repo says so. No
  measurement of a physical robot exists, because no physical robot exists.

## If you are extending this

The one rule that is not a preference: **do not add an op that actuates.**
The safety gates are enforced by the governor, but the reason no runaway is
possible at R0 is simpler than that — there is no actuation code to run. That
property is worth more than the gate, and it is lost the first time someone
adds a `:motor/*` op "just for testing".

If you add an op, add it to `ops` with its effect class and add a test that
asserts what it refuses, not only what it accepts.
