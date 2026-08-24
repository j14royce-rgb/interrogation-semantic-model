---
concept: structure
title: Structure (the Demand Side)
kind: structure
branch: 2-structure
aka: [mission type, mission, seat, crew, run, shift, shift type, operation,
      dispatch, anchor, location, site, hub, wave, wave time, arrival,
      "what work exists", "how many people", "who is needed", the schedule shell]

scope_note: |
  THE DEMAND SIDE. What work exists and who is REQUIRED to be in it, before
  anyone is assigned. Four levels — mission type -> seat -> mission ->
  mission seat — plus place, which is half of what a seat's ReportToKind
  means. NOT who fills it (that is assignment). NOT what a badge means
  (that is [[eligibility]]). NOT what an asset is (that is [[assets]]).
  "Operation" and "shift type" are in `aka` because managers still say them;
  neither exists.

disclosure:
  citable:  [mission type name, dispatch + anchor location Name/Address, seat Ordinal,
             role badge Label, Hours, StartOffsetMinutes, Quantity, ReportToKind,
             required AssetCategory + Quantity, DeliveryDate, seat counts]
  internal: [ColorFamilyID, MissionTypeDetailID / SeatDetailID routing, ST 33/34 lane mechanics]
  gated:    [HourlyRate is pay-sensitive — permission-gated, never open-citable]

grounding:
  definition:
    tblMissionType:        { keys: [MissionTypeID], carries: [ClientID, StartWeekID, Archived], role: thin hub }
    tblMissionTypeDetails: { carries: [DispatchLocationID, TransportBadgeID, IsAnchored, ServiceBadgeID, ColorFamilyID], role: the identity four-tuple + the two badges it attaches }
    tblSeat:               { keys: [SeatID], carries: [MissionTypeID, Ordinal, StartWeekID, Archived] }
    tblSeatDetails:        { carries: [RoleID, HourlyRate, Hours, StartOffsetMinutes, Quantity, ReportToKind], versioned: ST 33 detail / ST 34 live-flag }
    tblSeatRequirements:   { carries: [SeatID, AssetCategoryID, Quantity], role: THE MISSION KIT. No week — config, not ledgered. }
  instance:
    tblMission:            { keys: [MissionID], carries: [MissionTypeID, DeliveryDate, OverstaffingType, AnchorLocationID] }
    tblMissionSeat:        { keys: [MissionSeatID], carries: [MissionID, SeatID, Ordinal, SeatType, IsAdded, IsReduced], role: THE ATOM. One row per person required. }
  place:
    tblLocation:           { keys: [LocationID], carries: [ClientID, StartWeekID, Archived] }
    tblLocationDetails:    { carries: [Name, Address, City, State, Zip, Latitude, Longitude, LocationType, ParentLocationID], note: LocationType 1 dispatch · 2 off-site · 3 anchor; ParentLocationID tethers an anchor to its dispatch root }
  wave:
    tblWaveTime:           { keys: [WaveTimeID], carries: [ClientID, StartWeekID, Archived] }
    tblWaveTimeDetails:    { carries: [StartTime] }
    tblMissionTypeWaveTime:{ role: a bridge only — see WAVE TIME IS NOT LOAD-BEARING HERE }
  accessors:
    name:      fn_ResolveMissionTypeName(@ClientID,@WeekID)    # vehicle | service [| Anchored]
    display:   fn_ResolveMissionTypeDisplay
    seats:     fn_MissionTypeSeats(@MissionTypeID,@WeekID)     # + drives / delivers / isOperator
    holes:     fn_ResolveMissionSeats
    place:     fn_ResolveLocationDisplay
  write_doors:
    Op_MissionType_Submit:      the four-tuple; refuses a duplicate on it, never on the name
    Op_Seat_Submit:             the crew; refuses a non-driving role at ordinal 1
    Op_SeatRequirement_Submit:  the mission kit, one category one seat one call
    Op_MissionCard_Submit:      instantiation — where the multiplication happens
    Op_Mission_Breathe:         add or reduce a seat on a live mission
    Op_MissionTypeWaveTime_Submit

identity: |
  A mission type IS the four-tuple (dispatch location, transport badge,
  service badge, anchored). Op_MissionType_Submit refuses a duplicate on the
  tuple, never on the name — the name is DERIVED and never stored.

the_multiplication: |
  A seat of quantity N mints N tblMissionSeat rows at instantiation. The
  multiplication happens ONCE, at the mission card, and never again. A
  requirement applies once per mission-seat ROW: 25 missions is 25 scanners
  because there are 25 supervisor rows.

relationships:
  - Mission type BELONGS-TO a client and is VERSIONED-IN [[coordinate-frame]] time
  - Mission type ATTACHES a transport badge and a service badge -> [[eligibility]]
  - Seat BELONGS-TO a mission type and ATTACHES a role badge -> [[eligibility]]
  - Seat DEMANDS asset categories VIA tblSeatRequirements -> [[assets]]
  - Mission INSTANTIATES a mission type on a DATE
  - Mission seat IS-MINTED-FROM a seat, one row per person required
  - Seat REPORTS-TO a place kind (1 dispatch · 2 off-site · 3 anchor)
  - An ANCHORED mission type takes its place from tblMission.AnchorLocationID, not the dispatch
  - Structure DECLARES; [[eligibility]] and [[assets]] SATISFY; assignment FILLS

fill_reality:   # client 7293, verified 2026-08-23
  mission_types: 4        # all one dispatch; 8800048 and 8800051 differ ONLY by transport badge
  seats: 6
  seat_requirements: 0    # empty by ruling — the 48 legacy rows named shift types that exist nowhere
  missions: 613
  mission_seats: 1725     # 1 to 31 per mission
  locations: 4            # 1 dispatch (Manhattan Hub) · 1 off-site (Gowanus Lot) · 2 anchors (Upper East Side, Harlem)
  wave_times: 4           # 5 mission-type bridges
  retired_2026-08-23: [tblWaveTimeAllocation, tblOperationShiftType]   # zero readers, dropped

cite: the tblMissionType / tblSeat rows behind a declaration, and the tblMission / tblMissionSeat rows behind a dated one
intents: []
---

## Meaning

**THE DEMAND SIDE.** Structure says what work exists and how many people it
needs. It never says who. A mission with an empty seat is fully described here;
filling it is somebody else's concept.

**FOUR LEVELS, AND THE SEAM IS IN THE MIDDLE.** Mission type and seat are
**definition** — week-versioned, edited in the Rule Book, true until changed.
Mission and mission seat are **instance** — a date, and one row per person
required. The seam between them is instantiation.

**IDENTITY IS A FOUR-TUPLE, NOT A NAME.** A mission type is *(dispatch,
transport badge, service badge, anchored)*. The name is derived and never
stored — `vehicle | service [| Anchored]` — so two types can read alike and
still be distinct. On the demo client, two of the four differ only by transport
badge. Anything that identifies a mission type by its name is identifying it by
something the database does not keep.

**THE SEAT MAKES TWO DEMANDS, AND THEY ARE PARALLEL.** It attaches a **role
badge**, satisfied by documents in [[eligibility]]; and it demands **asset
categories**, satisfied by instances in [[assets]]. **Structure owns the
declaration in both cases and the satisfaction in neither.** The seat is the
subject; the badge and the category are both objects.

**THE MULTIPLICATION HAPPENS ONCE.** A seat of quantity N mints N mission-seat
rows at instantiation, and nothing multiplies again. A requirement of one scanner
on a supervisor seat is one scanner *per supervisor row*. The seat's quantity
must never meet a requirement's quantity — putting the two numbers side by side
invites a multiplication that has already happened.

**PLACE IS NOT COSMETIC.** A seat reports to a *kind* of place — dispatch,
off-site, or anchor — and an **anchored** mission type takes its actual place
from the mission's own `AnchorLocationID` rather than the type's dispatch. That
is why the grid can roll up every anchor while a modal scopes to the one
clicked, and why a cell key carries a location at all. An anchor is tethered to
its dispatch root by `ParentLocationID`; it is a place under a place, not a
place beside one.

**WAVE TIME IS NOT LOAD-BEARING HERE.** A wave time is a start time, bridged to
mission types, and it makes no difference to demand: nothing about who is
required changes with the wave. It is a ledgered component and belongs with the
components, not in this concept. Its allocation table was retired 2026-08-23
with no readers.

**WHAT STRUCTURE DOES NOT KNOW.** Whether anyone can fill a seat
([[eligibility]]), whether the equipment exists ([[assets]]), who is in it
(assignment), or whether they want to be ([[preferences]]).
