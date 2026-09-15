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
             valid/expired, cancelled, owner name, days remaining, notice lead]
  internal: [OwnerType / OwnerID routing, FileURL, LegacyTable, LegacyID]
  gated:    [DocumentNumber is a licence number — PII, permission-gated, never open-citable]

grounding:
  held:
    tblDocument: { keys: [DocumentID],
                   carries: [OwnerType, OwnerID, DocumentTypeID, DocumentNumber,
                             IssuingJurisdiction, IssueDate, ExpiryDate, FileURL,
                             IsCancelled, CancelledDate, StartWeekID, Archived, ArchivedDate],
                   role: ONE row per held document, polymorphic owner; unified 2026-07 from
                         two legacy stores, provenance in LegacyTable/LegacyID,
                   NOT_LEDGERED: "a held document has NO lane — see THE STORE IS NOT LEDGERED.
                                  StartWeekID is an anchor on the row, not a version lane." }
  catalog:
    tblDocumentType:        { keys: [DocumentTypeID],
                              carries: [Label, FacingID, ProofID, CapabilityGroupID, Expires, ClientID, Archived],
                              role: THREE AXES and one switch on one row — see THE TYPE CARRIES THREE AXES.
                                    Born-fixed; a process change is archive + author new. }
    tblClientBadgeDocument: { carries: [IsComplianceChecked, NotifyDaysBefore],
                              role: the client's POLICY on a driver-facing type — checked at handover or not, and how many days' notice before expiry }
    tblClientVehicleDocument: { carries: [DocReqID, AuditFrequency, NotifyDaysBefore],
                              role: the client's POLICY on a vehicle requirement } -> [[assets]]
  the_edge:
    tblBadgeDocument: { keys: [BadgeID, ClientID, SlotNo], carries: [DocumentTypeID, Archived],
                        role: THE ONE EDGE — badge x requirement number x document type.
                              Read demand-side by [[eligibility]], supply-side by [[labor]].
                              THIS side IS ledgered (StatusType 712), unlike the held store. }
  requirement:   # what is DEMANDED of a vehicle — defined elsewhere, never here
    tblDocumentRequirement: { role: the global baseline for an asset category } -> [[assets]]
  accessors:
    driver_status:  fn_LMDPDocumentStatus
    vehicle_status: fn_VehicleDocumentStatus
    client_held:    fn_ClientHeldDocuments
    driver_supply:  fn_LMDPDocumentSupply(@ClientID,@AsOf)    -> [[eligibility]]
    write_door:     Proc_AddDocument          # the ONLY writer; the expiry date is whatever the upload supplies
    catalog_doors:  Op_DocumentType_Submit / Op_ClientDocumentPolicy_Submit
    scanner:        Proc_ScanDocumentExpiry   # gates on Expires = 1 AND NotifyDaysBefore IS NOT NULL

type_axes:      # three independent questions about one document type, plus one switch
  FacingID:          { 1 operator · 2 vehicle, role: WHO the paperwork is ABOUT — and therefore where the record lives }
  CapabilityGroupID: { 1 Transport · 2 Role · 3 Service · NULL for vehicle types, role: WHICH BADGE CATEGORY it serves. The column keeps a retired name; the categories are the three badge categories. }
  ProofID:           { 1 attested by us · 2 issued by an authority, role: HOW it is proven }
  Expires:           { 0 never · 1 on a date, role: the switch — see HOW A DOCUMENT ENDS }

how_it_ends:
  never:     { Expires: 0 }
  on_a_date: { Expires: 1, role: the date printed on the credential; the upload ASKS for it. Notice lead is the CLIENT's policy (NotifyDaysBefore); NULL means no notice. }

relationships:
  - Document IS-OF a type VIA DocumentTypeID
  - Type FIXES its owner kind VIA FacingID (operator | vehicle) — no type is both
  - Document HELD-BY a driver | a vehicle VIA (OwnerType, OwnerID) -> [[labor]] | [[assets]]
  - Document SATISFIES a badge requirement VIA tblBadgeDocument, matched on DocumentTypeID -> [[eligibility]]
  - Type DECLARES whether it expires; the client DECLARES the notice lead
  - Type SERVES a badge category VIA CapabilityGroupID — never via FacingID
  - The held store is NOT ledgered; the REQUIREMENT edge is (StatusType 712)

fill_reality:   # verified live 2026-09-15
  documents_held: 2953          # 2,751 driver · 202 vehicle
  document_types: 22            # 19 driver · 3 vehicle — zero overlap
  by_facing:   { 1 operator: 19, 2 vehicle: 3 }
  by_category: { Transport: 9, Role: 5, Service: 5, vehicle (NULL): 3 }
  by_proof:    { attested: 13, issued: 9 }
  expiring_types: 10
  client_authored_types: 1
  requirement_rows: 45          # system, over 35 requirements; 1 client row on 7293
  vehicle_global_requirements: 13
  vocabulary_collision: |
    "Role" means TWO things here: a manager permission role (Roles — grounds
    [[actor]]) and a ROLE BADGE (badge category 2 — grounds [[structure]] and
    [[eligibility]]). Ours to disambiguate internally; never a question we put
    to a manager.

cite: the tblDocument row (+ its tblDocumentType) behind any validity, expiry or qualification claim
intents: []
---

## Meaning

**ONE TABLE, ONE LIFECYCLE.** A document is proof someone or something actually
holds something: issued, expiring, cancellable, archivable. `tblDocument` is
polymorphic — an `OwnerType` and an `OwnerID` say who holds it.

**THE TYPE FIXES THE OWNER, SO NOBODY EVER HAS TO ASK.** A document type faces
an operator or a vehicle, permanently. A driver can never hold a Registration
and a van can never hold a CDL. That is why documents are one concept and not
two: there is no question a manager can ask where *"driver or vehicle"* is the
missing information — naming the document names the owner.

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

**THE TYPE CARRIES THREE AXES, AND THEY ARE INDEPENDENT.** `FacingID` says who
the paperwork is *about*, and so where the record lives. `CapabilityGroupID`
says which *badge category* it serves — Transport, Role or Service — and is
NULL for a vehicle's own paperwork, which confers nothing. `ProofID` says how it
is proven. They are three questions, not three names for one. A fourth field,
`Expires`, is a switch, not an axis.

**ONE EDGE, TWO READINGS.** There is a single table joining badges to document
types, and it is read in both directions. Demand-side, *this badge requires
these documents* — grouped by requirement number, where any one document
sharing a number satisfies it. Supply-side, *holding this document satisfies
these badges*. It is not a conferral arrow and not a requirement arrow; it is
one join, and which way you read it depends on whether you started from the
work or from the person.

**HOW A DOCUMENT ENDS IS A PROPERTY OF ITS TYPE, AND THERE ARE TWO ANSWERS.**
It never expires, or it expires on **the date printed on it**, which the upload
asks for. Whether anyone is warned beforehand is the **client's** policy on that
type, a number of days or nothing. Nothing blocks a re-upload, ever, so there is
no third kind: an expired document is simply expired, and the badge it served
goes dark until a new one is recorded.

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
