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
    the_gate:     fn_ResolveSeatGate(@ClientId,@WeekId,@FirstDayOfWeek,@OverrideAvailability,@OverrideBadges)   # per driver, per seat, per day; verdict travels as MEMBERSHIP
    holdings:     tblDocument (OwnerType='Driver', OwnerID=LMDPID) -> [[eligibility]] / [[documents]]
    supply:       fn_LMDPDocumentSupply(@ClientID,@AsOf)           # what the driver holds, valid today
    satisfied:    fn_LMDPCapabilitySatisfied(@ClientID,@WeekID,@AsOf)   # which BADGES the driver's paperwork satisfies; the name predates the 2026-09-03 collapse, the body reads fn_BadgeDocuments
    requirements: fn_BadgeDocuments(@ClientID,@WeekID)              # the badge -> requirement -> document table, read from [[eligibility]]
  wants:    # ONE scale, eleven axes in the dictionary, THREE authorities
    tblLMDPPreferenceAxis: { carries: [ClientID, LMDPID, AxisID, TargetID, Archived], note: "TargetID is POLYMORPHIC — its meaning is the AXIS's, and an orphan check will not catch a stale id" }
    tblPreferenceType:     { keys: [PreferenceTypeID], carries: [PreferenceType, StatusTypeID], role: the axis dictionary — each axis has its own ledger StatusType }
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
  badge_component:          # one axis per badge category
    axes: [11 Vehicle (ST734) = the TRANSPORT badge, 12 Role (ST735) = the ROLE badge, 13 Delivery (ST736) = the SERVICE badge]
    note: "the badges the driver qualifies for, and the preference set on each. The dictionary's words (Vehicle, Delivery) are older than the badge words (Transport, Service); a manager says the badge word."
  availability_exception:
    tblTimeOffRequest: { carries: [DateStart, DateEnd, RequestStatus, ReasonTypeID, Impact, ResolvedBy, ResponseReason], role: a dated absence with an adjudication — NOT a value on the scale }
  accessors:
    preference: fn_LMDPPreference(@ClientID,@WeekID)    # THE preference read; decodes the ledger lane per axis
    roster:     Dash_TeamRoster_Hydrated                # 10 recordsets, one per grain
    profile:    Dash_LMDP_Profile_Hydrated              # the driver's full page
    surfaces:   Dash_LMDP_Home_Hydrated / Dash_LMDP_Qualifications_Hydrated / Dash_LMDP_Documents_Hydrated / Dash_LMDP_TimeOff_Hydrated
    ledger:     fn_LMDPLedgerChanges
    display:    fn_ResolveLMDPDisplay / fn_ResolveDriverStyle

gotcha: |
  The preference VALUE is not on tblLMDPPreferenceAxis. That table is the
  ANCHOR — (driver, axis, target); the value is a ledger status on the axis's
  own StatusType, decoded through fn_LMDPPreference. Read a driver's preferences
  through fn_LMDPPreference or the profile/roster hydrators, never by joining
  the anchor raw. See [[ledger]].

relationships:
  - Driver IS-A actor (login Type L) -> [[actor]]
  - Driver HOLDS documents; what those PROVE is [[eligibility]]'s
  - Driver is ADMITTED-OR-NOT by fn_ResolveSeatGate, and the verdict travels as MEMBERSHIP
  - Driver WANTS on one six-value scale across the axes, three authorities
  - Preference consolidates into AVAILABILITY and BADGES
  - Driver availability EXCEPTED-BY time off (dated, adjudicated)
  - Driver (supply) MEETS demand ([[structure]]) at the seat
  - Driver state STORED-IN [[ledger]]; weekly metrics -> [[output]]

fill_reality:   # client 7293, verified 2026-09-15
  drivers_total: 162
  drivers_active: 140
  documents_held: 900            # driver-owned, uncancelled
  preference_pair_rows: 4225     # across the pair axes
  time_off_requests: 27
  dormant_or_superseded:
    - "4 ShiftType (ST23) — GONE from the dictionary. The shift type was retired with the mission-type rebuild."
    - "6 WaveTime (ST738) — still in the dictionary, NOT adjudicated in the engine: expressed and never consumed."
    - "7 Language (ST702), 8 RosterStatus (ST31) — in the dictionary, zero pair rows."
    - "Operation authorization (tblLMDPOperationAuthorization) — table DROPPED 2026-08. Nothing gates on it."
    - "ST 4 shift-type enablement and ST 700 vehicle certification — retired. What a driver may operate is derived from documents."

cite: tblLMDP.LMDPID + the tblDocument row behind a holding, or the tblLMDPPreferenceAxis row and its ledger status behind a preference
intents: []
---

## Meaning

**THE SUPPLY SIDE.** The driver is who exists, what they hold, and what they
want. [[structure]] says what work exists; this says who could do it; they meet
at the seat.

**ONE ABSOLUTE, AND EVERYTHING ELSE IS A SCALE.** There is exactly one true
*can't*: a document the driver does not hold, which leaves a badge requirement
unmet and the seat closed to them. That is [[eligibility]]'s wall and no one can
override it. **Everything else — every day, every badge, every location — is one
six-value scale.** An older reading of this model called it hard-versus-soft. It
is not. It is one scale with a movable bar.

**THE SCALE HAS THREE AUTHORITIES, AND THAT IS THE REAL DISTINCTION.** The
driver alone sets **2, 3, 4** — *I'd rather not*, *That's fine*, *Yes, please*.
The driver **requests 1** and a manager grants it **through the Inbox**; it is
an approval, not a setting. The manager alone sets **0** (*Disabled*, the only
preference value classed Ineligible) and **5** (*Must*, Forced). So when a
driver was not scheduled, the question is not "was it hard or soft" — it is
**which value, and who was entitled to set it.**

**TWO COMPONENTS.** Preference consolidates into **availability** — the seven
days, overtime, standby, weekly hours, and location — and **badges** — the
Transport, Role and Service badges the driver qualifies for and the preference
set on each. Location belongs to availability because it is really about the
**anchor**: everyone qualifies for the dispatch, so a preference there says
nothing.

**WEEKLY HOURS HANGS OFF THE DRIVER, NOT OFF A PAIR.** The pair axes hang off a
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

**EXPRESSED AND NEVER CONSUMED.** WaveTime is still an axis in the dictionary and
**is not adjudicated by the engine**. Drivers can state it and nothing reads it.
That is a real gap between what the product asks people for and what it uses,
and the portal must never imply a wave preference influenced an outcome.

**RETIRED MECHANISMS.** Shift-type enablement, vehicle certification and
operation authorization are all gone — what a driver may operate is derived from
documents now, never stored as a flag, so "why can't he take the box truck"
resolves to a missing or expired document rather than a flag someone forgot to
set.

**THE LEDGER GOTCHA.** The anchor table holds (driver, axis, target); the VALUE
is a ledger status on that axis's own StatusType. Read it through
fn_LMDPPreference, never by joining the anchor raw — that is how lane drift and
wrong answers happen. See [[ledger]].

**DISCLOSURE.** Name and active status are citable to a manager. Phone, email,
image and the unique id are PII and permission-gated, not open-citable.
