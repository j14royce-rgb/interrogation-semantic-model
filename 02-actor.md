---
concept: actor
title: The Actor & Access Scope
kind: foundational
branch: 0-foundation
aka: [manager, user, actor, asker, driver, DA, supervisor, login, permission, access, who can see]

disclosure:
  citable:  [FirstName, LastName, Email (role-appropriate), RoleName]
  internal: [PasswordHash, OAuthProvider, OAuthSubject, OTP, raw permission rows, CompanyId]

grounding:
  tables:
    Users:             { keys: [UserId], role: login principal (credential only, not an actor) }
    tblUserMapping:    { keys: [MappingId], carries: [UserId, ID, Type, SupervisorAccess], role: resolves a login to a live actor entity }
    tblManagers:       { keys: [ManagerId], carries: [FirstName, LastName, Email, ClientId, RoleId], role: manager actor (the interrogator) }
    tblManagerDetails: { keys: [ManagerId], role: manager contact / detail }
    tblLMDP:           { keys: [LMDPID], role: driver actor (full concept in the Labor branch); the other LIVE actor }
    Roles:             { keys: [RoleId], carries: [RoleName, IsCrossCompany], role: role definition }
    tblRolePermissions:{ role: role -> permission grants }
    tblPermissions:    { keys: [PermissionId], carries: [Category, Permission], role: permission catalog }
    tblUserPermissions:{ role: per-user permission overrides }
    tblPermissionType: { role: permission gated by ClientType }
  mapping_types: { M: manager (-> tblManagers), L: driver (-> tblLMDP), C: client / owner-level }
  accessors:
    actor_scope: tblManagers.ClientId    # the one client this actor may enter
    permissions: RoleId -> tblRolePermissions -> tblPermissions   # + tblUserPermissions overrides

roles:
  catalog: 30 permissions across 6 categories [Administration, Analysis, Create, Engine, Operation, Team]
  live:
    Admin (RoleId 2):            173 actors, 29/30 perms  # de facto standard operator role
    Field Supervisor (RoleId 5): 3 actors, 2 perms        # the one genuinely restricted live role
    Super Admin (RoleId 1):      1 actor, 29 perms, IsCrossCompany=1
  dormant:
    Manager (RoleId 3): 24 perms defined, 0 live actors   # defined-but-unused, like the M:M scaffolding
    LMDP (RoleId 8):    the driver role (labor branch); not a portal asker
  reality: ~97% of live actors are Admin. Live distinction today is effectively Admin vs Field
    Supervisor. No thresholds needed here; the permission catalog is already granular and the
    two-tier disclosure consumes it. Sensitivity thresholds belong to governance, not the actor.
  portal_gate: the Analysis category (5 perms) is the natural gate for interrogation access

relationships:
  - Users RESOLVES-TO Manager|Driver VIA tblUserMapping.Type (M | L)
  - Manager SCOPED-TO exactly one client VIA tblManagers.ClientId
  - Manager GRANTED visibility VIA RoleId + tblUserPermissions
  - Actor GATES entry to [[coordinate-frame]]

scope_note: |
  Manager -> client is 1:1 today. Verified 2026-07-14: tblManagerClient holds 152
  active grants, 0 managers with more than one client, and every grant equals the
  manager's home tblManagers.ClientId. tblManagerClient (M:M), Roles.IsCrossCompany,
  and CompanyId are LATENT multi-tenant scaffolding, not live. Model scope as one
  manager, one client. Do NOT read tblManagerClient as a many-to-many source.

disclosure_behavior: |
  Two-tier. A fact surfaces only where the concept-level `citable` set INTERSECTS
  the actor's permission set (role grants + user overrides, gated by ClientType).
  On a denial the default is withhold-silently: answer as if the fact does not
  exist. A narrow, configurable set may be promoted to acknowledge-but-withhold.

cite: the ManagerId / UserId behind the session + the permission rows authorizing any shown-or-withheld decision
intents: []
---

## Meaning

The actor is who is asking, and which frame they may enter. It is the subject of
every interrogation and the portal's first gate, before the ClientID scope from
[[coordinate-frame]] applies.

WHO. Authentication and identity are separate. `Users` is only a credential; it is
neither manager nor driver. `tblUserMapping` resolves that login to a live actor by
Type: M to a manager (tblManagers), L to a driver (tblLMDP). Both actors are real
and shipping. The driver-facing app is already built on the LMDP side; the portal
v1 asker is the manager.

SCOPE. One manager, one client. Today the relationship is strictly 1:1 (verified).
The many-to-many scaffolding (tblManagerClient, Roles.IsCrossCompany, CompanyId)
exists but is exercised 1:1 and must not be modeled as many-to-many. The manager's
tblManagers.ClientId is the outermost boundary; only inside it does the coordinate
frame apply.

VISIBILITY. Role-based permissions with per-user overrides decide what the actor
may see. This is the dynamic half of disclosure: the concept's static class says
whether a fact is EVER citable; the actor's permission set says whether it is
citable to THIS asker. A fact surfaces only at the intersection of the two, and a
denial withholds silently by default.

ROLES, AS LIVED. The catalog is granular (30 permissions, 6 categories), but the
operation runs almost entirely on one role: ~97% of actors are Admin with near-full
permissions. The only genuinely restricted live role is Field Supervisor (2 perms);
Super Admin (1 actor) is the cross-company owner. The "Manager" role is defined but
unused, dormant like the M:M scaffolding above. So distinctions today are real but
shallow (Admin vs Field Supervisor), and they need no new machinery: the permission
set already carries them. Sensitivity thresholds are a governance concern, not the
actor's. For the portal, the natural entry gate is the Analysis permission category.
