# Proposal: Custom Project Roles
Author: Max Graustenzel (@maxgraustenzel-create)
Discussion: #18124

## Abstract

Add support for custom project roles in Harbor, allowing system administrators to create roles with flexible permission combinations beyond the five built-in roles. This extends the existing RBAC system to enable runtime role creation without code changes, addressing long-standing community requests for role customization while maintaining full backward compatibility.

## Background

Harbor currently provides five hardcoded project roles (Project Admin, Maintainer, Developer, Guest, Limited Guest) with fixed permissions defined in `rbac_role.go`. Organizations have diverse security and workflow requirements that cannot be met by these predefined roles alone.

**Current Limitations:**
- Role permissions are hardcoded in Go source code
- Adding new roles requires code changes and Harbor releases
- Organizations cannot adapt roles to specific workflows (e.g., "DevOps Engineer" with artifact management + robot creation but no member management)
- No middle-ground between overly permissive and overly restrictive built-in roles

**Community Demand:**
- Issue #18124 and related issues (#18143, #21306, #12062, #8632, #1486) have 60+ positive reactions
- Consistent feedback requesting role customization capabilities
- Organizations work around limitations by creating multiple projects or granting excessive permissions

**Example Use Cases:**
- **DevOps Engineer:** Push/pull artifacts + manage robot accounts, but cannot manage project members
- **Security Auditor:** Read-only access + trigger vulnerability scans, but cannot modify artifacts
- **Release Manager:** Manage artifacts + set tag immutability, but cannot delete repositories

## Proposal

Extend Harbor's existing RBAC infrastructure to support custom roles by linking the existing `role` table to the `permission_policy` table through the `role_permission` table.

**Core Changes:**

1. **Database:** Move role permissions from hardcoded `rbac_role.go` to the database (`role_permission` table) as the single source of truth
2. **API:** Add role CRUD endpoints for system administrators
3. **UI:** Add role management interface in System Administration section
4. **Security:** Implement privilege escalation prevention and audit logging
5. **Caching:** In-process (per-node) permission cache, on by default with a 1 s TTL; optional Redis tier off by default; all caching disableable (see *Performance Caching*)

**Key Design Decisions:**

- **Reuse existing infrastructure:** `role`, `permission_policy`, `role_permission`, and `project_member` tables already exist
- **Minimal schema changes:** Only extend `role` table with metadata columns (`is_builtin`, `description`, `modified`, `created_by`, `modified_by`, timestamps)
- **Discriminator pattern:** `role_permission.role_type` distinguishes 'project-role' (users/groups) from 'robotaccount' (direct permissions)
- **System admin only:** Only system administrators can create/modify custom roles (project admins assign roles, existing workflow unchanged)
- **Built-in roles are immutable:** The five built-in roles (projectAdmin, maintainer, developer, guest, limitedGuest) can be neither modified nor deleted — they are the stable, secure baseline. `is_builtin = TRUE` is enforced on both backend (create/update/delete reject built-in roles) and frontend (controls disabled). Only *custom* roles are editable.
- **Single source of truth:** Role permissions (built-in and custom) live exclusively in `role_permission` + `permission_policy`; the compile-time `rolePoliciesMap` in `rbac_role.go` is removed. Built-in roles and any future built-in roles are seeded via **database migration** as part of the version-upgrade workflow — no Go code path re-introduces hardcoded permissions.

**Technical Architecture:**

```
┌──────────────────────────────────────────────────────────────┐
│ Before: Hardcoded in rbac_role.go                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│         ┌─────────────────┐                                  │
│         │ users/groups    │                                  │
│         └────────┬────────┘                                  │
│                  ↑                                           │
│                  │                                           │
│         ┌─────────────────┐                                  │
│         │ project_member  │                                  │
│         └────────┬────────┘                                  │
│                  │                                           │
│                  ↓                                           │
│         ┌─────────────────┐                                  │
│         │  role (table)   │                                  │
│         └────────┬────────┘                                  │
│                  │                                           │
│                  ↓                                           │
│            permissions                                       │
│          (hardcoded in                                       │
│           rbac_role.go)                                      │
│                                                              │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ After: Database-driven                                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│         ┌─────────────────┐                                  │
│         │ users/groups    │                                  │
│         └─────────────────┘                                  │
│                  ↑                                           │
│                  │                                           │
│         ┌────────┴────────┐        ┌─────────────────┐       │
│         │ project_member  │───────→│  role (table)   │       │
│         └─────────────────┘        └─────────────────┘       │
│                                             ↑                │
│                                             │                │
│                                    ┌────────┴────────┐       │
│                                    │ role_permission │       │
│                                    │ (role_type=     │       │
│                                    │ 'project-role') │       │
│                                    └────────┬────────┘       │
│                                             │                │
│                                             ↓                │
│                                    ┌─────────────────┐       │
│                                    │ permission_     │       │
│                                    │ policy (table)  │       │
│                                    └─────────────────┘       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**User Workflow:**

1. **System Admin:** Creates custom role "DevOps Engineer" with specific permissions
2. **Project Admin:** Assigns "DevOps Engineer" role to user/group (same as built-in roles)
3. **Authorization:** Permission checks resolve the member's role from the per-node cache (falling back to the DB on a miss); a role change propagates to all nodes within the cache TTL (1 s default)

## Non-Goals

**Explicitly out of scope for this proposal:**

1. **Repository-level permissions:** Permissions apply at project level (all repositories in project). Repository-level permission assignment is tracked separately in Issue #10159
2. **Global role assignments:** Custom roles are assigned per-project (user can have different roles in different projects). Global role assignment is tracked in Issue #8351
3. **Role templates/marketplace:** No predefined custom role templates in initial release
4. **Role inheritance/composition:** Roles are flat, not hierarchical
5. **Instant global propagation:** Role changes converge across all nodes within the cache TTL (1 s by default), not instantaneously; a zero-stale mode is available by disabling the cache (per-request DB evaluation)
6. **Robot account roles:** Robots continue using direct permission assignment (no role concept)

## Rationale

**Why database-driven vs. more built-in roles:**

✅ **Advantages:**
- Organizations have diverse, unpredictable needs
- Reduces Harbor maintenance burden (no code changes for role requests)
- Enables rapid adaptation to new workflows
- Doesn't clutter built-in role list

❌ **Alternative: Add more built-in roles:**
- Cannot cover all use cases
- Each new role requires code review, testing, release cycle
- Built-in role list becomes overwhelming
- Organizations still request customization

**Why system admin only for role creation:**

✅ **Advantages:**
- Centralized governance and security control
- Prevents permission sprawl
- Aligns with enterprise security models
- Simpler initial implementation

❌ **Alternative: Project admins create custom roles:**
- Higher risk of permission sprawl
- Inconsistent roles across projects
- More complex audit trail
- Can be added later if needed

**Why a per-node in-memory cache (not session-scoped, not Redis-on-the-hot-path):**

The caching approach was chosen from a measured comparison (see *Performance Caching*), not
assumed up front.

✅ **Chosen: L1 in-process cache (default), 1 s TTL, optional L2 Redis (off by default):**
- Restores upstream-parity latency (the DB-per-request cost is +14–56 % on role-evaluating
  endpoints — the measured problem)
- Keeps authorization **in-process by default** — no Redis round-trip on the authz hot path,
  avoiding the availability/latency failure mode seen in goharbor/harbor#23335
- Bounded, tunable staleness: role changes converge across nodes within the L1 TTL (1 s)
- Fully disableable (`ROLE_CACHE_L1_MEMORY_TTL=-1`) to fall back to per-request DB evaluation

❌ **Rejected: session-scoped cache** — stale until logout; unacceptable revocation window.
❌ **Rejected: Redis version-key checked per request** — reintroduces a per-request Redis
dependency on authz (the exact hot-path risk of #23335).

## Compatibility

**Backward Compatibility: Fully maintained**

✅ **No breaking changes:**
- All existing APIs maintain contracts
- Built-in roles function identically
- Existing role assignments preserved
- Authentication/authorization flow unchanged
- Database migration is reversible

✅ **Migration strategy:**
- Zero-downtime migration
- Built-in role permissions migrated from `rbac_role.go` to `role_permission` table (e.g. `0190_2.16.0_schema.up.sql`)
- Existing roles automatically marked `is_builtin=true` (inmutable)
- All user/group assignments preserved
- Rollback: Standard Harbor rollback procedure (custom roles become inaccessible but data preserved)

**Future built-in roles.** Because the DB is the single source of truth, adding or changing a
built-in role in a later release is a **migration-only** change: the new definition is inserted
via that version's `up.sql` migration and marked `is_builtin=true`. No code path re-adds
hardcoded permissions, so there is exactly one authoritative definition to review and audit.
The hot lookup (`role_permission` by `role_type` + `role_id` → `permission_policy`) is index-backed
to keep login/CLI-handshake query volume in check.

**API Compatibility:**

- **New endpoints:** `/api/v2.0/roles/*` (no conflicts with existing endpoints)
- **Extended endpoint:** `/api/v2.0/permissions` (already existed for robots, now includes role permissions)
- **Unchanged endpoints:** All existing role assignment APIs work with custom roles (transparent)

**UI Compatibility:**

- New "Roles" section in System Administration (no impact on existing UI)
- Project member assignment dropdown includes custom roles (alongside built-in roles)
- No changes to existing workflows

**Performance Impact** (measured with `goharbor/perf`; see *Performance Caching* for method and full table):

- **Admins:** unaffected — a sysadmin skips the project-role evaluator, so there is no role lookup (feature is free regardless of cache).
- **Project members, default (L1 cache on):** upstream parity on role-evaluating endpoints (overall −0.8 % vs upstream).
- **Project members, cache off (per-request DB):** +14–56 % per endpoint (overall +15.4 %) — the cost the cache removes.
- **Database:** one role_permission lookup per uncached evaluation; the 1 s L1 TTL collapses these to roughly one query per role per node per second.

**Version Compatibility:**

- Harbor instances without custom roles: No impact
- Harbor instances with custom roles: Rollback preserves data but makes custom roles inaccessible
- Upgrade path: Standard Harbor upgrade (no special steps)

## Performance Caching

Moving permission resolution to the database adds a per-request cost. Rather than assume a
caching design, it was **measured** with `goharbor/perf` (k6 / xk6-harbor, 500 VUs,
5000 iters/endpoint, session-cookie auth, warm-up per endpoint) against the exact upstream
commit the fork is based on, so the delta isolates the feature. Sysadmins are excluded — they
skip the project-role evaluator entirely.

**Measured problem** — per-request DB evaluation vs upstream, and the effect of the cache:

| Endpoint | upstream | cache off | L1 (default) | L1 + L2 |
|---|--:|--:|--:|--:|
| get-project | 213 | 246 (+15.9 %) | 216 (+1.6 %) | 210 (−1.2 %) |
| list-project-members | 174 | 203 (+17.1 %) | 172 (−0.7 %) | 169 (−2.8 %) |
| list-repositories | 288 | 350 (+21.3 %) | 285 (−0.9 %) | 292 (+1.1 %) |
| get-repository | 149 | 184 (+23.2 %) | 147 (−1.5 %) | 146 (−2.3 %) |
| list-artifacts | 611 | 649 (+6.3 %) | 599 (−1.9 %) | 599 (−2.0 %) |
| list-artifact-tags | 193 | 246 (+27.2 %) | 194 (+0.5 %) | 195 (+0.8 %) |
| **overall mean** | **271** | **313 (+15.4 %)** | **269 (−0.8 %)** | **268 (−1.1 %)** |

Uncached DB evaluation costs +15.4 % overall (up to +56 % on individual runs); the in-process
cache restores upstream parity. L1-only and L1+L2 are statistically indistinguishable — Redis
adds nothing on the per-request path.

**Design — two tiers, both configurable, Redis off by default:**

| `ROLE_CACHE_L1_MEMORY_TTL` / `ROLE_CACHE_L2_REDIS_TTL` | Behavior | Stale window | Redis on authz path |
|---|---|---|---|
| `1s` / `-1` (**default**) | in-process only | ≤ 1 s cross-node | none |
| `-1` / `-1` | per-request DB | none (always fresh) | none |
| `1s` / `30m` | two-tier | ≤ 1 s cross-node | opt-in only |
| `-1` / `30m` | Redis only | none¹ | every request |

¹ invalidated on change; 30-min TTL self-heals out-of-band DB edits.

- **L1 (in-process, default 1 s):** the dominant freshness control; keeps authorization
  in-process, so the default config puts **no Redis round-trip on the authz hot path** —
  avoiding the failure mode of goharbor/harbor#23335 and the config-cache history
  (#19156/63/64).
- **L2 (Redis, opt-in, off by default):** for operators who want cross-node sharing; when
  enabled it uses the shared `lib/cache.FetchOrSave` helper and **degrades to the DB** if
  Redis is unavailable, so a Redis hiccup never fails an authz check.
- **Invalidation:** a role create/update/delete drops the entry on the writing node
  immediately (and the shared L2 key cluster-wide); other nodes converge within the L1 TTL.

## Implementation

**Current Status: ~75% Complete** — core implementation (Phases 1–4) done; remaining work is test completion (integration/E2E), documentation, and review/polish.

### Phase 1: Foundation (✅ Complete)
- Database schema design and migrations
- Core RBAC permission loading logic
- `role_permission` table integration


### Phase 2: API Layer (✅ Complete)
- `/api/v2.0/roles` endpoints (CRUD operations)
- `/api/v2.0/permissions` endpoint extension
- OpenAPI/Swagger specification



### Phase 3: UI Components (✅ Complete)
- System Administration → Roles management interface
- Role creation/edit wizard
- Permission selection interface
- Built-in vs custom role indicators



### Phase 4: Security Validation (✅ Complete)
- Privilege escalation prevention — **robot creation**: a user may create a robot only with permissions ≤ their own (`validateNoEscalation` / `isValidPermissionScope`), with a robot↔role permission mapping where needed
- Privilege escalation prevention — **member assignment**: a project admin may assign a custom role only if they hold every permission it grants (`checkNoEscalation`), enforced on both member **create** and **update** paths
- Custom-role **definition** restricted to system admins (`checkSysAdmin`); requested permissions constrained to the role permission catalog (`rbac.ScopeRole`)
- Registry (v2) token behavior unchanged: permissions are frozen into the signed JWT at issue time and take effect on the next token negotiation (bounded by `token_expiration`, default 30 min) — standard Docker Registry v2 behavior, identical for project removal, built-in downgrades, and custom roles
- Audit logging for all role operations
- UI security enhancements (disable invalid permissions/roles in forms)


### Phase 5: Testing (🔄 In Progress)
- Unit tests: role CRUD, permission validation, two-tier cache behavior (L1 hit/expiry, L2 hit, invalidate, disabled tiers) — done
- Security tests: escalation attempts covered at both unit and handler level (robot creation, member assign **and** modify paths, sysadmin-gated role definition) — done
- Integration tests (API contracts, multi-user scenarios, migrations) — in progress
- E2E tests (complete workflows via UI) — in progress



### Phase 6: Documentation (🔄 In Progress - 30%)
- User guide (creating and managing custom roles)
- Administrator guide (installation, migration, security)
- API documentation (code examples, migration guide)
- Design documentation (architecture, trade-offs)


### Phase 7: Review & Polish (⏳ Not Started)
- Performance optimization
- Security audit
- Code review addressing feedback
- Release preparation



**Repository:**
- Fork: https://github.com/maxgraustenzel-create/harbor
- Branch: `origin/18124-custom-role-feature`

## Open Issues


### 1. Permission Change Propagation

**Resolved:** Role changes converge across all nodes within the L1 cache TTL (**1 s** by
default). The writing node updates its cache immediately; other nodes refresh on their next
evaluation once their entry expires. Operators who need zero staleness can disable the cache
(`ROLE_CACHE_L1_MEMORY_TTL=-1`) for per-request DB evaluation; those who prefer even lower DB
load can raise the TTL, trading freshness for fewer queries. Session-scoped ("next login")
caching was **withdrawn** as its revocation window was unacceptable.

---

**Feedback Welcome**

This proposal is open for community discussion. Please provide feedback on:
- Architecture and design approach
- Security model and concerns
- API design and compatibility
- UI/UX and usability
- Documentation clarity
- Implementation timeline
- Open issues and alternatives

**Contact:** Max Graustenzel (@maxgraustenzel-create)
