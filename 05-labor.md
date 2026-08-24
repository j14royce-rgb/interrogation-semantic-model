---
concept: labor
title: The Driver (the Supply Side)
kind: entity
branch: 3-labor
aka: [driver, DA, associate, walker, team, LMDP, labor, headcount, roster,
      availability, time off, preference, "who's available", "who can",
      "who wants", "why wasn't he scheduled", qualified, eligible]

scope_note: |
  THE SUPPLY SIDE — who exists, what they hold, and what they want. It
  complements [[structure]] (the demand side); the two meet at the seat.
  WHAT a document proves and WHAT a badge demands is [[eligibility]]; this
  concept holds the driver's END of that — the documents they carry and the
  preferences they set. The driver is also an ACTOR when they are the one
  asking, see [[actor]] — but the portal has no driver asker.

disclosure:
  citable:  [FirstName, LastName, active status, documents held + expiry, badge
             qualification, preference values, time-off status, weekly hours]
  internal: [routing IDs, TargetID polymorphism, lane/StatusType mechanics]
  gated:    [MobilePhone, Email, DriverImage, UniqueId are PII — permission-gated]

shared_pattern: |
  Thin hub (ClientID, StartWeekId, Archived) + details, week-versioned via
  [[coordinate-frame]]. Active-driver gate = Archived = 0.

grounding:
  identity:
    tblLMDP:                    { keys: [LMDPID], carries: [ClientID, StartWeekId, Archived], gate: "Archived = 0 is the headcount gate, not 'scheduled this week'" }
    tblLMDPDetails:             { carries: [FirstName, LastName, MobilePhone, Email, UniqueId, DriverImage] }
    tblLMDPContactVerification: { carries: [ContactKind, VerifiedAt, Method] }
  can:      # the ONLY absolute. Everything else is a scale.
    the_gate:  fn_ResolveSeatGate    # per driver, per seat, per day; verdict travels as MEMBERSHIP
    holdings:  tblDocument (OwnerType='Driver', OwnerID=LMDPID) -> [[eligibility]] / [[documents]]
    satisfied: fn_LMDPCapabilitySatisfied
  wants:    # ONE scale, twelve axes, THREE authorities
    tblLMDPPreferenceAxis: { carries: [ClientID, LMDPID, AxisID, TargetID, Archived], note: "TargetID is POLYMORPHIC — its meaning is the AXIS's, and an orphan check will not catch a stale id" }
    tblPreferenceType:     { role: the axis dictionary — twelve axes, each with its own ledger StatusType }
    tblPreferenceValueSemantics: { role: the six values and what each one MEANS }
    values:
      0: { label: Disabled,         class: Ineligible,       set_by: manager }
      1: { label: Override-only,    class: OverrideRequired, set_by: "driver REQUESTS, manager approves via the Inbox" }
      2: { label: "I'd rather not", class: Eligible,         set_by: driver }
      3: { label: "That's fine",    class: Eligible,         set_by: driver, default: true }
      4: { label: "Yes, please",    class: Eligible,         set_by: driver }
      5: { label: Must,             class: Forced,           set_by: manager }
  availability_component:   # Day · StandBy · OT · WeeklyHours · Location
    axes: [1 Day (ST730), 2 StandBy (ST731), 3 OT (ST732), 5 WeeklyHours (ST704), 10 Location (ST733)]
    note: "WeeklyHours hangs off the DRIVER, not off a pair — which is why it has no tblLMDPPreferenceAxis rows and is not a defect"
  badge_component:          # Vehicle · Role · Delivery
    axes: [11 Vehicle (ST734), 12 Role (ST735), 13 Delivery (ST736)]
    note: "the badges the driver qualifies for, and the preference set on each"
  availability_exception:
    tblTimeOffRequest: { carries: [DateStart, DateEnd, RequestStatus, ReasonTypeID, Impact, ResolvedBy, ResponseReason], role: a dated absence with an adjudication — NOT a value on the scale }
  accessors:
    full_state: Proc_Hub_FatRow_Hydrator     # THE driver accessor; never hand-decode lanes
    preference: fn_LMDPPreference(@ClientID,@WeekID)
    roster:     Dash_TeamRoster_Hydrated     # 10 recordsets, one per grain
    surfaces:   Dash_LMDP_Home_Hydrated / Dash_LMDP_Profile_Hydrated
    ledger:     fn_LMDPLedgerChanges
    display:    fn_ResolveLMDPDisplay / fn_ResolveDriverStyle

gotcha: |
  The preference VALUE is not on tblLMDPPreferenceAxis. That table is the
  ANCHOR — (driver, axis, target); the value is a ledger status on the axis's
  own StatusType, decoded through fn_LMDPPreference or the fat-row hydrator.
  Read a driver through Proc_Hub_FatRow_Hydrator, never by joining the anchor
  raw. See [[ledger]].

relationships:
  - Driver IS-A actor (login Type L) -> [[actor]]
  - Driver HOLDS documents; what those PROVE is [[eligibility]]'s
  - Driver is ADMITTED-OR-NOT by fn_ResolveSeatGate, and the verdict travels as MEMBERSHIP
  - Driver WANTS on one six-value scale across twelve axes, three authorities
  - Preference consolidates into AVAILABILITY and BADGES
  - Driver availability EXCEPTED-BY time off (dated, adjudicated)
  - Driver (supply) MEETS demand ([[structure]]) at the seat
  - Driver state STORED-IN [[ledger]]; weekly metrics -> [[output]]

fill_reality:   # client 7293, verified 2026-08-23
  drivers_total: 139
  drivers_active: 117
  documents_held: 638
  preference_pair_rows: 2704     # across 8 axes
  weekly_hours_rows: 1165        # ST 704, on the driver, 1099 drivers
  time_off_requests: 25          # 8 pending · 15 resolved · 2 other
  dormant_or_superseded:
    - "4 ShiftType (ST23) — SUPERSEDED. 3,948 authored ledger rows, zero pair rows, pinned for migration."
    - "6 WaveTime (ST738) — 468 live pair rows, but NOT adjudicated in the engine: expressed and never consumed."
    - "7 Language (ST702), 8 RosterStatus (ST31) — ledger rows, zero pair rows."
    - "op-auth (tblLMDPOperationAuthorization, 10,652 rows) and ST 35 — no longer a gate."
    - "ST 4 shift-type enablement and ST 700 vehicle certification — ZERO rows. Both mechanisms retired."

cite: tblLMDP.LMDPID + the tblDocument row behind a holding, or the tblLMDPPreferenceAxis row and its ledger status behind a preference
intents: []
---

## Meaning

**THE SUPPLY SIDE.** The driver is who exists, what they hold, and what they
want. [[structure]] says what work exists; this says who could do it; they meet
at the seat.

**ONE ABSOLUTE, AND EVERYTHING ELSE IS A SCALE.** There is exactly one true
*can't*: a document the driver does not hold, which leaves a slot unmet and the
seat unfillable. That is [[eligibility]]'s wall and no one can override it.
**Everything else — every day, every badge, every location — is one six-value
scale.** An older reading of this model called it hard-versus-soft. It is not.
It is one scale with a movable bar.

**THE SCALE HAS THREE AUTHORITIES, AND THAT IS THE REAL DISTINCTION.** The
driver alone sets **2, 3, 4** — *I'd rather not*, *That's fine*, *Yes, please*.
The driver **requests 1** and a manager grants it **through the Inbox**; it is
an approval, not a setting. The manager alone sets **0** (*Disabled*, the only
preference value classed Ineligible) and **5** (*Must*, Forced). So when a
driver was not scheduled, the question is not "was it hard or soft" — it is
**which value, and who was entitled to set it.**

**TWO COMPONENTS.** Preference consolidates into **availability** — the seven
days, overtime, standby, weekly hours, and location — and **badges** — the
transport, role and service badges the driver qualifies for and the preference
set on each. Location belongs to availability because it is really about the
**anchor**: everyone qualifies for the dispatch, so a preference there says
nothing.

**WEEKLY HOURS HANGS OFF THE DRIVER, NOT OFF A PAIR.** Eight axes hang off a
driver-and-target pair that must be minted; weekly hours hangs off the driver,
who always exists. Same seven columns, two existence rules, and nothing in the
payload saying which. A weekly-hours preference with no pair row is correct,
not missing.

**THE POLYMORPHIC TARGET.** `TargetID` means whatever its axis says it means — a
day, a badge, a location. **An orphan check will not catch a stale one**: a
service preference left pointing at target 1 resolves cleanly to badge 1, which
is Driver. Only the KIND catches it.

**TIME OFF IS AN EXCEPTION, NOT A PREFERENCE.** A time-off request is a dated
absence with a status and an adjudication, not a value on a scale. Approved time
off makes a driver unavailable; it does not make them ineligible. The two must
never be reported as the same kind of "no".

**EXPRESSED AND NEVER CONSUMED.** WaveTime carries 468 live preferences and **is
not adjudicated by the engine**. Drivers can state it and nothing reads it. That
is a real gap between what the product asks people for and what it uses, and the
portal must never imply a wave preference influenced an outcome.

**RETIRED MECHANISMS.** Shift-type enablement (ST 4) and vehicle certification
(ST 700) are both empty — what a driver may operate is derived from documents
now, never stored as a flag, so "why can't he take the box truck" resolves to a
missing or expired document rather than a flag someone forgot to set. Operation
authorization still holds 10,652 rows and gates nothing.

**THE LEDGER GOTCHA.** The anchor table holds (driver, axis, target); the VALUE
is a ledger status on that axis's own StatusType. Read a driver through
Proc_Hub_FatRow_Hydrator, never by joining the anchor raw — that is how lane
drift and wrong answers happen. See [[ledger]].

**DISCLOSURE.** Name and active status are citable to a manager. Phone, email,
image and the unique id are PII and permission-gated, not open-citable.
