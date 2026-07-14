---
concept: assets
title: Assets (the Instance Layer: Owned, Held, Conditioned)
kind: entity
branch: 4-assets
aka: [asset, equipment, vehicle, van, truck, EDV, device, scanner, tool, uniform, item,
      fleet, inventory, serial number, in service, online, grounded, who has it,
      how many do we have, what's available]

scope_note: |
  One concept: the actual equipment items that realize [[rulebook]] kinds. Two storage modes,
  frozen by the rulebook (serialized item vs bulk pool), and two orthogonal facts that must
  never be conflated: OWNERSHIP (what the client has) vs POSSESSION (who holds it now). The
  events that move or change assets (handover, issuance, inspection) are [[execution]]; this
  concept is the item and its current truth, not the events.

disclosure:
  citable:  [SerialNumber, CategoryName, LocationNickName, current status (Online/Grounded/...),
             ownership count, current holder, condition grade]
  internal: [routing IDs, HandshakeType codes]

grounding:
  serialized:
    tblAssetInstance: { keys: [AssetInstanceID], carries: [AssetCategoryID, LocationID, SerialNumber, StatusID], note: one row per physical item }
  bulk:
    tblAssetInventory: { keys: [InventoryID], carries: [ClientAssetCategoryID, LocationID, Quantity, StateID, VariantID], note: counted pool, not instanced }
    tblAssetBulkVariant / tblAssetBulkVariantValue: { variants of a bulk category (e.g. sizes) + property values }
  state_condition:   # 5 operational states + class-grained reasons
    tblAssetStatus:            { ONLINE, MAINTENANCE, GROUNDED, OFFLINE, RETIRED }
    tblAssetStateChange:       { AssetInstanceID, FromStatusID -> ToStatusID, ReasonID, Note, EventDateTime; the asset's own history }
    tblAssetStateChangeReason: { Code, Label, ImpliesOffline, IsPinIndicator (Under-Review pin), AppliesToClassID; class-grain }
    condition:                 tblVehicleConditionSnapshot (OverallGrade, ActivePinCount) -> [[execution]]
  possession:        # CURRENT custody; assigned through [[preparation]]/[[execution]]
    tblAssetAssignment:     { AssignmentScheduledID -> AssetInstanceID, IsActive, StartDate, EndDate, note: serialized custody }
    tblAssetBulkAssignment: { ClientAssetCategoryID, IssuedFromLocationID, IsActive, ClosedState, note: bulk custody }
    tblAssetBundleMember:   { ParentInstanceID -> MemberInstanceID / MemberQuantity, note: a vehicle bundle carrying its equipment }
  realizes:          # instance realizes rulebook obligations (definition -> instance arc)
    property_values: tblAssetPropertyValue (PropertyID -> PropertyValue) realizes obligated properties -> [[rulebook]]
    documents_held:  tblVehicleDocument -> [[documents]]
  audit_trail:
    tblAssetAuditLog: { AssetInstanceID, AssignmentID, HandshakeType, custody event log (29,823 rows) -> [[execution]] }
  accessors:
    current_state: fn_AssetCurrentState / fn_AssetStateAt / fn_AssetStateBetween
    problems:      fn_AssetProblems
    registry:      Dash_AssetRegistry_Browse_Hydrated / Dash_AvailableAssets_Hydrated
    fat_row:       Proc_Hub_FatRow_Hydrator     # VEHICLES class

gotcha: |
  OWNERSHIP is not POSSESSION. Ownership is what the client HAS: an instance exists, or an
  inventory pool holds Quantity N. Possession is who holds it RIGHT NOW: an active
  tblAssetAssignment / tblAssetBulkAssignment. Issuing an asset moves possession; it NEVER
  decrements inventory. "How many do we have" is ownership; "who has it now / what's available"
  is ownership minus active possession. Conflating the two produces false "we're out" answers
  when every unit is simply out on the road.

realized_in:   # neighbors a question crosses to
  possession assigned in         -> [[preparation]] / [[execution]]
  condition + custody events      -> [[execution]]
  kind / capability / doc / prop  -> [[rulebook]]

relationships:
  - Instance IS-A kind defined in [[rulebook]] (AssetCategoryID)
  - Instance stored serialized (tblAssetInstance) | bulk (tblAssetInventory) -- FROZEN mode per [[rulebook]]
  - Instance HAS state (Online/Maintenance/Grounded/Offline/Retired) VIA tblAssetStateChange (asset's own history)
  - Instance HELD-BY a scheduled assignment VIA tblAssetAssignment (possession, current) -> [[preparation]]
  - Instance BUNDLES member equipment VIA tblAssetBundleMember
  - Instance REALIZES obligated properties VIA tblAssetPropertyValue -> [[rulebook]]
  - Instance HOLDS documents VIA tblVehicleDocument -> [[documents]]
  - Custody + inspection events -> [[execution]]

fill_reality:   # client 7293, verified 2026-07-14
  serialized_instances: 687   # at client locations
  bulk_inventory_rows: 8
  state_changes: 401
  audit_log_events: 29823
  states: [ONLINE, MAINTENANCE, GROUNDED, OFFLINE, RETIRED]

cite: tblAssetInstance.AssetInstanceID (or tblAssetInventory row) + the tblAssetStateChange / tblAssetAssignment rows behind a state or custody claim
intents: []
---

## Meaning

Assets are the instance layer: the actual items that realize the kinds defined in [[rulebook]].
One concept, two storage modes. A serialized asset is a numbered physical item (tblAssetInstance,
one row each). A bulk asset is a counted pool (tblAssetInventory, a Quantity, not instances).
Which mode applies is frozen by the rulebook.

OWNERSHIP vs POSSESSION. This is the distinction that must never blur. Ownership is what the
client has; possession is who holds it now. Issuing an asset moves possession and never
decrements ownership. So "how many do we have" and "who has it, what's free" are different
questions: the second is ownership minus active possession. Conflate them and the portal will
report a shortage when in truth every unit is just out on the road.

STATE. An asset sits in one of five operational states: Online, Maintenance, Grounded, Offline,
Retired. State changes flow through the asset's own history (tblAssetStateChange) with
class-grained reasons, one of which raises the Under-Review pin. Current truth comes from
fn_AssetCurrentState; what's wrong comes from fn_AssetProblems.

REALIZES THE RULEBOOK. An instance carries the property values the rulebook obligated
(tblAssetPropertyValue) and holds the documents it required (tblVehicleDocument). The definition
lives in [[rulebook]]; the realized value lives here. That is the arc a compliance question rides.

EVENTS LIVE IN EXECUTION. Custody transfers (handover, issuance, return, the 29k-row audit log)
and inspection captures are [[execution]]. This concept is the item and its current state, not
the events that change it.

DISCLOSURE. The client's own fleet, broadly citable: what they own, where it sits, its condition,
and who holds it. Internal is only the routing and handshake plumbing.
