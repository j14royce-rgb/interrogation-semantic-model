---
concept: documents
title: Documents (the Held Proof)
kind: entity
branch: 5-documents
aka: [document, paperwork, papers, license, licence, CDL, non-CDL, medical card,
      DOT card, registration, insurance, DOT inspection, certificate, cert,
      training, enrollment, road test, written test, expiry, expires, expired,
      lapsed, renewal, out of date, "what's expiring", "who's missing paperwork",
      "is he legal", "is the van legal"]

scope_note: |
  One table, one lifecycle: the proof a driver or a vehicle actually HOLDS.
  What is REQUIRED of whom is [[eligibility]] (of people) and [[assets]] (of
  equipment). What a held document lets someone DO resolves through
  [[eligibility]]. This entry is the document and whether it is good — not the
  requirement, and not the gate.

disclosure:
  citable:  [DocumentType Label, IssuingJurisdiction, IssueDate, ExpiryDate,
             valid/expired/lapsed, cancelled, owner name, days remaining,
             validity window]
  internal: [OwnerType / OwnerID routing, FileURL, LegacyTable, LegacyID]
  gated:    [DocumentNumber is a licence number — PII, permission-gated, never open-citable]

grounding:
  held:
    tblDocument: { keys: [DocumentID],
                   carries: [OwnerType, OwnerID, DocumentTypeID, DocumentNumber,
                             IssuingJurisdiction, IssueDate, ExpiryDate, FileURL,
                             IsCancelled, CancelledDate, StartWeekID, Archived, ArchivedDate],
                   role: ONE row per held document, polymorphic owner; unified 2026-07 from
                         tblLMDPDocument + tblVehicleDocument, provenance in LegacyTable/LegacyID,
                   NOT_LEDGERED: "a held document has NO lane — see THE STORE IS NOT LEDGERED.
                                  StartWeekID is an anchor on the row, not a version lane." }
  catalog:
    tblDocumentType:       { keys: [DocumentTypeID],
                             carries: [Label, OwnerType, FacingID, ProofID, CapabilityGroupID,
                                       Expires, NotifyDaysBefore, ValidityMinWeeks, ValidityMaxWeeks,
                                       ClientID, Archived],
                             role: FOUR AXES on one row — see THE TYPE CARRIES FOUR AXES }
    tblClientDocumentType: { carries: [IsComplianceChecked, NotifyDaysBefore, ValidityWeeks],
                             role: the client's POLICY on a shared type }
  the_edge:
    tblCapabilityDocument: { role: THE ONE EDGE — capability x slot x document type.
                                   Read demand-side by [[eligibility]], supply-side by [[labor]].
                                   THIS side IS ledgered (StatusType 712), unlike the held store. }
  requirement:   # what is DEMANDED — defined elsewhere, never here
    tblDocumentRequirement: { role: the global baseline for an asset category } -> [[assets]]
  accessors:
    driver_status:  fn_LMDPDocumentStatus
    vehicle_status: fn_VehicleDocumentStatus
    client_held:    fn_ClientHeldDocuments
    driver_supply:  fn_LMDPDocumentSupply    -> [[eligibility]]
    write_door:     Proc_AddDocument          # the ONLY writer; derives a windowed expiry
    catalog_doors:  Op_DocumentType_Submit / Op_ClientDocumentPolicy_Submit
    scanner:        Proc_ScanDocumentExpiry   # gates on Expires = 1 AND Notify IS NOT NULL

type_axes:      # four independent questions about one document type
  OwnerType:         { Driver | Vehicle, role: WHERE the record is stored and who it hangs off }
  FacingID:          { 1 operator · 2 vehicle · 3 role, role: WHO the paperwork is ABOUT }
  CapabilityGroupID: { 1 Transport · 2 Role · 3 Service · NULL, role: WHICH TIER it serves }
  ProofID:           { 1 attested by us · 2 issued by an authority, role: HOW it is proven }

how_it_ends:
  never:       { Expires: 0 }
  on_a_date:   { Expires: 1, window: NULL, role: the date printed on the credential; the upload ASKS for it }
  on_a_window: { Expires: 1, window: set,  role: LAPSES after N weeks; the upload does NOT ask, and it carries NO notice }

relationships:
  - Document IS-OF a type VIA DocumentTypeID
  - Type FIXES its owner kind VIA OwnerType (Driver | Vehicle) — no type is both
  - Document HELD-BY a driver | a vehicle VIA (OwnerType, OwnerID) -> [[labor]] | [[assets]]
  - Document SATISFIES a capability VIA tblCapabilityDocument, matched on DocumentTypeID -> [[eligibility]]
  - Type DECLARES how it ends (never | a printed date | a window)
  - Type SERVES a tier VIA CapabilityGroupID — NOT via FacingID, which spans two
  - The held store is NOT ledgered; the REQUIREMENT edge is (StatusType 712)

fill_reality:   # verified live 2026-08-23
  documents_held: 2508          # 2,298 driver · 210 vehicle
  document_types: 23            # 20 driver · 3 vehicle — zero overlap
  by_facing:  { 1 operator: 10, 2 vehicle: 3, 3 role: 10 }
  by_proof:   { attested: 13, issued: 10 }
  facing_3_spans_two_tiers: { Role: 5, Service: 5 }
  capability_edges: 46          # Transport 31 (10 capabilities) · Service 6 (6) · Role 0
  global_requirements: 13
  windowed_types: 1             # Apprentice Enrollment, 2-6 weeks
  vocabulary_collision: |
    "Role" means THREE things here: a manager permission role (Roles — grounds
    [[actor]]), a ROLE BADGE (tblBadge group 2 — grounds [[structure]] and
    [[eligibility]]), and Facing 3 on a document type. Ours to disambiguate
    internally; never a question we put to a manager.

cite: the tblDocument row (+ its tblDocumentType) behind any validity, expiry or qualification claim
intents: []
---

## Meaning

**ONE TABLE, ONE LIFECYCLE.** A document is proof someone or something actually
holds something: issued, expiring, cancellable, archivable. `tblDocument` is
polymorphic — an `OwnerType` and an `OwnerID` say who holds it.

**THE TYPE FIXES THE OWNER, SO NOBODY EVER HAS TO ASK.** A document type is
permanently a driver type or a vehicle type. A driver can never hold a
Registration and a van can never hold a CDL. That is why documents are one
concept and not two: there is no question a manager can ask where *"driver or
vehicle"* is the missing information — naming the document names the owner.

**★ THE STORE IS NOT LEDGERED.** This is the one place in the model where the
usual pattern does not apply. A held document is **held or it is not** —
unarchived, uncancelled, and unexpired at the client's local date. There is no
lane to read, no week to resolve, no version to pick. `StartWeekID` sits on the
row as an anchor; it is not a version lane and must never be read as one. The
temporal machinery in this area is all on the *requirement* side, where a
client's own requirement rows ARE ledgered (StatusType 712). Ask a held document
what it is today and the answer is a fact, not a resolution.

**HOLDING MEANS VALID.** This is the trap. A row proves a document was
*recorded*, not that it is *good*. Cancelled, archived and expired rows sit in
the same table looking identical to current ones. In this system "has the
document" and "is qualified" mean the same thing, deliberately, because having
implies active and correct and fit for the purpose. **So a count that skips the
validity filter has not made a small error — it has answered a different
question, and it will report a compliant fleet that is not.**

**THE TYPE CARRIES FOUR AXES, AND THEY ARE INDEPENDENT.** `OwnerType` says where
the record lives. `FacingID` says who the paperwork is *about*.
`CapabilityGroupID` says which *tier* it serves. `ProofID` says how it is
proven. They are four questions, not four names for one.

**AND FACING NO LONGER IDENTIFIES A TIER.** It used to: operator paperwork
bought vehicles, role paperwork bought services. Now **Facing 3 spans two
tiers** — five of those types serve Role and five serve Service — so a question
that routes on Facing alone will collect a supervisor's certification and a
white-glove training into the same answer. **The tier is `CapabilityGroupID`'s
to say.** Facing 2 remains what it always was: a vehicle's own standing,
conferring nothing, and it carries a NULL tier to say so.

**ONE EDGE, TWO READINGS.** There is a single table joining capabilities to
document types, and it is read in both directions. Demand-side, *this capability
requires these documents* — the slot, where any one of the listed documents
satisfies it. Supply-side, *holding this document satisfies these capabilities*.
It is not a conferral arrow and not a requirement arrow; it is one join, and
which way you read it depends on whether you started from the work or from the
person.

**HOW A DOCUMENT ENDS IS A PROPERTY OF ITS TYPE, AND THERE ARE THREE ANSWERS.**
It never expires. Or it expires on **the date printed on it**, which the upload
asks for and a notice warns about. Or it **lapses on a window** — a length the
client chooses inside a range we permit — in which case the upload asks for no
date at all and there is **no notice**, because notice exists to prompt a
renewal and a lapsing document is designed to end. When it lapses, the badge
simply goes dark.

**HOW IT IS PROVEN IS ITS OWN AXIS.** `ProofID` separates a document issued by
an outside authority from one we attested ourselves. A CDL can be checked
against a jurisdiction; a Road Test is our own word. Both are held, both count,
and they carry very different weight the moment anyone challenges them.

**THE REQUIREMENT LIVES ELSEWHERE.** What must be held, by whom, and for what
work is [[eligibility]]'s of people and [[assets]]'s of equipment. This concept
is only the held proof and whether it is good.

**DISCLOSURE.** Types, dates, jurisdictions and validity are the client's own
compliance picture and broadly citable. The licence **number** is personal
identification and is gated. Owner routing, file URL and migration provenance
are plumbing.
