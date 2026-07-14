---
concept: structure
title: Structure (the Work-Definition Shell)
kind: structure
branch: 2-structure
aka: [operation, op, shift type, shift, wave time, wave, dispatch time, location, site,
      hub, spoke, station, arrival, loadout, what work exists, the schedule shell]

scope_note: |
  ONE concept, four interdependent parts. Operation, Shift Type, Wave Time, and Location
  define what work EXISTS, before anyone is assigned. None stands alone, so they are one
  shell, not four entries. Ranked by consequence: Operation (the shell/hub) > Shift Type
  (load-bearing, the rulebook seam) > Wave Time (thin, one property) > Location (thinnest,
  1:1, no interesting properties). NOT the same as Team: the driver side shares this exact
  hub/details/week-versioned pattern but is a distinct concept, see [[labor]]. Structural
  similarity is not conceptual identity.

disclosure:
  citable:  [OperationName, CustomerCode, ShiftType Description + Duration + CapabilityRequired,
             required category + Quantity, WaveTime StartTime, LocationNickName + Address + type]
  internal: [routing IDs, ColorFamilyID, IconID]
  gated:    [HourlyRate is pay-sensitive -> permission-gated, not open-citable]

shared_pattern: |
  All four parts are a thin hub (ClientID, StartWeekID, Archived) + a details table,
  week-versioned and archivable through the ledger. Versioning semantics live in
  [[coordinate-frame]]; not re-explained per part.

grounding:
  operation:   # the shell / hub
    tblOperation:        { keys: [OperationId], carries: [ClientId, StartWeekID, Archived] }
    tblOperationDetails: { carries: [OperationName, ArrivalLocation, LoadoutLocation, LeadTime, ParentOperationID, CustomerCode], note: ParentOperationID = hub/spoke hierarchy }
  shift_type:  # load-bearing; the rulebook seam
    tblShiftType:          { keys: [ShiftID], carries: [ClientID, StartWeekID, Archived] }
    tblShiftTypeDetails:   { carries: [Description, Duration, HourlyRate, CapabilityRequired] }
    tblShiftTypeRequirements: { carries: [ShiftTypeID, CategoryID, Quantity, IsRequired], role: demands equipment categories -> [[rulebook]] }
  wave_time:   # thin: one real property
    tblWaveTimes:          { keys: [WaveTimeID], carries: [ClientID, StartWeekID, Archived] }
    tblWaveTimeDetails:    { carries: [StartTime] }
    tblWaveTimeAllocation: { carries: [OperationWaveTimeId, DeliveryDate, Count], role: per-date planned counts -> [[preparation]] }
  location:    # thinnest: 1:1, no interesting properties
    tblLocations:       { keys: [LocationID], carries: [ClientID, StartWeekID, Archived] }
    tblLocationDetails: { carries: [LocationNickName, LocationAddress, LocationType, Latitude, Longitude] }
  bridges:
    tblOperationShiftType: { OperationId <-> ShiftType }
    tblOperationWaveTime:  { OperationId <-> WaveTimeId }
  accessors:
    build:      Dash_Build_Operations_Hydrated
    display:    fn_ResolveOperationDisplay / fn_ResolveShiftTypeDisplay
    ledger:     fn_OperationLedgerChanges / fn_ShiftTypeLedgerChanges / fn_WaveTimeLedgerChanges / fn_LocationLedgerChanges

composition: |
  An Operation happens at Locations (arrival + loadout), offers Shift Types
  (tblOperationShiftType), and runs on Wave Times (tblOperationWaveTime). ParentOperationID
  gives the hub-and-spoke hierarchy. The other three parts are what the shell binds.

realized_in:   # structure DEFINES the work; these REALIZE it (definition -> instance arc)
  demand -> filled_work: missions + assignments realize the defined work -> [[preparation]]
  wave_allocation -> planned_counts: tblWaveTimeAllocation.Count feeds demand -> [[preparation]]
  shift_requirement -> equipment: CategoryID + Quantity resolve as categories -> [[rulebook]]

relationships:
  - Operation HAPPENS-AT Location (arrival + loadout) VIA tblOperationDetails
  - Operation OFFERS ShiftType VIA tblOperationShiftType
  - Operation RUNS-ON WaveTime VIA tblOperationWaveTime
  - Operation PARENT-OF Operation VIA ParentOperationID (hub/spoke)
  - ShiftType REQUIRES capability + equipment categories VIA CapabilityRequired + tblShiftTypeRequirements -> [[rulebook]]
  - WaveTime CARRIES planned per-date counts VIA tblWaveTimeAllocation -> [[preparation]]
  - Structure IS-REALIZED-BY missions + assignments -> [[preparation]]
  - All parts VERSIONED-IN [[coordinate-frame]] time (StartWeekID, Archived)
  - Structurally like, but conceptually NOT, Team -> [[labor]]

fill_reality:   # client 7293, verified 2026-07-14
  operations: 9   # hub-and-spoke
  shift_types: 4
  wave_times: 21
  locations: 20

cite: the tblOperation / tblShiftType / tblWaveTimes / tblLocations rows (+ their details) behind a work-definition claim
intents: []
---

## Meaning

Structure is the client's definition of what work EXISTS, before anyone is assigned to it.
It is one shell with four interdependent parts, none of which stands alone.

THE PARTS, BY WEIGHT. Operation is the shell: the top-level unit of work, with a hub-and-spoke
hierarchy (ParentOperationID). Shift Type is the load-bearing part, and the seam to the
rulebook. Wave Time is thin, essentially one property (a start time). Location is thinnest of
all: 1:1, ledger-versioned like the rest but carrying no interesting properties beyond an
address. That ranking is deliberate, so the model knows where the weight sits.

COMPOSITION. An operation happens at locations (an arrival and a loadout), offers shift types,
and runs on wave times. The other three are what the shell binds together.

THE RULEBOOK SEAM. Shift Type is where the demand side meets [[rulebook]]. A shift type carries
CapabilityRequired and, through tblShiftTypeRequirements, demands specific equipment categories
in specific quantities. This is where the need for equipment is born: structure says "this shift
needs a box truck and a scanner," and the rulebook says what those are.

REALIZED IN PREPARATION. Structure is only the definition. The actual filled work, missions and
assignments, realizes it in [[preparation]]; wave allocations carry the planned per-date counts
that become demand. "What work exists" is here; "what work got filled" is there.

DISCLOSURE. Structure is the client's own operation definition, so it is broadly citable: the
operations they run, their shifts, times, and sites. The one guarded field is HourlyRate, which
is pay-sensitive and permission-gated rather than open.
