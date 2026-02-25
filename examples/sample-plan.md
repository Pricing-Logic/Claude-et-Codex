## Objective
Add a user preferences API to the web app, allowing users to save and retrieve UI preferences (theme, language, sidebar state) via a REST endpoint backed by PostgreSQL.

## Key Files to Examine
- `app/api/` — existing API route structure and conventions
- `lib/db.ts` — database connection and query helpers
- `middleware.ts` — auth middleware that protects API routes
- `types/user.ts` — existing user type definitions

## Files to Modify
- `lib/db.ts` — add `preferences` table queries (getPreferences, upsertPreferences)
- `middleware.ts` — ensure `/api/preferences` is behind auth
- `types/user.ts` — add `UserPreferences` type

## Files to Create
- `app/api/preferences/route.ts` — GET and PATCH handlers
- `migrations/003_add_preferences.sql` — create preferences table

## Architecture Decisions
- **Upsert pattern**: PATCH uses INSERT ... ON CONFLICT UPDATE rather than separate create/update flows
- **JSON column**: preferences stored as JSONB for flexibility, with TypeScript type validation at the API layer
- **No caching**: preferences are small and infrequently read; direct DB query is fine for now

## Constraints / Non-Goals
- No real-time sync across devices (not needed yet)
- No preferences UI in this PR — API only
- No migration runner integration — SQL file for manual execution
- Not adding rate limiting to this internal endpoint

## Sequence
1. Create migration SQL
2. Add TypeScript types
3. Add DB query functions
4. Create API route handlers
5. Verify middleware coverage
