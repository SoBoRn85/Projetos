---
description: Database design workflow. Schema design, ORM selection, indexing strategy, query optimization, and safe migrations. Use for new databases, schema changes, or performance issues.
---

# /db - Database Design

$ARGUMENTS

---

## Purpose

This command activates the `database-design` skill for designing schemas, selecting databases/ORMs, planning indexes, and optimizing queries.

---

## Sub-commands

```
/db design <feature>    - Design new schema for a feature
/db select              - Choose the right database for the project
/db orm                 - Choose the right ORM
/db index               - Analyze and optimize indexes
/db optimize <query>    - Optimize a slow query
/db migrate             - Plan safe schema migration
/db audit               - Full database health audit
```

---

## Behavior

When `/db` is triggered:

### Step 1: Clarify Context

**ALWAYS ask** before designing:
1. What database are you using (or what are the options)?
2. What is the deployment environment? (serverless, VPS, cloud)
3. What are the scale requirements? (users, records, RPS)
4. Are there existing tables to extend, or is this greenfield?

> ⚠️ **Never default to PostgreSQL without asking.** SQLite may suffice for simple apps. Turso for edge. Neon for serverless.

### Step 2: Database Selection Guide

| Database | Best For |
|----------|----------|
| **PostgreSQL** | Complex queries, full-text search, JSONB |
| **SQLite** | Simple apps, embedded, low traffic |
| **Neon** | Serverless, scale-to-zero |
| **Turso** | Edge computing, global distribution |
| **MongoDB** | Document-heavy, flexible schema |
| **Redis** | Caching, sessions, real-time |

### Step 3: ORM Selection Guide

| ORM | Best For |
|-----|----------|
| **Drizzle** | TypeScript, type-safe, lightweight |
| **Prisma** | Prototyping, auto-migrations, good DX |
| **Kysely** | Raw SQL control, complex queries |

### Step 4: Schema Design Checklist

Before finalizing schema:

- [ ] Asked user about database preference?
- [ ] Chosen database for THIS context?
- [ ] Considered deployment environment?
- [ ] Used UUIDs or appropriate PKs?
- [ ] Defined relationships (1:1, 1:N, N:N)?
- [ ] Planned index strategy?
- [ ] Considered soft deletes (if needed)?
- [ ] Added `created_at` / `updated_at`?

### Step 5: Indexing Strategy

| Index Type | Use When |
|------------|----------|
| Single column | Frequent WHERE on one column |
| Composite | Multiple columns in same WHERE/ORDER |
| Partial | Subset of rows (e.g., `WHERE active = true`) |
| Unique | Enforce uniqueness (email, slug) |
| Full-text | Text search |

### Step 6: Migration Safety Rules

```sql
-- ✅ Safe: Add nullable column (no downtime)
ALTER TABLE users ADD COLUMN bio TEXT;

-- ✅ Safe: Add column with default
ALTER TABLE users ADD COLUMN plan VARCHAR(20) DEFAULT 'free';

-- ⚠️ Careful: Add NOT NULL (requires backfill first)
-- 1. Add nullable
-- 2. Backfill existing rows
-- 3. Add NOT NULL constraint

-- ⚠️ Careful: Rename/drop column (need 3-phase deploy)
-- Phase 1: Write to both old + new columns
-- Phase 2: Read from new column
-- Phase 3: Drop old column
```

---

## Output Format

```markdown
## 🗄️ Database Design: [Feature/Project]

### Selected Stack
- Database: [choice + reason]
- ORM: [choice + reason]

### Schema Design

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### Relationships
- users → posts (1:N)
- posts → tags (N:N via post_tags)

### Index Plan
| Table | Column(s) | Type | Reason |
|-------|-----------|------|--------|
| users | email | Unique | Login lookup |
| posts | user_id, created_at | Composite | User feed |

### Migration Plan
1. [Step 1 - safe to run]
2. [Step 2 - safe to run]
3. [Step 3 - requires maintenance window?]

### Anti-Patterns Avoided
- ✅ No SELECT * in application queries
- ✅ No N+1 queries in ORM usage
- ✅ No unindexed foreign keys
```

---

## Common Anti-Patterns

```
❌ Default to PostgreSQL for simple apps
❌ Skip indexing foreign keys
❌ SELECT * in production queries
❌ Store JSON when structured data is better
❌ Ignore N+1 queries
❌ Drop columns without 3-phase migration
❌ Add NOT NULL without backfilling first
```

---

## Examples

```
/db design user authentication with roles
/db select (choosing between Neon vs SQLite)
/db orm (React app on Vercel)
/db index users table
/db optimize slow search query
/db migrate add soft deletes to posts
/db audit
```

---

## Key Principles

- **Ask before assuming** - always confirm DB choice with user
- **Context decides everything** - serverless ≠ VPS ≠ edge
- **Index foreign keys always** - otherwise JOINs are full table scans
- **Safe migrations** - never break production with schema changes
- **Measure first** - use EXPLAIN ANALYZE before adding indexes
