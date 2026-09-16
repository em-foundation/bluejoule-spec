# Changelog

## v1.2 — record of decisions

Resolved in Bob Frankel's written review of 2026-08-20, against the nine questions of draft v1:

1. Platform definition includes **executing core** (the gatt wording).
2. Charge convention is **total-energy-in-declared-window**, verified by Novel Bits against the
   EM•Scope 26.0 source and confirmed in the review; `event_duration` is the scoring window, never
   the on-air span. Declared-window/tail integration is the first candidate on the reference
   implementation's work list; ordering is deferred to what the first implementation pass needs.
3. Closure tolerance starts at **1%, provisional**; closure runs over the **declared accounting
   scope**, not automatically the whole capture.
4. Ranked comparisons require a **projection-fidelity statement** (demonstrated two-rate, or
   inherited default no smaller than the demonstrated range); sub-uncertainty differences present
   as indistinguishable; direct captures outrank projections.
5. The **functional boundary is the preferred method, provisional**; a declared fixed
   post-fiducial window is a conformant alternative when recorded and closure passes. The
   requirement that ranked per-event entries demonstrate tail inclusion via the section 7.4
   plateau is Novel Bits' tightening.
6. **Analyzers move to PEDS declarations** (`PEDS/analyzers/`), probably yes and
   not a blocker; lightweight; the suffix rule stays here; BlueJoule is never an
   approved-instrument program.
7. **Named rates stay mechanical** (`adv-1s`); scenario names live at the presentation layer.
8. **Baseline ADV stays non-connectable, non-scannable**; scannable advertising becomes a
   peer-dependent activity with declared scanner load (quiet, controlled single-scanner,
   saturated) and controlled/observed/uncontrolled environment labeling.
9. **Episodes enter now as a lightweight first-class unit** with their own composition mode;
   scoped activities such as the current GATT benchmark declare included/excluded regions.

Still provisional, by design: the functional boundary and the 1% tolerance (both single-silicon
until a second vendor's part is submitted), and the episodic composition mode (hardens with the
first episodic benchmark revision).

**Introduced in this version beyond the decisions recorded above.** Four provisions go further than
folding in the answers, and each is attributed here:

- **The generalized closure formula in section 7.2** (`abs((Q_w + I × (T_s − t_w)) / T_s − mean) /
  mean` over the included duration `T_s`). Extending closure to scoped and episodic captures was
  Bob Frankel's ruling; this arithmetic form is Novel Bits'. It reduces exactly to the per-period form when the
  period is taken as the included duration over the event count. It does **not** reduce to it when
  the period is the measured mean inter-event interval over a scope that is not an exact integer
  number of periods: on a ten-second trimmed span holding 9.95 periods, the two differ by roughly 0.3 to 0.5 points depending on the capture, against a 1% tolerance. The scoped form is the self-consistent one and reads lower, so this
  is a correction rather than a restatement, and published residuals computed the old way should be
  recomputed.
- **The episodic composition formula in section 6.2.** Episodes becoming a first-class unit was Bob Frankel's
  ruling; the formula is Novel Bits'.
- **The ranked-entry plateau requirement in section 7.5.** Novel Bits'. Ranked per-event entries must
  demonstrate tail inclusion via the section 7.4 plateau, tightening the declared-fixed-window
  alternative recorded in decision 5.
- **Section 10.1, who rules on conformance.** Novel Bits'. It supplies no answer and decides nothing;
  it records that section 10 defines conformance without naming who adjudicates it, and requires the
  Foundation's designation to be written down. Raised because the memo of 2026-08-28 makes vendor
  submission the normal path, which is the point at which the omission starts to matter.
