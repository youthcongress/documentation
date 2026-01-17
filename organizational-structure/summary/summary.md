# Organizational Structure — Summary

## Overview

This document summarizes the current and target organizational structure based on the raw notes in the `details/raw.md` source. It defines terms, committee layers, roles, departments, access-level principles, election/transition rules, and a recommended database schema to support flexible growth across terms (e.g., 2.0 → 2.1).

## Key Concepts

- Term: a versioned period of the organization's structure and leadership (e.g., 1.0, 2.0, 2.1). Terms capture which positions are active and any structural variations.
- Levels: three governance layers — Central Committee, Provincial Committee, District Committee.
- Flexibility: structure should allow adding/removing committees and departments, and varying member counts per level while keeping consistent role semantics and access mappings.

## Term 2.0 (current short-term setup)

Purpose: a lean, short-term committee focused on preparing a stable 2.1 organizational setup.

Central Committee (example counts)
- President — 1
- General Secretary — 1
- Deputy General Secretary — 1
- Treasurer — 1
- Committee Members — 29

Province Level (example per-province team in 2.0)
- Province Coordinator — 1
- Province Committee Members — 18
(Team size = 19)

District Level (example per-district team in 2.0)
- District Coordinator — 1
- District Committee Members — 4
(Team size = 5)

Notes: In 2.0 some provinces/districts may still be transitioning; election timing may vary across locations.

## Term 2.1 (target/stable setup)

Term 2.1 represents the fuller organization model and standardizes role names and counts across levels.

Central Committee (same as 2.0)
- President — 1
- General Secretary — 1
- Deputy General Secretary — 1
- Treasurer — 1
- Committee Members — 29

Province Level (per province)
- Province President — 1
- Province General Secretary — 1
- Province Deputy General Secretary — 1
- Province Treasurer — 1
- Province Committee Members — 15

District Level (per district)
- District President — 1
- District General Secretary — 1
- District Deputy General Secretary — 1
- District Treasurer — 1
- District Committee Members — 15

Notes: counts above (e.g., 15 committee members) are the recommended stable defaults for 2.1 and may be enforced as the canonical size in the schema while allowing overrides per unit if explicitly needed.

## Departments (applies across layers where enabled)

Common departments listed in the raw notes (each department can be enabled at Central/Province/District depending on phase):
1. Policy, Research and Training
2. International Relations and Diplomacy
3. Education, Health and Sports
4. Human Rights, Parliamentary Affairs and Legal
5. Environmental and Sustainability
6. Economic Affairs
7. Social Welfare and Inclusion
8. Information, Communication and Publicity
9. Organization, Coordination and Youth Engagement

Department positions:
- Department Lead — 1
- Department Co-Lead — 1
- Department Members — 10 (default)

Rule: departments are created when a committee/level moves into a phase that enables departments (e.g., 2.1). Departments follow the same position template across levels.

## Standing Committees

- Accounts Committee — Committee Lead (1)
- Discipline Committee — Committee Lead (1)

Committees can be added/removed; each committee should have a single lead and optional members.

## Access Levels & Role Mapping

Principles:
- Access levels are defined by capabilities (permissions) rather than display names.
- Access mappings reference canonical roles (as defined for the stable term, e.g., 2.1). Even when a unit is in 2.0 and uses a different display title (e.g., "Province Coordinator"), the underlying access level maps to the canonical 2.1 role (e.g., `province-president` capability group).

Suggested capability groups (examples):
- system-admin: full access across all terms and levels
- central-admin: manage central-level data and users
- province-admin: manage province-level data and users within assigned province(s)
- district-admin: manage district-level data and users within assigned district(s)
- department-lead: manage department resources and members
- member: read/update limited profile fields

Mapping guidance:
- Use an `access_levels` table that maps canonical role slugs (e.g., `province-president`) to permissions sets. Assign `access_level_id` to memberships in the `memberships` table.
- When showing UI labels, use the term-aware display name (e.g., show "Province Coordinator" for 2.0 but keep `access_level: province-president`).

## Election & Transition Rules

- Elections can occur independently across provinces and districts. The central committee may remain stable while lower levels transition.
- When a level transitions to the next term, new positions may appear (e.g., department leads) — migration scripts should:
  1. Create new positions with default vacancies
  2. Map existing members to canonical access levels where appropriate
  3. Preserve historical `term` on memberships for audit

## Database Schema Recommendations

Goal: support flexible hierarchies, term/versioned structures, per-unit overrides, and permission mapping.

Minimal core tables (columns shown for guidance):

1) `terms`
  - `id` (pk)
  - `slug` (e.g., '2.0')
  - `title`
  - `started_at`, `ended_at`
  - `notes`

2) `units` (represents Central / Province / District / other organizational units)
  - `id` (pk)
  - `type` (enum: central, province, district)
  - `parent_id` (nullable FK -> units.id)
  - `name`
  - `code` (short unique code)
  - `metadata` (json)

3) `positions`
  - `id` (pk)
  - `slug` (canonical role slug, e.g., 'province-president')
  - `display_name` (default label for the canonical role)
  - `default_count` (default number of seats for this position per unit)
  - `level` (which unit type it applies to: central/province/district/global)

4) `units_positions` (defines which positions are active for a unit in a specific term)
  - `id` (pk)
  - `unit_id` (fk)
  - `position_id` (fk)
  - `term_id` (fk)
  - `count` (overrides `positions.default_count`)

5) `members`
  - `id` (pk)
  - `person_id` (fk -> people table)
  - `username`, `email`, `profile` (json)

6) `memberships` (who holds which position / which access level)
  - `id` (pk)
  - `member_id` (fk)
  - `unit_id` (fk)
  - `position_id` (fk)
  - `term_id` (fk)
  - `access_level_id` (fk)
  - `start_date`, `end_date`
  - `status` (active, pending, retired)

7) `departments`
  - `id` (pk)
  - `unit_id` (fk)
  - `slug`, `name`, `description`

8) `access_levels`
  - `id` (pk)
  - `slug` (e.g., 'province-admin')
  - `permissions` (json array or jsonb)

9) `committees` (optional table for named committees like Accounts)
  - `id`, `unit_id`, `slug`, `name`, `description`

Example SQL (simplified) — create `terms` and `units`:

```sql
CREATE TABLE terms (
  id SERIAL PRIMARY KEY,
  slug VARCHAR(16) UNIQUE NOT NULL,
  title TEXT,
  started_at TIMESTAMP,
  ended_at TIMESTAMP,
  notes TEXT
);

CREATE TABLE units (
  id SERIAL PRIMARY KEY,
  type VARCHAR(32) NOT NULL,
  parent_id INTEGER REFERENCES units(id),
  name VARCHAR(255) NOT NULL,
  code VARCHAR(64),
  metadata JSONB DEFAULT '{}'
);
```

Design notes:
- Use JSONB for `metadata` to allow unit-level customizations without schema changes.
- Use `units_positions` to version which positions are active per term.
- Keep `access_levels.permissions` as a JSON array of permission keys to simplify RBAC checks.

## Example: Province/District Data Flow

- A `province` unit has child `district` units (parent-child via `units.parent_id`).
- When a province moves from 2.0 → 2.1:
  - Create `units_positions` rows for departments that now exist
  - Populate `memberships` with empty slots or migrated members
  - Assign `access_level` values to existing members according to canonical mapping

## Migration & Versioning Recommendations

- Provide migration scripts that:
  - Read the previous term's `units_positions` and create the next term's configuration, applying defaults and any overrides
  - Produce an audit log of mapping changes
- Maintain historical `memberships` for reporting and rollback

## UX / Display Considerations

- The UI should be term-aware — when rendering role names show the localized/display label for the unit's current term while using canonical slugs for backend logic.
- When editing access levels or positions, surface the `term` context prominently.

## Next Steps

1. Review this summary and confirm default counts per level (e.g., committee members per province/district).
2. Accept or adjust the department list and default member counts.
3. If agreed, I can generate migration SQL and an ER diagram for the schema above.

---

Generated from: `documentation/organizational-structure/details/raw.md`
