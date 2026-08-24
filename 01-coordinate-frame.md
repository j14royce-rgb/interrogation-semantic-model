---
concept: coordinate-frame
title: Client & Time (the Coordinate Frame)
kind: foundational
branch: 0-foundation
aka: [client, operator, DSP, tenant, account, week, current week, this week, now, today]

disclosure:
  citable:  [ClientName, Address, City, State, WeekDisplay, WeekStarting, WeekEnding, Year]
  internal: [Type, CompanyId, UTCConversion, ObservesDST, FirstDayOfWeek]

grounding:
  tables:
    tblClient:             { keys: [ClientId], role: auth / company hinge (thin) }
    tblClientDetails:      { keys: [ClientId], carries: [ClientName, Address, WeekId], role: identity + current-week anchor }
    tblOnboardingSettings: { keys: [ClientID], carries: [UTCConversion, ObservesDST], role: timezone / DST binding }
    tblWeek:               { keys: [WeekId], scope: GLOBAL (no ClientId), role: shared logical calendar }
  accessors:
    now:          fn_GetClientLocalTime(@ClientID)
    current_week: fn_GetClientCurrentWeekId(@ClientID)
    date_to_week: fn_ResolveWeekIdFromDate(@ClientId, @TargetDate)   # param order verified live 2026-08-22
    to_utc:       fn_ConvertClientToUTC     # write edge
    from_utc:     fn_ConvertUTCToClient     # read edge
  scope_predicate: fn_*ClientPredicate(@ClientID)   # per-entity family

relationships:
  - client HAS timezone-binding VIA tblOnboardingSettings
  - client ANCHORS current-week VIA fn_GetClientCurrentWeekId
  - EVERY downstream fact STAMPED-WITH (ClientId, WeekId)

cite: tblClientDetails.ClientId + the resolved WeekId behind any dated claim
intents: []   # populated in the curated-intent pass, later
---

## Meaning

The coordinate frame answers the two questions that must be settled before any
other fact means anything: whose, and when. Every read in the system is located
in it.

WHOSE. The client is the scope root that every ClientID predicate enforces. It is
not the company (tblCompany sits above it) and not the manager (managers act on
its behalf). Its identity is assembled, not stored in one place: name and address
from tblClientDetails, and the clock (timezone + DST) from tblOnboardingSettings.

WHEN. The calendar (tblWeek) is global and shared, not client-owned. A WeekId is a
logical bucket every client points into. The client's relationship to time is by
pointer and offset, never by ownership: its timezone, its DST rule, and its
FirstDayOfWeek offset. So the same calendar day can map to different WeekIds for
two different clients.

THE CONTRACT. Never GETDATE(), never a raw date. "Now" and "current week" resolve
only through the client-local accessors above. Storage is UTC; conversion happens
at the edges (to_utc on write, from_utc on read). A week is not a date range you
can eyeball; it is a client-relative logical bucket.
