---
concept: eligibility
title: Eligibility (Badges, Requirements, Documents)
kind: rulebook
branch: 1-eligibility          # replaces 03-rulebook; equipment moved to [[assets]]
aka: [eligibility, qualified, qualifications, badge, badges, requirement, requirements,
      role, roles, transport, service, delivery type, vehicle type, licence, license,
      certification, document, credential, expiry, expired, "can they drive",
      "is he qualified", "who can fill", "why can't they"]

scope_note: |
  PEOPLE. May a PERSON be put in a seat. Every requirement in this chain
  resolves to a DRIVER-owned document. Vehicle-owned documents (3 types, 202
  held) are the asset's roadworthiness and belong to [[assets]]; they never
  enter this chain. The two share the word "document" and nothing else. WHERE
  badges attach to the work is [[structure]]; what the driver HOLDS is
  [[documents]] and [[labor]]. This concept is the rule that joins them.

disclosure:
  citable:  [badge Label, the three category words, DocumentType Label, ExpiryDate,
             held / not held, eligible / not eligible, drives, delivers]
  internal: [SlotNo (the requirement number — the FE renumbers 1..N for display),
             slotKey composition, BadgeAliasID, ledger StatusType 712,
             the gate's internal tiering, tblCapabilityGroup.Label (a retired word)]

grounding:
  definition:
    tblCapabilityGroup:    { keys: [CapabilityGroupID], carries: [Code, BadgeLabel], role: the THREE badge categories — BadgeLabel is the word (Transport · Role · Service). The table name and its Label column are retired vocabulary and never cited. }
    tblBadge:              { keys: [BadgeID], carries: [CapabilityGroupID, Label, IconID, Drives, Delivers, HasRoute, Archived], role: every badge, all three categories, one table. Drives/Delivers are REACH and sit only on Role badges. }
    tblBadgeAlias:         { keys: [BadgeAliasID], role: the ONLY client-editable surface on a badge — label and glyph }
    tblBadgeDocument:      { keys: [BadgeID, ClientID, SlotNo], carries: [DocumentTypeID, Archived], role: badge -> REQUIREMENT NUMBER -> document type. The whole rule, one table. ClientID NULL = system law; a client id = that client's addition. }
    tblBadgeAssetCategory: { keys: [BadgeID, AssetCategoryID], role: which vehicle categories a Transport badge PERMITS — one badge, one or more categories } -> [[assets]]
  supply:
    tblDocument:            { keys: [DocumentID], carries: [OwnerType, OwnerID, DocumentTypeID, ExpiryDate, IsCancelled, Archived], role: what a driver HOLDS } -> [[documents]]
    tblDocumentType:        { keys: [DocumentTypeID], carries: [Label, FacingID, ProofID, CapabilityGroupID, Expires, ClientID, Archived], role: the TYPE fixes who it faces, which category it serves, how it is proven, and whether it expires } -> [[documents]]
    tblClientBadgeDocument: { carries: [IsComplianceChecked, NotifyDaysBefore], role: the client's POLICY on a type — checked at handover or not, and the notice lead }
  accessors:
    badge_read:     fn_ResolveBadgeDisplay(@ClientID)         # ONE read, all categories, alias applied; emits the category word
    reach:          fn_BadgeReach()                            # which Role badges drive / deliver
    requirements:   fn_BadgeDocuments(@ClientID,@WeekID)       # badge -> SlotNo -> document type, with scope; THE keystone, every gate reads it
    supply:         fn_LMDPDocumentSupply(@ClientID,@AsOf)     # what each driver holds, valid on the day
    satisfied:      fn_LMDPCapabilitySatisfied(@ClientID,@WeekID,@AsOf)   # which badges each driver's paperwork satisfies (name predates the collapse; body reads fn_BadgeDocuments)
    demand_at_seat: fn_SeatDemandedSlots(@ClientID,@WeekID)    # the requirements a seat composes, via fn_MissionTypeSeats + fn_RoleDemandedSlots
    match:          fn_LMDPSeatEligible(@ClientID,@WeekID,@AsOf)   # driver x seat pairs with no requirement unmet
    the_gate:       fn_ResolveSeatGate(@ClientId,@WeekId,@FirstDayOfWeek,@OverrideAvailability,@OverrideBadges)   # per driver, per seat, per day; the verdict travels as MEMBERSHIP
    screen:         Dash_Badges_Hydrated                       # Rule Book > Qualification: badges | documents, the requirement numbers
  write_doors:
    Op_Badge_Submit:                alias only, all three categories. No door creates a badge.
    Op_DocumentType_Submit:         a document type's identity AND its badge links (@BadgeIDs), create or edit, ledgered — one gesture, one batch. Subject, proof and expiry are BORN-FIXED; a process change is archive + author new.
    Op_ClientDocumentPolicy_Submit: the client's compliance-checked flag and notice lead, in place.
    Op_SeatRequirement_Submit:      the seat's ASSET demands (kit), not documents -> [[structure]]

composition:   # WHICH badges reach the person in a seat. Reads [[structure]], decided here.
  ROLE:      always                 -> the SEAT's Role badge
  TRANSPORT: if the role DRIVES     -> the MISSION TYPE's Transport badge
  SERVICE:   if the role DELIVERS   -> the MISSION TYPE's Service badge

relationships:
  - Badge BELONGS-TO one of three categories VIA CapabilityGroupID (the category is DATA on the row, never a separate table)
  - Badge DEMANDS document types in NUMBERED REQUIREMENTS VIA tblBadgeDocument
  - Documents sharing a number are ALTERNATIVES (any one satisfies); different numbers are ALL required
  - Driver HOLDS documents VIA tblDocument (OwnerType='Driver')
  - Scope lives INSIDE the requirement key (badge:client:number) — a client may only ADD, never relax system law
  - Reach is two bits on the ROLE badge (Drives, Delivers), never a property of the category
  - Transport badge PERMITS vehicle categories VIA tblBadgeAssetCategory -> [[assets]]   # the single touch point
  - Badges are ATTACHED to the work in [[structure]]; a client may only ALIAS them
  - Client requirement rows are VERSIONED-IN [[coordinate-frame]] time (ledger StatusType 712) and take effect from a future week
  - The gate PUBLISHES membership rows, never scores; fit is [[labor]]'s preference scale

fill_reality:   # client 7293, week 360, verified 2026-09-15
  categories:
    1: { code: VEHICLE,   word: Transport, badges: 8 }
    2: { code: AUTHORITY, word: Role,      badges: 7 }
    3: { code: DELIVERY,  word: Service,   badges: 4 }
  reach: { Driver: [drives, delivers], Backup: [drives, delivers], Supervisor: [drives, delivers],
           Trainer: [drives, delivers], Helper: [delivers], Walker: [delivers], Apprentice: [] }
  requirements_system: 35      # over 45 rows; 3 Transport badges (E-bike, Personal Vehicle, Company Vehicle) demand nothing
  requirements_client_7293: 1  # Step Van Maintenance added under Step Van
  role_requirements: "every Role badge demands Delivery Orientation; driving roles add Driver Authorization; Supervisor adds Supervisor Training; Trainer adds Trainer Certification; Apprentice demands only Apprentice Enrollment"
  transport_permits: 8         # tblBadgeAssetCategory rows, one category per badge today
  demand_at_seat: 42           # fn_SeatDemandedSlots rows
  eligible_pairs: 840          # fn_LMDPSeatEligible rows, driver x seat
  drivers_active: 140
  driver_documents_held: 900
  document_types: 22           # 19 driver-facing · 3 vehicle-facing
  dormant:
    - Trainer Certification holders decide whether a Trainer seat is open at all; issue one and the seat opens

cite: the tblBadgeDocument rows behind a requirement + the tblDocument row behind a holding
intents: []
---

## Meaning

**THE QUESTION.** Eligibility answers one thing: may this person be put in this
seat. Not how good a fit they are — that is [[labor]]'s preference scale.
Whether they are permitted at all.

**THREE CATEGORIES OF BADGE, ONE TABLE.** Transport, Role and Service. They
differ by a column value and nothing more. Badges are ours: no door creates one,
and a client may rename one and change its glyph, which is the whole of their
authorship. One alias store is read by every surface, so one rename reaches the
badge list, the mission type's name and the seat label together.

**WHERE THE BADGES STAND.** A mission type carries one Transport badge and one
Service badge, and they are part of its identity. A seat inside it carries one
Role badge. A person sitting in a mission seat is therefore standing under at
most three badges. Which of them actually reach the person is the next rule.

**REACH, AND WHY A WALKER IS NEVER ASKED FOR A LICENSE.** Every Role badge says
whether it drives and whether it delivers. A role that drives reaches the
mission type's Transport badge; a role that delivers reaches its Service badge.
Driver, Backup, Supervisor and Trainer do both. Helper and Walker deliver and do
not drive, so the Service documents compose and the Transport documents never
do. **There is no exception written anywhere, because the demand was never
composed.** An Apprentice reaches neither, which is what makes it a role that
carries no responsibility.

**THE REQUIREMENT IS A NUMBER.** Each badge demands document types in numbered
requirements. Two documents sharing a number are alternatives, and **any one**
satisfies — that is how "CDL A *or* CDL B" is said without writing an
exception. Different numbers are all required. The count of distinct numbers
is the count of things a person has to go and get. The number itself is
internal and never shown; the screen renumbers what it displays.

**THE CLIENT MAY ADD, NEVER RELAX.** Scope lives *inside* the requirement key —
`badge : client : number`. A client row reusing number 1 forms its own
requirement rather than becoming an alternative to the system's number 1,
which would be a relaxation wearing an addition's clothes. Thirty-five system
requirements stand today; a client's additions sit beside them and take effect
from a future week.

**ROLE REQUIREMENTS ARE SYSTEM LAW TOO.** Every Role badge demands Delivery
Orientation. The roles that drive add Driver Authorization. Supervisor adds
Supervisor Training, Trainer adds Trainer Certification, and Apprentice demands
only its enrollment. Supervise, train and accompany are documents, not a second
kind of reach.

**SUPPLY HAS NO VERSIONS.** A document is held or it is not: unarchived,
uncancelled, unexpired at the client's local date. There is nothing to version
and no lane to read. All the temporal machinery in this concept is on the demand
side.

**HOW A DOCUMENT ENDS.** A type either expires or it does not. An expiring
document carries the date printed on it, and whether anyone is warned beforehand
is the client's policy on that type. When it expires the badge it served goes
dark until a new one is recorded. See [[documents]].

**THE ONE TOUCH INTO ASSETS.** A Transport badge permits one or more vehicle
categories through a link table. A badge is an eligibility statement — "may
operate" — and one badge can cover several vehicles, which is why it is a link
and not a column.

**THE VERDICT IS MEMBERSHIP.** The gate publishes rows, not scores. Present
means admitted; absent means not. Nothing on the wire is a verdict except the
row itself. This is the one absolute in the system; everything else about fit
is a preference on the six-value scale.
