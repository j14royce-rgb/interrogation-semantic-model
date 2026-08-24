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
    condition:                 # NOT a separate store since 2026-08: condition IS a property value
      tblAssetPropertyValue (Condition group; produced by RUNNING something, per StorageMode)  -> [[rulebook]]
      tblAssetCategoryPosition: { AssetCategoryID -> PositionCode + PositionLabel, role: a grain BENEATH a property (18 rows) }
      tblConditionEvidence:     { DisputeKeyID -> Uri, CapturedBy, CapturedAt, PurgedAt, role: the photo behind a claim }
  possession:        # CURRENT custody; assigned through [[preparation]]/[[execution]]
    tblAssetAssignment:     { AssignmentScheduledID -> AssetInstanceID, IsActive, StartDate, EndDate, note: serialized custody }
    tblAssetBulkAssignment: { ClientAssetCategoryID, IssuedFromLocationID, IsActive, ClosedState, note: bulk custody }
    tblAssetBundleMember:   { ParentInstanceID -> MemberInstanceID / MemberQuantity, note: a vehicle bundle carrying its equipment }
  realizes:          # instance realizes rulebook obligations (definition -> instance arc)
    property_values: tblAssetPropertyValue (PropertyID -> PropertyValue) realizes obligated properties -> [[rulebook]]
    documents_held:  tblDocument (OwnerType='Vehicle', OwnerID=AssetInstanceID) -> [[documents]]
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
  - Instance HOLDS documents VIA tblDocument (OwnerType='Vehicle') -> [[documents]]
  - A property value may be POSITIONED VIA tblAssetCategoryPosition (per wheel, per face), read with fn_AssetPositionValues
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
(tblAssetPropertyValue) and holds the documents it required (tblDocument, OwnerType 'Vehicle').
The definition lives in [[rulebook]]; the realized value lives here. That is the arc a compliance
question rides.

CONDITION IS A PROPERTY, AND SOME PROPERTIES HAVE POSITIONS. Condition is not stored apart from
everything else; it is an ordinary property value, distinguished only by StorageMode, meaning its
value is produced by RUNNING something rather than by being recorded once. Underneath a property
there can be a further grain: tire pressure is not one number per van, it is one per wheel
(tblAssetCategoryPosition, read through fn_AssetPositionValues). Any question of the form "which
vans have low tire pressure" is unanswerable at the instance grain alone, because the van does
not have a pressure; four positions on it do. Photographic evidence attaches to the claim rather
than to the asset (tblConditionEvidence).

EVENTS LIVE IN EXECUTION. Custody transfers (handover, issuance, return, the 29k-row audit log)
and inspection captures are [[execution]]. This concept is the item and its current state, not
the events that change it.

DISCLOSURE. The client's own fleet, broadly citable: what they own, where it sits, its condition,
and who holds it. Internal is only the routing and handshake plumbing.

---

# ⛔ INBOUND FROM `03-rulebook` — NOT YET INTEGRATED (2026-08-23)

The old `03-rulebook` concept held TWO unrelated subjects in one file: the
**person** eligibility chain (badges → capabilities → documents) and the
**equipment** rulebook below. Jim split them on 2026-08-23. The person half is
now `03-eligibility`; `03-rulebook` has been retired.

**The material below is the equipment half, moved here verbatim so nothing is
stranded. It has NOT been rewritten in the assets voice, its grounding has NOT
been re-verified against live, and its fill numbers are from 2026-08-11.** It is
raw input for the assets pass, not part of this concept yet — do not ground an
answer on it until it has been integrated and re-verified.

Known stale on sight, from the 8/20-23 work:
- `tblLicenseConfers` → now `tblCapabilityDocument` / `fn_CapabilityDocuments`
- `tblBadgeGroup` → now `tblCapabilityGroup`; `tblBadgeDocument` → `tblCapabilityDocument`
- `tblCategoryDocumentRequirement` no longer exists
- `fn_LMDPHasVehicleCert` no longer exists
- badge groups are **Transport / Role / Service**, not OPERATE / CARRY / DELIVER
- the "five conceptual classes" are wrong twice: shift is now MISSION_TYPE and SEAT is a sixth
- TRAILERS is documented as its own class but carries `Disposition = CARRIER`, identical to VEHICLES
- `tblAssetViolationRule` holds 4 rows and **zero modules read it** — dormant, and the
  portal must never answer from it as though something were being enforced

> ## Meaning
> 
> The equipment rulebook is the client's definitional policy for equipment: for each
> kind of equipment, what capability it confers, what documents it requires, and what
> properties must be tracked and audited. It defines kinds. It does not describe the
> actual items (those are [[assets]]) or the gate that reads it (that is [[governance]]).
> 
> SCOPE. Equipment only. The database registers equipment alongside the five conceptual
> classes (driver, shift, location, operation, wave) in one table, tblAssetClass, but that
> is shared audit and hydrator plumbing, not a shared concept. Those five are modeled in
> [[labor]] and [[structure]]. This entry does not reach into them.
> 
> FROZEN vs EVOLVING. The macro decision is frozen: a category exists, and its class/type
> binding (Box Truck is a Vehicles-class category) is permanent. What evolves is the config
> on top, the required documents, the audited properties, and whether the client runs the
> category this week. So the rulebook is a frozen spine carrying an evolving policy.
> 
> WHAT IT DEMANDS. Two demands hang off an equipment category: required documents (a global
> baseline in tblDocumentRequirement plus client-scoped rows in tblCategoryDocumentRequirement,
> each naming a document type that carries its own owner kind, expiry and notify-window), and
> obligated properties (tblAssetCategoryPropertyMap), which the client configures for audit
> (IsAudited, AuditFrequency, TargetQuantity). These are the inputs the compliance gate reads;
> the gate itself lives in [[governance]].
> 
> THE ARROW REVERSED. A category used to confer a ranked capability tier, and the tier then
> demanded documents. That is retired. Now the document grants: a licence confers the vehicle
> CATEGORIES it permits (tblLicenseConfers), and a training confers a service BADGE
> (tblBadgeDocument). Demand and supply are exact mirrors around it. The demand side
> containerizes documents through a shift type, its vehicle and its badges; the supply side
> holds documents that roll up into badges and vehicle categories, which then satisfy the shift
> type. Documents are the gate at the bottom of both. Which of the two mechanisms consumes a
> document is decided by its FacingID, not by who holds it -- see [[documents]].
> 
> DEFINITION vs INSTANCE. The rulebook only ever states what is REQUIRED. Every requirement
> has a realized counterpart living elsewhere, and the highest-value questions ride that arc:
> a required document (here) versus the one actually held (tblDocument, one table for both
> drivers and vehicles, matched on DocumentTypeID, in [[documents]]); an equipment
> category (here) versus the actual item (serialized in tblAssetInstance, bulk in
> tblAssetInventory, in [[assets]]); an obligated property (here) versus its captured value
> (tblAssetPropertyValue). "Is this vehicle missing a required document" starts here and
> finishes in the instance; the entry names both ends so the AI never guesses where the other
> half lives.
> 
> INSPECTION AT HANDOVER. The rulebook sets the inspection/audit OBLIGATION (which properties,
> how often, which violation rules). The inspection PROCESS that satisfies it, and its primary
> use-case the asset HANDOVER, are execution: an asset changing hands (a ledger event through
> Op_Asset_Handover, read via vw_AssetHandoverCurrent) triggers an inspection (tblInspection)
> whose findings are written as ordinary property VALUES on the instance
> (tblAssetPropertyValue, Condition group) and may raise objections. Those live in [[execution]] and
> [[assets]]. The rulebook is the "what must be checked," never the checking.
> 
> LIVE vs DORMANT. For 7293 the rulebook is richly populated: 43 adopted categories, 122
> property audit configs against 405 global category-property obligations, and 15 licence
> conferrals. Three soft spots: tblPropertyWorkflow is empty (audit cadence rides on
> tblClientAssetProperty instead), tblAssetViolationRule is thin (4 rules), and 7293's own
> document-type catalogue carries live test residue ("New", "My New Certificate") that a
> client-facing picker would show.
> 
> DISCLOSURE. The rulebook is the client's own policy, so what they require is highly
> citable: the categories they run, the documents they demand, the properties they audit.
> Internal is the registry routing (class binding, storage modes): plumbing that grounds
> the AI but never surfaces.

> ---
> concept: rulebook
> title: The Equipment Rulebook (Definitional / Kind Layer)
> kind: rulebook
> branch: 1-rulebook            # supersedes structure; EQUIPMENT-scoped only
> aka: [rulebook, equipment catalog, asset categories, equipment types, vehicle types,
>       capabilities, required documents, tracked properties, audited properties,
>       violation rules, what's required, what qualifies]
> 
> scope_note: |
>   NARROW. This concept is the definitional policy for EQUIPMENT kinds only. The five
>   conceptual asset classes (driver, shift, location, operation, wave) are NOT equipment
>   and are NOT modeled here; they share the tblAssetClass registry for audit/hydrator
>   plumbing and nothing more. See [[labor]] and [[structure]]. Actual equipment items are
>   the instance layer, see [[assets]]. The compliance gate that consumes this rulebook is
>   [[governance]]. This entry defines kinds; it does not describe instances, structure, or
>   the gate.
> 
> disclosure:
>   citable:  [CategoryName, capability tier, required DocumentType Label, RequiresOnPerson,
>              Expires, NotifyDaysBefore, tracked PropertyName + Unit, IsAudited, AuditFrequency,
>              TargetQuantity, ViolationCode + DisplayLabel + severity]
>   internal: [EquipmentClassID routing, StorageMode, BridgeTable, IconID, the registry plumbing]
> 
> frozen_vs_evolving:
>   frozen:   The macro decisions, set at onboarding and permanent. A category's existence,
>             its class/type binding (Box Truck is a Vehicles-class category), AND its
>             serialized-vs-bulk nature, which decides the instance home: serialized -> a
>             SerialNumber row in tblAssetInstance, bulk -> a Quantity row in tblAssetInventory.
>   evolving: The rulebook config layered on top. Required documents, obligated/audited
>             properties, and the client's adoption (on/off) of a category by week.
> 
> grounding:
>   taxonomy:
>     tblAssetCategory:       { keys: [AssetCategoryID], carries: [CategoryName, EquipmentClassID, CertInheritOrder, DefaultIsBulk], scope: GLOBAL; FROZEN spine }
>     tblClientAssetCategory: { keys: [ClientAssetCategoryID], carries: [ClientID, AssetCategoryID, StartWeekID, Archived, IsBulk, IssuanceModeID], role: client ADOPTS a category, week-anchored + archivable; EVOLVING }
>   equipment_classes: [VEHICLES, TOOLS, DEVICES, POUCHES, ANCILLARY, UNIFORMS, TELEMATICS, TRAILERS]   # the only classes in scope
>   qualification:   # REVERSED 2026-08: a licence CONFERS categories; the ranked capability tier is retired
>     tblLicenseConfers: { DocumentTypeID -> AssetCategoryID, role: the vehicle categories a licence permits (15 rows) } -> [[documents]]
>     tblBadge / tblBadgeGroup / tblBadgeDocument: { role: a training CONFERS a service badge; groups OPERATE|CARRY|DELIVER, OPERATE archived since conferral replaced it } -> [[labor]]
>   documents_required:
>     tblDocumentRequirement:         { DocumentTypeID -> AssetCategoryID, scope: GLOBAL baseline (13 rows) }
>     tblCategoryDocumentRequirement: { AssetCategoryID -> DocumentTypeID + SlotNo, carries: [ClientID], role: client-scoped requirement (12 rows) }
>     tblDocumentType:       { keys: [DocumentTypeID], carries: [Label, OwnerType (Driver|Vehicle), Expires, NotifyDaysBefore, FacingID, ProofID, ClientID], role: the TYPE fixes the owner kind } -> [[documents]]
>     tblClientDocumentType: { role: client adoption of a global doc type }
>   properties_tracked:
>     tblAssetProperty:           { keys: [PropertyID], carries: [PropertyName, ValueDataType, PropertyGroupID, PropertyTypeID, Unit] }
>     tblAssetCategoryPropertyMap:{ AssetCategoryID -> PropertyID, role: category OBLIGATES a property (412 global) }
>     tblClientAssetProperty:     { keys: [ClientAssetPropertyID], carries: [CategoryPropertyID, IsAudited, AuditFrequency, AuditActorID, TargetQuantity], role: client audit config (123 for 7293) }
>     tblPropertyFrequency:       { FrequencyLabel -> DaysInterval, role: cadence lookup }
>     tblPropertyWorkflow:        { CategoryPropertyID x FrequencyID x InputMethodID, status: DORMANT (0 rows) }
>   violations:
>     tblAssetViolationRule: { keys: [RuleID], carries: [ViolationCode, DisplayLabel, ClientAssetCategoryID, AudienceRole, DefaultSeverityLane, IsActive, EffectiveFrom, EffectiveTo], status: thin (4 rows) }
>   accessors:
>     rulebook_read: Dash_AssetRulebook_Hydrated
>     capability:    fn_LMDPCapabilityStates / fn_LMDPHasVehicleCert
>     doc_status:    fn_VehicleDocumentStatus / fn_LMDPDocumentStatus / fn_LMDPDocumentRollup
> 
> realized_in:   # the definition -> instance arc; compliance/readiness questions live HERE, not inside either end
>   required_document -> held_document:      # UNIFIED 2026-07 into one polymorphic table
>     driver:  tblDocument (OwnerType='Driver',  OwnerID=LMDPID,         DocumentTypeID, ExpiryDate) -> [[documents]] / [[labor]]
>     vehicle: tblDocument (OwnerType='Vehicle', OwnerID=AssetInstanceID, DocumentTypeID, ExpiryDate) -> [[documents]] / [[assets]]
>   category -> asset_item:
>     serialized: tblAssetInstance (AssetCategoryID, SerialNumber)             -> [[assets]]
>     bulk:       tblAssetInventory (ClientAssetCategoryID, Quantity)          -> [[assets]]
>   obligated_property -> captured_value:
>     tblAssetPropertyValue (PropertyID, PropertyValue, via AuditLogID)        -> [[assets]] / [[execution]]
>   inspection_obligation -> inspection_process (use-case: handover):
>     # the handover is a LEDGER EVENT since 2026-07, not a table; condition is a PROPERTY VALUE since 2026-08
>     Op_Asset_Handover / vw_AssetHandoverCurrent -> tblInspection -> tblAssetPropertyValue (Condition group, StorageMode)
>       -> tblAssetConditionObjection  -> [[execution]] / [[assets]]
> 
> fill_reality:   # client 7293 unless noted, verified 2026-08-11
>   adopted_categories: 43 active
>   category_doc_requirements: 3 for 7293 (12 all clients); global baseline tblDocumentRequirement 13
>   client_defined_doc_types: 3   # NOTE: 'New' and 'My New Certificate' are live test residue, 'ZZ_SMOKE_RENAMED' archived
>   property_audit_configs: 122
>   category_property_map: 405 global
>   licence_conferrals: 15
>   dormant: [tblPropertyWorkflow (0 rows; cadence rides on tblClientAssetProperty instead)]
>   thin:    [tblAssetViolationRule (4 rules)]
> 
> relationships:
>   - Equipment category BELONGS-TO an equipment class VIA EquipmentClassID            # FROZEN
>   - Client ADOPTS category VIA tblClientAssetCategory (week-anchored, archivable)    # EVOLVING
>   - Licence CONFERS categories VIA tblLicenseConfers          # REVERSED: the document grants, the category no longer does
>   - Training CONFERS a service badge VIA tblBadgeDocument -> [[labor]]
>   - Category REQUIRES document VIA tblDocumentRequirement (global) | tblCategoryDocumentRequirement (client)
>   - ShiftType DEMANDS a badge VIA tblShiftTypeQualification -> [[structure]]
>   - Category OBLIGATES property VIA tblAssetCategoryPropertyMap; client CONFIGURES audit VIA tblClientAssetProperty
>   - Required document IS-REALIZED-BY held document (tblDocument, owner fixed by the TYPE) VIA DocumentTypeID -> [[documents]]
>   - Category IS-INSTANTIATED-BY tblAssetInstance (serialized) | tblAssetInventory (bulk)  -> [[assets]]
>   - Obligated property IS-CAPTURED-AS tblAssetPropertyValue -> [[assets]]
>   - Inspection obligation IS-EXERCISED-BY the inspection process at handover (Op_Asset_Handover -> tblInspection -> tblAssetPropertyValue) -> [[execution]]
>   - Rulebook is READ-BY the compliance gate -> [[governance]]
>   - Rulebook DEFINES the kinds instantiated in [[assets]]
>   - Equipment classes are registered in tblAssetClass (shared registry; non-equipment members -> [[labor]], [[structure]])
>   - Rulebook config VERSIONED-IN [[coordinate-frame]] time (StartWeekID)
> 
> cite: the tblClientAssetCategory / *DocumentRequirement / tblClientAssetProperty rows that define a given obligation
