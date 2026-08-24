---
concept: eligibility
title: Eligibility (Badges, Capabilities, Documents)
kind: rulebook
branch: 1-eligibility          # replaces 03-rulebook; equipment moved to [[assets]]
aka: [eligibility, qualified, qualifications, badge, badges, capability, capabilities,
      role, roles, transport, endorsement, endorsements, service, delivery type,
      vehicle type, licence, license, certification, document, credential,
      expiry, expired, lapsed, "can they drive", "is he qualified",
      "who can fill", "why can't they"]

scope_note: |
  PEOPLE. May a PERSON be put in a seat. Every capability requirement in the
  database resolves to a DRIVER-owned document — all of them, all three tiers.
  Vehicle-owned documents (3 types, 210 held) are the asset's roadworthiness
  and belong to [[assets]]; they never enter this chain. The two share the word
  "document" and nothing else. WHERE badges attach is [[structure]].

disclosure:
  citable:  [badge Label, capability Label, tier word, DocumentType Label,
             Expires, NotifyDaysBefore, ExpiryDate, IsExpired, validity window,
             slot satisfaction, isEligible, drives, delivers]
  internal: [CapabilityDocumentID, slotKey composition, BadgeAliasID,
             ledger StatusType 712, the gate's internal tiering]

grounding:
  definition:
    tblCapabilityGroup:    { keys: [CapabilityGroupID], carries: [Code, Label, BadgeLabel], role: THE ONLY PLACE A TIER IS NAMED — and it names it TWICE, see THE TIER HAS TWO WORDS }
    tblBadge:              { keys: [BadgeID], carries: [CapabilityGroupID, Label, IconID], role: every badge, all tiers, one table }
    tblBadgeAlias:         { keys: [BadgeAliasID], role: the ONLY client-editable surface in this concept }
    tblCapability:         { keys: [CapabilityID], carries: [CapabilityGroupID, Label, AssetCategoryID] }
    tblBadgeCapability:    { role: badge -> capability. What a badge DEMANDS. No tier logic. }
    tblCapabilityDocument: { keys: [CapabilityDocumentID], carries: [CapabilityID, SlotNo, DocumentTypeID, ClientID, Archived], role: capability -> SLOT -> document type }
  supply:
    tblDocument:           { keys: [DocumentID], carries: [OwnerType, OwnerID, DocumentTypeID, ExpiryDate, IsCancelled], role: what a driver HOLDS }
    tblDocumentType:       { keys: [DocumentTypeID], carries: [Label, OwnerType, Expires, NotifyDaysBefore, ValidityMinWeeks, ValidityMaxWeeks], role: the TYPE fixes the owner kind AND how it ends } -> [[documents]]
    tblClientDocumentType: { carries: [NotifyDaysBefore, ValidityWeeks], role: the client's POLICY on a shared type }
  accessors:
    badge_read:     fn_ResolveBadgeDisplay(@ClientID)   # ONE read, all tiers; emits groupLabel = the BADGE word
    reach:          fn_BadgeReach()                     # does this badge hold Drive(48) / Deliver(47)
    slots:          fn_CapabilityDocuments(@ClientID,@WeekID)
    supply:         fn_LMDPDocumentSupply(@ClientID,@AsOf)
    satisfied:      fn_LMDPCapabilitySatisfied
    demand_at_seat: fn_SeatDemandedSlots(@ClientID,@WeekID)
    match:          fn_LMDPSeatEligible(@ClientID,@WeekID,@AsOf)
    the_gate:       fn_ResolveSeatGate                  # the verdict travels as MEMBERSHIP
  write_doors:
    Op_Badge_Submit:                alias only, all three tiers. No door creates a badge.
    Op_DocumentType_Submit:         subject, proof, expires AND the validity range are BORN-FIXED.
    Op_ClientDocumentPolicy_Submit: the client's notice lead and lapse window.

composition:   # WHICH attachment reaches a seat. Reads [[structure]], decided here.
  ROLE:      always              -> the SEAT's role badge
  TRANSPORT: if the role DRIVES  -> the MISSION TYPE's transport badge
  SERVICE:   if the role DELIVERS-> the MISSION TYPE's service badge

relationships:
  - Badge BELONGS-TO a tier VIA CapabilityGroupID (the tier is DATA on the row, never a separate table)
  - Badge DEMANDS capabilities VIA tblBadgeCapability
  - Capability IS-PROVEN-BY a SLOT of document types VIA tblCapabilityDocument
  - A SLOT is met by ANY ONE of its documents; a seat is fillable when NO slot is missing
  - Driver HOLDS documents VIA tblDocument (OwnerType='Driver')
  - Scope lives INSIDE the slot key (capability:client:slot) — a client may only ADD
  - Reach is a property of WHAT THE BADGE HOLDS, never of its tier
  - A ROLE requirement is CLIENT-AUTHORED. Unauthored means no demand at all.
  - A windowed document LAPSES; it is never renewed, and carries no notice
  - Badges are ATTACHED in [[structure]]; a client may only ALIAS them
  - TRANSPORT capability POINTS AT an asset category VIA tblCapability.AssetCategoryID -> [[assets]]   # the single touch point
  - Client requirement rows are VERSIONED-IN [[coordinate-frame]] time (ledger StatusType 712)

fill_reality:   # client 7293, week 357, verified 2026-08-23
  tiers:
    1: { code: VEHICLE,   capability_word: Endorsement, badge_word: Transport, badges: 8, capabilities: 8 }
    2: { code: AUTHORITY, capability_word: Authority,   badge_word: Role,      badges: 7, capabilities: 5 }
    3: { code: DELIVERY,  capability_word: Delivery,    badge_word: Service,   badges: 4, capabilities: 5 }
  roles: { Driver: [drives, delivers], Helper: [delivers], Walker: [delivers],
           Backup: [drives, delivers], Supervisor: [supervises, drives, delivers],
           Trainer: [trains, drives, delivers], Apprentice: [] }
  slots: 36            # 27 SYSTEM (unsuppressible), 9 CLIENT (additive only)
  demand_at_seat: 31   # TRANSPORT 21 rows, SERVICE 10, ROLE none — unauthored
  eligible: 227 of 702
  gate_rows: 6201
  drivers: 117
  driver_documents_held: 638
  document_types: 23   # 1 windowed
  dormant:
    - ROLE requirements are unauthored on every client, so no role gate demands anything
    - Trainer Certification has 0 holders, so a Trainer seat arrives closed until one is issued

cite: the tblCapabilityDocument rows behind a requirement + the tblDocument row behind a holding
intents: []
---

## Meaning

**THE QUESTION.** Eligibility answers one thing: may this person be put in this
seat. Not how good a fit they are — that is [[preferences]]. Whether they are
permitted at all.

**TWO FACES, ONE MEETING POINT.** Demand is what a seat requires; supply is what
a driver holds; they meet at the **slot**, and the slot — not the document — is
the unit of match. A slot lists every document that would satisfy it and **any
one** of them satisfies it, which is how "CDL A *or* CDL B" is said without
writing an exception. A seat is fillable when no slot is missing.

**THE TIER IS DATA.** There is one badge table and one capability table, both
keyed by `CapabilityGroupID`. The three tiers differ by a column value and
nothing more. Any code that special-cases a tier is suspect on sight — that is
what the 2026-08-22 collapse removed, when three display readers and one
tier-locked write door became one of each.

**★ THE TIER HAS TWO WORDS, AND THEY ARE NOT INTERCHANGEABLE.** Badges and
capabilities share the group, so the group has to name both — and it needs a
different word for each. Group 1 is **Endorsement** as a capability and
**Transport** as a badge. `Label` is the capability word; `BadgeLabel` is the
badge word. A screen showing badges takes the badge word, and a lens that picks
which badges to show is a badge word too. Handing one to a reader asking the
other question is precisely the mistake a single column used to force.

**REACH, AND WHY A WALKER IS NEVER ASKED FOR A CDL.** Two capabilities — Drive
and Deliver — decide whether a seat's demand reaches *outward* to the badges the
mission type carries. A role holding Drive reaches the transport badge; a role
holding Deliver reaches the service badge. A Walker holds Deliver and not Drive,
so the service documents compose and the vehicle documents never do. **There is
no exception written anywhere, because the demand was never composed.** An
Apprentice holds neither and reaches nothing, which is what makes it a role that
carries no responsibility.

**THE CLIENT MAY ADD, NEVER RELAX.** Scope lives *inside* the slot key —
`capability : client : slot`. A client row reusing slot 1 forms its own slot
rather than becoming an alternative to the system's slot 1, which would be a
relaxation wearing an addition's clothes. Twenty-seven system slots are
unsuppressible; nine client slots sit beside them.

**AND ROLE REQUIREMENTS ARE THE CLIENT'S TO AUTHOR.** Transport and Service
requirements are system law. Role requirements are not: every client is expected
to author its own. This is not an inversion of the rule and not an asymmetry —
**there is simply no demand authored, and demand is what creates the need for
supply.** A gate nobody has built is not an open gate; it is not a gate. One
rule, one justified exception.

**SUPPLY HAS NO VERSIONS.** A document is held or it is not: unarchived,
uncancelled, unexpired at the client's local date. There is nothing to version
and no lane to read. All the temporal machinery in this concept is on the demand
side.

**HOW A DOCUMENT ENDS — AND THE DISTINCTION IS RENEWABILITY, NOT DURATION.** Most
credentials carry a printed date and are **renewed**, so a notice prompts the
action. A **windowed** document lapses on its own after a length the client picks
inside the range the system permits, and carries **no notice at all** — because a
notice prompts a renewal that does not exist, and when it lapses the badge simply
goes dark. The window is read exactly once, at write time; everything downstream
reads the materialised expiry date, which is why a lapsing document needs no
other machinery anywhere.

**BADGES ARE OURS.** There is no client-owned badge and no door that creates one.
A client may alias the label and the glyph — one store, read by every surface, so
one rename reaches the badge list, the mission type's name and the seat label
together — and that is the entire extent of their authorship.

**THE VERDICT IS MEMBERSHIP.** The gate publishes rows, not scores. Present means
admitted; absent means not. Nothing on the wire is a verdict except the row
itself.
