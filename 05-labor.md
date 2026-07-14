---
concept: labor
title: Team (the Driver, and the Supply Side)
kind: entity
branch: 3-labor
aka: [driver, DA, associate, walker, team, LMDP, labor, headcount, roster,
      who's available, who can, who wants, certified, eligible]

scope_note: |
  One concept, one entity: the driver (tblLMDP), with many lanes hanging off it. This is the
  SUPPLY side; it complements [[structure]] (the demand side), and the two meet in
  [[preparation]]. The driver is also an ACTOR (login Type L, see [[actor]]). Two axes govern
  everything: CAN (eligibility, a non-overridable "can't") vs WANTS (preference, an overridable
  "don't," even when set fully off at 0). Structurally these are the same ledger values; the
  distinction is overridability, and it must stay isolated. Distinct from [[structure]] despite
  the identical hub/details pattern.

disclosure:
  citable:  [FirstName, LastName, active status, eligibility/certifications, preferences,
             time-off status, weekly preference scores]
  internal: [routing IDs]
  gated:    [MobilePhone, Email, DriverImage, UniqueId are PII -> permission-gated, not open]

shared_pattern: |
  Thin hub (ClientID, StartWeekId, Archived) + details, week-versioned via [[coordinate-frame]].
  Active-driver gate = Archived = 0.

grounding:
  identity:
    tblLMDP:                    { keys: [LMDPID], carries: [ClientID, StartWeekId, Archived], gate: Archived=0 = active }
    tblLMDPDetails:             { carries: [FirstName, LastName, MobilePhone, Email, UniqueId, DriverImage] }
    tblLMDPContactVerification: { carries: [ContactKind, VerifiedAt, Method] }
  can_eligibility:   # HARD constraints; gates [[preparation]], reads [[rulebook]]
    tblLMDPOperationAuthorization: { LMDPID -> OperationID, note: ST35 operation eligibility gate }
    vehicle_cert:                  ST700 vehicle certification (ledger) -> fn_ResolveLMDPCapability / fn_LMDPHasVehicleCert
    tblLMDPShiftType:              { LMDPID -> ShiftType, note: ST4 shift-type enablement }
    doc_gate:                      fn_LMDPDocGate / fn_LMDPShiftDocsStatus -> [[documents]]
    matrix:                        fn_LMDPEligibilityMatrix
  wants_preference:  # SOFT; feeds [[preparation]] optimization scores
    packed_prefs:        ST500 (ledger, base-4: Mon-Sun + OT + Standby + WeeklyHours) -> fn_GetStatus_Decoded
    shift_wave_pref:     tblLMDPShiftType (ST23) / tblLMDPWaveTime (ST24/25) anchors
    tblLMDPSchedulePref: { carries: [TargetCode], note: thin; real values live in the ledger }
    tblLMDPInterest:     { carries: [PreferenceValue, Active] }
  availability:
    tblTimeOffRequest: { carries: [DateStart, DateEnd, RequestStatus, ReasonTypeID, Impact, ResolvedBy, ResponseReason], role: availability exception }
  accessors:
    full_state: Proc_Hub_FatRow_Hydrator     # THE driver accessor; never hand-decode lanes
    surfaces:   Dash_LMDP_Home_Hydrated / Dash_LMDP_Profile_Hydrated / Dash_Build_Team_Hydrated / Dash_Team_Main_Hydrated
    ledger:     fn_LMDPLedgerChanges
    display:    fn_ResolveLMDPDisplay / fn_ResolveDriverStyle

gotcha: |
  The bridge tables (tblLMDPShiftType, tblLMDPWaveTime, tblLMDPOperationAuthorization,
  tblLMDPSchedulePref) are ANCHORS. The actual on/off and preference VALUES are ledger status
  (tblStatusChange), decoded by the fat-row hydrator. Read the driver through
  Proc_Hub_FatRow_Hydrator, not by joining these tables raw. See [[ledger]].

realized_in:   # consumers of the driver
  eligibility -> assignment legality: gates who can fill work        -> [[preparation]]
  preference  -> optimization score: feeds scheduling               -> [[preparation]]
  weekly scores -> tblLMDPWeeklyMetrics (TotalHours, pref scores)    -> [[output]]
  documents_held -> tblLMDPDocument                                  -> [[documents]]

relationships:
  - Driver IS-A actor (login Type L) -> [[actor]]
  - Driver CAN do work VIA eligibility (op auth ST35 + vehicle cert ST700 + shift-type ST4 + doc gates) -> reads [[rulebook]]
  - Driver WANTS VIA preferences (ST500 packed + ST23/24/25 + interest) -> feeds [[preparation]]
  - Driver availability EXCEPTED-BY time-off (tblTimeOffRequest)
  - Driver (supply) MEETS demand ([[structure]]) in [[preparation]]
  - Driver state STORED-IN [[ledger]], read via Proc_Hub_FatRow_Hydrator
  - Weekly metrics -> [[output]]; documents held -> [[documents]]

fill_reality:   # client 7293, verified 2026-07-14
  drivers_total: 135
  drivers_active: 117   # Archived = 0
  time_off_requests: 14

cite: tblLMDP.LMDPID + the tblStatusChange rows behind any eligibility or preference claim
intents: []
---

## Meaning

The driver is the supply side: who can do the work, and what they prefer. It complements
[[structure]], which is the work itself, and the two meet in [[preparation]], where supply
is assigned to demand.

ONE ENTITY, MANY LANES. Everything hangs off a single driver (tblLMDP): identity, eligibility,
preference, availability, documents, and metrics. A driver is active when Archived = 0. That is
the headcount gate, not "assigned this week."

CAN vs WANTS. Two axes govern the driver, and the real difference is not soft-versus-hard, it is
OVERRIDABLE versus not. CAN is eligibility: operation authorization, vehicle certification,
shift-type enablement, and the document gates. A failed eligibility is a true "can't" the manager
cannot override; an uncertified driver simply cannot take a box truck. It reads [[rulebook]] and
gates assignment. WANTS is preference: day, hours, OT, standby, shift-type, and wave. A preference
is a "don't schedule me," and the manager can override it, even when it is set fully off (a 0
preference behaves like disabled but is still an overridable request, not a prohibition). So when
a driver is not scheduled, the question that matters is which "no" it was: the non-overridable
can't, or the overridable don't. Structurally they are the same ledger values; in meaning they
must be kept apart.

THE LEDGER GOTCHA. The bridge tables are only anchors. The real on/off states and preference
values live in the ledger as status rows, decoded by the fat-row hydrator. Read a driver through
Proc_Hub_FatRow_Hydrator, never by joining the anchor tables raw; that is how lane drift and
wrong answers happen. See [[ledger]].

ALSO AN ACTOR. The same driver logs into the user app (login Type L). When the driver is the
subject of a question they are Labor; when they are the one asking, they are the actor in
[[actor]].

DISCLOSURE. Name and active status are citable to a manager. Personal contact details, phone,
email, image, and the unique ID, are PII and permission-gated, not open-citable.
