---
concept: rulebook
title: The Equipment Rulebook (Definitional / Kind Layer)
kind: rulebook
branch: 1-rulebook            # supersedes structure; EQUIPMENT-scoped only
aka: [rulebook, equipment catalog, asset categories, equipment types, vehicle types,
      capabilities, required documents, tracked properties, audited properties,
      violation rules, what's required, what qualifies]

scope_note: |
  NARROW. This concept is the definitional policy for EQUIPMENT kinds only. The five
  conceptual asset classes (driver, shift, location, operation, wave) are NOT equipment
  and are NOT modeled here; they share the tblAssetClass registry for audit/hydrator
  plumbing and nothing more. See [[labor]] and [[structure]]. Actual equipment items are
  the instance layer, see [[assets]]. The compliance gate that consumes this rulebook is
  [[governance]]. This entry defines kinds; it does not describe instances, structure, or
  the gate.

disclosure:
  citable:  [CategoryName, capability tier, required DocumentType Label, RequiresOnPerson,
             Expires, NotifyDaysBefore, tracked PropertyName + Unit, IsAudited, AuditFrequency,
             TargetQuantity, ViolationCode + DisplayLabel + severity]
  internal: [EquipmentClassID routing, StorageMode, BridgeTable, IconID, the registry plumbing]

frozen_vs_evolving:
  frozen:   The macro decisions, set at onboarding and permanent. A category's existence,
            its class/type binding (Box Truck is a Vehicles-class category), AND its
            serialized-vs-bulk nature, which decides the instance home: serialized -> a
            SerialNumber row in tblAssetInstance, bulk -> a Quantity row in tblAssetInventory.
  evolving: The rulebook config layered on top. Required documents, obligated/audited
            properties, and the client's adoption (on/off) of a category by week.

grounding:
  taxonomy:
    tblAssetCategory:       { keys: [AssetCategoryID], carries: [CategoryName, EquipmentClassID, CertInheritOrder, DefaultIsBulk], scope: GLOBAL; FROZEN spine }
    tblClientAssetCategory: { keys: [ClientAssetCategoryID], carries: [ClientID, AssetCategoryID, StartWeekID, Archived, IsBulk, IssuanceModeID], role: client ADOPTS a category, week-anchored + archivable; EVOLVING }
  equipment_classes: [VEHICLES, TOOLS, DEVICES, POUCHES, ANCILLARY, UNIFORMS, TELEMATICS, TRAILERS]   # the only classes in scope
  capability:
    tblVehicleCapability:  { AssetCategoryID -> Capability, role: capability tier a category confers (7 rows) }
  documents_required:
    tblVehicleCategoryDocumentRequirement: { AssetCategoryID -> DocumentTypeID (23) }
    tblCapabilityDocumentRequirement:      { Capability -> DocumentTypeID (8) }
    tblDocumentType:       { keys: [DocumentTypeID], carries: [Label, OwnerType (driver|vehicle), Expires, NotifyDaysBefore, RequiresOnPerson, LicenseRank, ClientID] }
    tblClientDocumentType: { role: client config/override of a doc type (10 for 7293) }
  properties_tracked:
    tblAssetProperty:           { keys: [PropertyID], carries: [PropertyName, ValueDataType, PropertyGroupID, PropertyTypeID, Unit] }
    tblAssetCategoryPropertyMap:{ AssetCategoryID -> PropertyID, role: category OBLIGATES a property (412 global) }
    tblClientAssetProperty:     { keys: [ClientAssetPropertyID], carries: [CategoryPropertyID, IsAudited, AuditFrequency, AuditActorID, TargetQuantity], role: client audit config (123 for 7293) }
    tblPropertyFrequency:       { FrequencyLabel -> DaysInterval, role: cadence lookup }
    tblPropertyWorkflow:        { CategoryPropertyID x FrequencyID x InputMethodID, status: DORMANT (0 rows) }
  violations:
    tblAssetViolationRule: { keys: [RuleID], carries: [ViolationCode, DisplayLabel, ClientAssetCategoryID, AudienceRole, DefaultSeverityLane, IsActive, EffectiveFrom, EffectiveTo], status: thin (4 rows) }
  accessors:
    rulebook_read: Dash_AssetRulebook_Hydrated
    capability:    fn_ResolveLMDPCapability / fn_LMDPHasVehicleCert
    doc_status:    fn_VehicleDocumentStatus / fn_LMDPShiftDocsStatus

realized_in:   # the definition -> instance arc; compliance/readiness questions live HERE, not inside either end
  required_document -> held_document:
    driver:  tblLMDPDocument (LMDPID, DocumentTypeID, ExpiryDate)            -> [[labor]] / [[documents]]
    vehicle: tblVehicleDocument (AssetInstanceID, DocumentTypeID, ExpiryDate) -> [[assets]] / [[documents]]
  category -> asset_item:
    serialized: tblAssetInstance (AssetCategoryID, SerialNumber)             -> [[assets]]
    bulk:       tblAssetInventory (ClientAssetCategoryID, Quantity)          -> [[assets]]
  obligated_property -> captured_value:
    tblAssetPropertyValue (PropertyID, PropertyValue, via AuditLogID)        -> [[assets]] / [[execution]]
  inspection_obligation -> inspection_process (use-case: handover):
    trigger tblAssetHandover -> tblInspection -> tblVehicleConditionSnapshot -> tblAssetConditionObjection  -> [[execution]]

fill_reality:   # client 7293, verified 2026-07-14
  adopted_categories: 37 active across 7 equipment classes (no POUCHES); "No Vehicle" (2010) archived sentinel
  category_doc_requirements: 23
  configured_doc_types: 10
  property_audit_configs: 123
  dormant: [tblPropertyWorkflow (0 rows; cadence rides on tblClientAssetProperty instead)]
  thin:    [tblAssetViolationRule (4 rules)]

relationships:
  - Equipment category BELONGS-TO an equipment class VIA EquipmentClassID            # FROZEN
  - Client ADOPTS category VIA tblClientAssetCategory (week-anchored, archivable)    # EVOLVING
  - Category CONFERS capability VIA tblVehicleCapability
  - Category|Capability REQUIRES document VIA tblVehicleCategoryDocumentRequirement | tblCapabilityDocumentRequirement
  - Category OBLIGATES property VIA tblAssetCategoryPropertyMap; client CONFIGURES audit VIA tblClientAssetProperty
  - Required document IS-REALIZED-BY held document (tblLMDPDocument | tblVehicleDocument) VIA DocumentTypeID -> [[documents]]
  - Category IS-INSTANTIATED-BY tblAssetInstance (serialized) | tblAssetInventory (bulk)  -> [[assets]]
  - Obligated property IS-CAPTURED-AS tblAssetPropertyValue -> [[assets]]
  - Inspection obligation IS-EXERCISED-BY the inspection process at handover (tblAssetHandover -> tblInspection -> tblVehicleConditionSnapshot) -> [[execution]]
  - Rulebook is READ-BY the compliance gate -> [[governance]]
  - Rulebook DEFINES the kinds instantiated in [[assets]]
  - Equipment classes are registered in tblAssetClass (shared registry; non-equipment members -> [[labor]], [[structure]])
  - Rulebook config VERSIONED-IN [[coordinate-frame]] time (StartWeekID)

cite: the tblClientAssetCategory / *DocumentRequirement / tblClientAssetProperty rows that define a given obligation
intents: []
---

## Meaning

The equipment rulebook is the client's definitional policy for equipment: for each
kind of equipment, what capability it confers, what documents it requires, and what
properties must be tracked and audited. It defines kinds. It does not describe the
actual items (those are [[assets]]) or the gate that reads it (that is [[governance]]).

SCOPE. Equipment only. The database registers equipment alongside the five conceptual
classes (driver, shift, location, operation, wave) in one table, tblAssetClass, but that
is shared audit and hydrator plumbing, not a shared concept. Those five are modeled in
[[labor]] and [[structure]]. This entry does not reach into them.

FROZEN vs EVOLVING. The macro decision is frozen: a category exists, and its class/type
binding (Box Truck is a Vehicles-class category) is permanent. What evolves is the config
on top, the required documents, the audited properties, and whether the client runs the
category this week. So the rulebook is a frozen spine carrying an evolving policy.

WHAT IT DEMANDS. Three demands hang off an equipment category: a capability tier
(tblVehicleCapability), required documents (per category and per capability, with owner
type, expiry, and notify-window), and obligated properties (tblAssetCategoryPropertyMap),
which the client configures for audit (IsAudited, AuditFrequency, TargetQuantity). These
are the inputs the compliance gate reads; the gate itself lives in [[governance]].

DEFINITION vs INSTANCE. The rulebook only ever states what is REQUIRED. Every requirement
has a realized counterpart living elsewhere, and the highest-value questions ride that arc:
a required document (here) versus the one actually held (tblLMDPDocument for a driver,
tblVehicleDocument for a vehicle, matched on DocumentTypeID, in [[documents]]); an equipment
category (here) versus the actual item (serialized in tblAssetInstance, bulk in
tblAssetInventory, in [[assets]]); an obligated property (here) versus its captured value
(tblAssetPropertyValue). "Is this vehicle missing a required document" starts here and
finishes in the instance; the entry names both ends so the AI never guesses where the other
half lives.

INSPECTION AT HANDOVER. The rulebook sets the inspection/audit OBLIGATION (which properties,
how often, which violation rules). The inspection PROCESS that satisfies it, and its primary
use-case the asset HANDOVER, are execution: an asset changing hands (tblAssetHandover)
triggers an inspection (tblInspection) that writes a condition snapshot
(tblVehicleConditionSnapshot) and may raise objections. Those live in [[execution]] and
[[assets]]. The rulebook is the "what must be checked," never the checking.

LIVE vs DORMANT. For 7293 the rulebook is richly populated: 37 adopted categories across
7 equipment classes, 23 category->document requirements, 10 configured doc types, 123
property audit configs. Two soft spots: tblPropertyWorkflow is empty (audit cadence rides
on tblClientAssetProperty instead), and tblAssetViolationRule is thin (4 rules).

DISCLOSURE. The rulebook is the client's own policy, so what they require is highly
citable: the categories they run, the documents they demand, the properties they audit.
Internal is the registry routing (class binding, storage modes): plumbing that grounds
the AI but never surfaces.
