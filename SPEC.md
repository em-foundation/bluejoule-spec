# BlueJoule Specification

**Version:** 1.2. Supersedes v1.1.
**Authors:** Bob Frankel (The EM Foundation), Mohammad Afaneh (Novel Bits).
**License:** CC BY 4.0. See `LICENSE.md`.

The decisions behind this version, and the open items still marked provisional, are recorded in
`CHANGELOG.md`.

---

## 1. Purpose and scope

BlueJoule is a specification for measuring the energy cost of Bluetooth LE activity, not a tool. Any
instrument and any software that satisfy this document can produce a conformant capture. EM•Scope is
the reference implementation; it is not the definition.

This document holds only what is invariant across every BlueJoule benchmark. A benchmark repository
holds one prescribed activity and its captures, and links here for everything else.

**The same primitives serve two uses, and this document supports both without letting either
overclaim the other.** Activity-centric comparison measures one narrowly declared activity across
many platforms, which is what a benchmark ranking is. Platform-centric composition combines many
declared activities on one platform into an application-level energy model, which is what a product
budget is. A capture may serve both, but a generic benchmark primitive is not automatically a
product model, and a product-specific capture is not automatically a fair ranking entry. The
declaration rules below (scope, environment, fidelity) exist so each use knows the limits of the
other.

## 2. The measurement model

```text
activity
    measured on a platform
    powered by a source
    at an operating point
    observed by an analyzer
```

`activity` is the only variable. Platform, source, operating point and analyzer are the same kind of
noun in every benchmark that will ever exist. A new benchmark defines a new activity and changes
nothing else.

## 3. Definitions

**Activity.** The prescribed, observable behavior being measured, defined by over-the-air behavior
rather than by a canonical source implementation. An activity declaration states the parameters a
conformant implementation must reproduce. Two further declaration duties:

- An activity whose per-event energy depends on peer behavior must declare that behavior as part of
  the activity (a scannable advertiser's scan-request load, a connection's prescribed peer), and the
  capture states which environmental conditions were **controlled**, which were merely **observed**,
  and which were **uncontrolled**; otherwise two conformant labs in different radio environments
  measure different numbers with neither of them wrong.
- An activity that scores only part of what its capture records (for example, a connected
  transaction that excludes pre-connection acquisition) must declare its **accounting scope**: the
  included and excluded regions (section 7.2).

An activity is either **periodic** (a repeating event at a rate) or **episodic** (a bounded episode
that occurs some number of times); section 6.2 gives each its composition mode.

**Platform.** The complete power-relevant execution environment: MCU, executing core, board, software
stack, clocking, retained memory, build configuration, and related settings. For single-core parts
"executing core" adds no burden; for multi-core parts it prevents two captures from sharing a chip
and board while differing materially in what actually ran.

**Power source.** What supplies the platform during the capture: a bench supply at a stated voltage, a
named cell, or a modeled source.

**Operating point.** The supply voltage and any other stated conditions under which the activity was
measured.

**Analyzer.** The instrument that observed the capture. Each analyzer carries a single-letter suffix
used in capture folder names (section 4).

## 4. The capture contract

```text
captures/
    <vendor-prefixed-platform-id>/
        build/
        <voltage>-<analyzer>/
```

Platform IDs are vendor-prefixed and stable, for example `nrf-54l15-zephyr`,
`sil-efr32xg22e-simplicity`, `ti-cc2340r5-simplelink`.

A platform folder may contain:

- `build/`, best-effort source project and generated firmware artifacts
- capture folders, one per operating point, named `<voltage>-<analyzer>`

**Analyzer suffix registry.** This document specifies that a suffix exists; analyzer declarations,
including the letter assignments, live in `PEDS/analyzers/`. The initial assignments:

| Suffix | Analyzer |
|---|---|
| `J` | Joulescope |
| `P` | PPK2 |
| `O` | Otii |

Declarations exist to make a capture auditable, never to approve instruments: BlueJoule is not an
approved-instrument or approved-bench-supply program. The guardrails are the measured voltage
behavior with its droop and ripple evidence, the closure residual, and the capture notes.

A finalized capture may contain:

```text
.emscope/
capture.yaml
analysis.yaml
emscope-capture.zip
CAPTURE.md
ABOUT.md
about.json
event-*.png
```

- `emscope-capture.zip` preserves the raw measurement and is managed through Git LFS
- `CAPTURE.md` carries optional capture-specific notes
- `ABOUT.md` and `about.json` are generated (section 8) and may be regenerated at any time

## 5. Authority of raw data

Raw capture data is the authoritative measurement record. Generated summaries and scores may be
recomputed as tooling evolves.

A regeneration that changes a published score is therefore expected and permitted. What may not change
without a version of this document changing is the definition the score is computed from.

## 6. Measured primitives and composed scores

A capture **measures**. A score at a rate is **composed** from what was measured. The two are
different kinds of statement and this document keeps them separate.

### 6.1 What a capture reports

A capture reports its measured primitives, and only those, as its primary result:

- `event_energy`, the total energy integrated over the **declared event window** (section 7), averaged
  across the detected events, with its standard deviation. It includes the platform's baseline draw
  during the window. The radio's detected on-air span may be reported separately but is not the window.
- `event_duration`, the declared event window length. It is the scoring integration window, never
  merely the visually obvious on-air span: an event that looks like 3 ms may have a far longer power
  consequence through its settling tail.
- `sleep_power` (equivalently `sleep_current` at the stated voltage), the average inside the declared
  sleep window
- the measured mean inter-event time at the captured operating point
- voltage statistics, analyzer metadata, and the resolved activity, platform and power declarations
- the boundary declaration and closure residual of section 7

An episodic capture (section 6.2) reports `episode_energy` and `episode_duration` analogously, with
the episode bounded exactly as an event window is: from a named fiducial to a declared end.

### 6.2 What is composed, and where

**Periodic activities** project to continuous operation at a stated period:

```text
energy_per_period = event_energy + sleep_power × (period − event_duration)     [J]
energy_per_day    = energy_per_period × (86400 / period)                       [J]
energy_per_month  = energy_per_day × 30                                        [J]
EM•erald          = CR2032_JOULES / energy_per_month     // 2400 J, equivalently 80 / energy_per_day
```

**Episodic activities** compose per occurrence:

```text
energy_per_day = episode_energy × occurrences_per_day
               + sleep_power × (86400 − occurrences_per_day × episode_duration)   [J]
```

with `energy_per_month` and the EM•erald defined from `energy_per_day` as above. The episodic mode
is deliberately lightweight in this version; it hardens with the first episodic benchmark revision,
so the periodic model never over-constrains transaction-style benchmarks.

All energies are in joules, all times in seconds, all powers in watts. The EM•erald month is 30 days
by definition. The formulas and the primitives share one charge convention: `event_energy` (and
`episode_energy`) is the total energy in the declared window, so sleep is charged only for the
remainder.

**Composition happens above the benchmark repositories.** This document defines the composition rules
and a registry of **named rates** (for example `adv-1s`, `adv-10s`); a benchmark declares which named
rates apply to its activity; frontends such as bluejoule.org, and conformant tools, compute a score at
any named rate from the published primitives. Benchmark repositories do not grow per-rate columns.
Named rates stay mechanical in this registry; friendlier application scenarios ("asset tag",
"sensor beacon") belong to the presentation layer, because a scenario implies far more than a period.

**A named rate is an exact period, and it names an activity instance within a family.** Composing at
`adv-1s` means `period = 1 s` in the formula above. `adv-1s` and `adv-10s` are distinct configured
instances of the same activity family, not two readings of one measurement; projection fidelity
(section 6.4) is the validated claim that connects instances of a family. The measured-mean rule of
section 6.3 governs the **captured** period: it maps a configured interval to the real mean the
device produced. A device configured at an interval equal to a named rate runs at a longer mean
period, so a comparison against a configured device maps through measurement.

A generated record may include example projections for convenience, but they are labeled as
projections under named rates, never presented as the primary measured result (section 8).

| Layer | Holds |
|---|---|
| benchmark repositories | authoritative captures and measured primitives |
| `about.json` | the structural record of one capture's measured quantities |
| this document | the composition rules, the boundary contract, and the named-rate registry |
| bluejoule.org and conformant tools | computed rate-specific EM•eralds and scenarios |

### 6.3 The captured period is measured, not configured

**The captured `period` is the measured mean inter-event time, not the configured interval.** For
undirected advertising the two are not the same: the Link Layer computes
`T_advEvent = advInterval + advDelay`, where `advDelay` is a (pseudo-)random value in the range
0 ms to 10 ms regenerated for each such advertising event, and advertising events "shall be
perturbed in time using the advDelay" (Core v6.3, Vol 6 Part B, §4.4.2.2.1). A device configured at
a one second interval therefore emits fewer than 86400 events per day. Projecting at the configured
interval would over-count events by up to `10 ms / period`, which is up to 1% at one second and up
to 50% at a 20 ms interval. Using the measured mean removes the correction rather than documenting
it, which is also what section 5 requires.

**Do not "correct" the configured interval with a constant.** The Core Specification fixes only the
0 ms to 10 ms range and specifies no distribution for `advDelay`, so any mean is a property of the
controller under test, not of Bluetooth LE. The familiar 0.5% midpoint assumes a uniform `advDelay`
and must not be written into a specification as though it were a specified value. Only the direction
and the bound are spec-guaranteed: `advDelay` is never negative, so projecting at the configured
interval always over-counts, by 0 to `10 ms / period`.

Two error terms, both named. The event window is counted once, and sleep is charged for the remainder
of the period rather than for the whole period; charging a full period of sleep in addition to the
event overstates by `sleep_power × event_duration`, which scales with `event_duration / period`. The
second term is the event-count error above. Both are negligible for advertising at a one second period
and material for any activity that occupies a real fraction of its period.

### 6.4 What a composed score is, and is not

**A composed score is a projection, not a measurement**, and a conformant presentation says so. A
direct capture at the target rate is always more authoritative than a projection to it.

**Precondition.** Projecting a capture to a period other than the one it was captured at assumes
two independences: per-event energy independent of the period, and the sleep floor independent of
the rate. The first holds while the period is well above the event duration, and fails as the two
approach each other. The second must be verified per platform rather than inherited (section 7.3).
A benchmark that projects outside the range where these were verified must say so.

**Fidelity, and why it is not the closure residual.** Two different quantities describe a
submission's quality, and they must not be conflated. The **closure residual** (section 7) is a
same-rate self-consistency check: it proves the declared numbers reproduce the capture's own
average. **Projection fidelity** is a cross-rate accuracy statement: how closely a score composed at
a rate that was never captured matches what a direct measurement at that rate would read. On the one
platform where fidelity has been tested so far (an nRF54L15 across a 100x interval range), composed
projections agreed with direct measurement to within roughly one to two percent, several times
larger than that platform's closure residual, and larger than the repeatability of a single direct
measurement. That figure is one part and one build; it must not be assumed elsewhere.

A submission therefore states its projection fidelity separately from its closure residual: either
**demonstrated**, by capturing the activity family at two or more rates and predicting each from the
other, or **inherited**, as a stated default uncertainty no smaller than the demonstrated one to two
percent, until its platform is verified.

**Ties.** A difference between two composed scores that is smaller than the combined stated
**fidelities** of the platforms behind them is not a ranking. A conformant leaderboard or comparison
presents such results as indistinguishable.

**The EM•erald unit.** 2400 J is the nominal energy content of a CR2032 at 3 V (about 222 mAh). It is
a **normalization unit, not a battery model**: no droop, no ESR, no state of charge, no load
dependence. **EM•eralds are not months of service life**, and a conformant implementation must not
present them as one. Predicting service life requires a real cell model and is out of scope for this
document.

More generally, and this applies to any figure derived from a BlueJoule capture: **average current
does not predict battery life; cell droop and end-of-life internal resistance dominate.**

## 7. The boundary declaration and the closure test

### 7.1 The problem

`event_energy` and `sleep_power` are both defined by where the event window is cut, and the
specification as inherited never said where. On one capture of one platform, cutting at the radio's
detected span versus at the settling plateau moved the sleep term by more than 40% at a 100 ms
period, with both readings defensible. Two conforming labs could publish different numbers from
identical hardware while each following the text. Mixing the two conventions misplaces settling
charge: paired one way, it lands once inside the event and again inside the floor and is counted
twice; paired the other way, it falls between the two windows and is dropped.

### 7.2 The declaration

Every capture record carries the boundary declaration:

| field | meaning |
|---|---|
| `event_window` | start and end of the integration window, relative to a named fiducial |
| `sleep_window` | where the floor was measured |
| `accounting_scope` | the region closure runs over: the whole capture by default; a scoped activity enumerates its excluded regions (for example, pre-connection acquisition) |
| `partition` | assertion that the event windows and the floor-charged remainder tile the accounting scope: no gap, no overlap |
| `closure_residual` | `abs(Q_w/T + I × (T − t_w)/T − measured_mean) / measured_mean` |
| `floor_residual` | 🆕 **v1.2.** `I − I_gap`, in amperes, **signed**, where `I_gap` is the mean current over the accounting scope outside all declared windows. Informative: it carries no tolerance of its own. It states the same disagreement `closure_residual` measures, in the units of the quantity under test, and it keeps the sign that the absolute value above discards |

The residual is defined over the accounting scope. Let `T_s` be the total duration of the included
regions, `Q_w` the total charge inside all declared event or episode windows within them, `t_w` the
total declared window time, `I` the declared sleep current, and `measured_mean` the average current
over the included regions of the same capture:

```text
closure_residual = abs((Q_w + I × (T_s − t_w)) / T_s − measured_mean) / measured_mean
```

For a periodic activity whose scope is the whole capture, this reduces to the per-period form in the
table when `T` is taken as `T_s` divided by the event count, with per-event averages of `Q_w` and
`t_w`; the table states the per-period form for that common case. The two forms are **not**
interchangeable when `T` is instead the measured mean inter-event interval and the scope is not an
exact integer number of periods, which is the normal case whenever a capture is trimmed by duration
rather than by event count. The scoped form above is the normative one; where the two differ, it is
the one that closes. `sleep_window` is where `I` was estimated;
it may lie anywhere the platform is demonstrably settled, including before the first event, and
need not be the remainder itself. The residual is section 6.2's composition evaluated over the
capture's own scope and compared against what the instrument recorded; the formulas share the same
charge convention by construction.

The windows are **declared, not mandated**. Settling behavior is chip-specific (a decoupling network,
a regulator refresh mode, a DC/DC ripple period), so a numeric window written into this document
would privilege one vendor's silicon. The sleep mode is likewise named per platform in its
declaration (System ON idle with RAM retention, EM2, standby), never assumed.

### 7.3 The closure test: a floor-declaration check

**A capture is valid only if its declared numbers reproduce the average current it actually measured
over its declared accounting scope**: `closure_residual` must fall within the tolerance below.

🆕 **v1.2, what the residual actually tests.** Because the declaration asserts that the event windows
and the floor-charged remainder tile the accounting scope (`partition`, 7.2), the residual reduces
exactly to the floor error:

```text
closure_residual = abs(I − I_gap) × (1 − t_w/T_s) / measured_mean
```

Every other term cancels: `Q_w` appears on both sides of the comparison and is drawn from the same
trace. This is an identity, not an approximation. Two consequences a reader should carry:

- The test adjudicates **the declared sleep current and nothing else**. It cannot see how energy is
  divided among events: moving charge from one declared window to another leaves the residual
  unchanged. A benchmark whose value lies in the per-event split cannot rely on closure to validate it.
- The tolerance is a fraction of the **mean**, while the quantity under test is the **floor**. What a
  given tolerance permits therefore depends on the ratio between them, and it is loosest exactly where
  the floor is smallest relative to the mean, which is where the floor is hardest to measure. A record
  reporting `floor_residual` in amperes makes that visible without arithmetic.
Closure is a self-consistency check, never a ranking metric. The test is pure arithmetic on data
every submitter already has, and it fails loudly when the two terms do not share a boundary, and
when charge that belongs to neither declared window is dropped between them.

For a closed, total activity such as baseline advertising, the accounting scope is the whole capture
and closure runs over all of it. For a scoped activity, closure runs over the included regions only,
and the excluded regions must be explicitly declared so a reader can see what was left out.

What closure does not see: a platform whose idle genuinely varies with rate (for example,
housekeeping triggered per N events rather than per unit time) may still close at the captured rate
if its sleep window averages that activity in; what breaks is the projection to other rates, which
the closure residual cannot detect. Detecting that case requires captures at two or more rates,
which is the demonstrated-fidelity path of section 6.4.

**The tolerance is one percent, provisional.** It has been shown reachable on real hardware (one
platform, an nRF54L15, one build, after measurement artifacts were corrected), and it loosens only
if a second platform shows the procedure sound but one percent practically unreachable.

### 7.4 Finding the window: the procedure is the validation

The specification does not supply the window; it supplies the method. The method carries no
chip-specific constants; it has been exercised on one platform so far:

1. Sweep the event-window length.
2. Compute `closure_residual` at each length.
3. The residual falls and then plateaus. Declare a window on the plateau.

A capture is analyzed, not re-taken: because the raw record is authoritative (section 5), it may be
rescanned with different window parameters until closure passes, and the final parameters become
part of the declared result.

### 7.5 The boundary method, and cross-vendor comparability of the split

**The preferred boundary is functional**, defined by the signal rather than the clock:

> The event window ends when the instantaneous current has returned to within a stated fraction of
> the declared sleep floor and remains there.

It is chip-agnostic and computable from any capture with adequate time resolution. It is preferred
because it never writes a chip-specific duration into a declaration, and it remains **provisional**:
it has been exercised on one vendor's silicon, and the first submission on another vendor's part is
its test.

**A declared fixed window is a conformant alternative**: a conservative post-fiducial window (for
example, a fixed number of milliseconds after event onset) is valid provided it is recorded in the
declaration and closure passes, and it should deliberately include the settling tail, demonstrated
by sitting on the closure plateau of section 7.4 rather than asserted. For entry into a ranked
per-event comparison, that demonstration is required, per the comparability rule below.

**Comparability.** Closure makes composed **totals** comparable across platforms regardless of
boundary choice, because it ties every split back to the same measured average. For the per-event
**split** to be ranked across vendors, the quantity to compare is the **marginal event charge**,
`Q_w − I × t_w`: the window's total charge less the floor's contribution during it. Platforms settle
for different lengths of time, so their windows carry different amounts of floor charge; subtracting
it removes that bias, and the subtraction needs nothing beyond the published primitives. The
marginal event charge is comparable across platforms **only when each window captures its
platform's full settling excess**, which the functional boundary does by construction and a fixed
window does only if it sits on the closure plateau of section 7.4. A ranked per-event comparison
therefore requires tail-inclusive windows on every entry.

### 7.6 A caution the record must carry

The floor must be measured in a window where the platform has genuinely settled, never in a short
gap between events. On the platform tested, a meter averaging the gap at a 100 ms period reported
roughly half of the true idle current, because part of the gap's consumption is supplied from
charge the platform banked during the event, so the meter under-reads the gap; that charge appears
inside the event span instead. The pre-event window (before the radio has ever run) and a long
post-settling window agreed to better than one percent; the short gap did not. The `sleep_window`
field exists so this choice is visible. At periods where sleep carries a real share of the
capture's mean, a floor taken from a window that disagrees with the settled floor fails closure;
at short periods the event window spans nearly the whole period and closure cannot police the
floor at all, which is one more reason the two-rate path of section 6.4 exists.

## 8. The `about.json` record

Every capture carries a generated, machine-readable record. The schema is deliberately structural,
defining a stable container rather than a semantic ontology.

Required top-level members:

| Member | Holds |
|---|---|
| `id` | stable capture identifier |
| `schema_version` | version of this schema |
| `generator` | name and version of the tool that produced the record |
| `units` | unit for each measured quantity |
| `activity`, `platform`, `power` | resolved declarations (see below) |
| `capture` | name, device, timestamps, duration, sampling rate, sample count, voltage statistics, images, build artifacts |
| `events` | count, average duration, duration standard deviation, average energy, energy standard deviation; an episodic capture reports `episodes` analogously |
| `sleep` | average current, current standard deviation, average power |
| `boundary` | the declaration of section 7.2, including the accounting scope |
| `projections` | optional; per named rate: energy per period, energy per day, EM•erald, each labeled as a projection |

`projections` is a convenience emitted by the reference implementation or other conformant tools.
The measured members are the primary result; a consumer that recomputes a projection from the
primitives and the composition rules of section 6 must arrive at the same number.

A resolved declaration is an object carrying an identifier, a title, a declaration source URL,
`factoids` as an ordered list of strings, and `references`.

Factoids stay strings. The schema does not turn activity-specific or platform-specific factoids into
typed fields. A field is promoted to typed JSON only when a real consumer needs it, such as a
frontend or an analysis tool. This keeps the first schema a container contract and lets conventions
harden from use rather than from anticipation.

## 9. Where a new definition goes

As benchmarks are added, the recurring question is which repository owns a given sentence. The rule:

> A definition belongs at the highest layer that does not have to name anything from a lower one.

| Layer | Holds | Home | Churn |
|---|---|---|---|
| 1. Method | what a BlueJoule benchmark *is* | this document | rare |
| 2. Vocabulary | named instances of the nouns | `PEDS` | slow |
| 3. Benchmark | one prescribed activity | `bluejoule-<name>` | rare |
| 4. Measurements | captures of that activity | inside its benchmark repo | constant |

It is a test you run, not a table you look up. Try to write the definition at Layer 1 and see what you
are forced to name:

1. Nothing specific named → Layer 1
2. Had to name a chip, board, instrument, cell or vendor part → Layer 2
3. Had to name an activity → Layer 3
4. Had to name a run, a bench session or a date → Layer 4

The default is Layer 1. Demote only when forced.

Two symmetric checks. If the same sentence has to be written in two repositories, it belongs in the
layer above them. If a sentence here needs an "except on X", then X is a Layer 2 fact and the
exception belongs in X's declaration.

Worked examples:

| New thing | Layer | Where |
|---|---|---|
| A new benchmark (connection, Channel Sounding) | 3 | new `bluejoule-<name>` repository |
| A new platform | 2 | `PEDS/platforms/` |
| A new analyzer | 1 + 2 | suffix rule here, declaration and letter assignment in `PEDS/analyzers/` |
| A new power source | 2 | `PEDS/power/` |
| A vendor quirk ("the DC/DC must be enabled at this operating point") | 2 | that platform's declaration |
| A change to how EM•erald scores | 1 | this document |
| A new named rate | 1 | this document's registry |
| A measurement anomaly in one run | 4 | that capture's notes |

## 10. Conformance

A capture is conformant when it reproduces the activity's **declared over-the-air parameters** (PDU
type, PHY, channel set, TX power, payload, and the configured interval), states its platform, power
source and operating point by reference to declarations, preserves its raw measurement record,
carries the boundary declaration of section 7 (including the accounting scope, with any excluded
regions enumerated) with a closure residual inside the stated tolerance, and carries a generated
record satisfying section 8.

Conformance is defined against the declared parameters rather than against observed timing, because
for advertising activities the Link Layer mandates that per-event timing vary: undirected
advertising events are perturbed by `advDelay`, regenerated per event (section 6), and other
activities carry their own timing variation. Two conformant implementations, and two runs of the
same implementation, cannot be expected to produce identical over-the-air timing, so per-event
timing is not a conformance criterion.

### 10.1 Who rules on conformance

This specification defines what conformance is. It does not yet say who decides a disputed case.

That gap is tolerable while captures are produced by the same parties who wrote the specification.
It stops being tolerable once vendors submit their own results, which is the intended normal path. A
submitter whose capture is rejected, and a reader comparing two entries of different provenance,
both need to know whose determination is final.

**The Foundation designates the party that resolves disputes about whether a submission conforms,
and that designation is recorded in this specification.** Whether that party is a person, a role, or
a documented procedure is the Foundation's to decide. This specification requires only that the
answer be written down and citable, so that a submitter can read it before submitting rather than
discover it during a dispute.

