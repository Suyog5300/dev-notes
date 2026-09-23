# ADM ATS: Database Changes

**Database:** `adm_db` on `103.171.45.223:9871`

All changes are additive. No existing tables, columns or stored functions were modified or dropped.

## 1. New Tables

### `cv_resume_versions` (CV versioning)
Keeps every resume uploaded for a candidate. Status: created and working.

```sql
CREATE TABLE public.cv_resume_versions (
  id SERIAL PRIMARY KEY,
  cv_candidate_id INTEGER NOT NULL REFERENCES public.cv_candidates(cv_candidate_id),
  file_id TEXT NOT NULL,
  original_filename TEXT,
  uploaded_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  uploaded_by CHARACTER VARYING NOT NULL
);
```

### `cv_status_history` (status and remark trail)
Logs every status change, plus remarks saved from the Interview Tracking page. Status: not confirmed.

```sql
CREATE TABLE public.cv_status_history (
  id SERIAL PRIMARY KEY,
  cv_candidate_id INTEGER NOT NULL REFERENCES public.cv_candidates(cv_candidate_id),
  status TEXT NOT NULL,
  remark TEXT,
  changed_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  changed_by CHARACTER VARYING NOT NULL
);
```

### `cv_roles` (Function & Role Master)
Roles or positions under a function, e.g. "Frontend Developer" under "Software". Status: not confirmed.

```sql
CREATE TABLE public.cv_roles (
  id SERIAL PRIMARY KEY,
  function_id INTEGER NOT NULL REFERENCES public.cv_functions(id),
  role_name TEXT NOT NULL,
  is_active INTEGER NOT NULL DEFAULT 1,
  created_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  created_by CHARACTER VARYING NOT NULL,
  updated_at TIMESTAMP WITHOUT TIME ZONE NOT NULL DEFAULT NOW(),
  updated_by CHARACTER VARYING
);
```

## 2. New Columns on `cv_candidates` (interview scheduling)
One date per round, plus a skip flag per round. Status: created and working.

```sql
ALTER TABLE public.cv_candidates
  ADD COLUMN IF NOT EXISTS first_round_date  date,
  ADD COLUMN IF NOT EXISTS second_round_date date,
  ADD COLUMN IF NOT EXISTS third_round_date  date,
  ADD COLUMN IF NOT EXISTS hr_round_date     date,
  ADD COLUMN IF NOT EXISTS first_round_skipped  boolean NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS second_round_skipped boolean NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS third_round_skipped  boolean NOT NULL DEFAULT false,
  ADD COLUMN IF NOT EXISTS hr_round_skipped     boolean NOT NULL DEFAULT false;
```

## 3. Data Added Through the App (not SQL)

- **Status Master:** New, Under Review, Shortlisted, Hold, Interview Scheduled, Offer Released, Joined.
- The existing Pending, Rejected and Selected were kept.

## 4. Not Changed

- Stored functions (`f_insert_cv_candidate_data`, `f_update_cv_candidate_data`, `f_get_cv_candidate_data`, ...) are untouched. Interview dates are merged into the list by the Python service.
- `roles` / RBAC tables: no schema change. The Role Master page uses the existing tables.

## Rollback

```sql
DROP TABLE IF EXISTS public.cv_resume_versions;
DROP TABLE IF EXISTS public.cv_status_history;
DROP TABLE IF EXISTS public.cv_roles;

ALTER TABLE public.cv_candidates
  DROP COLUMN IF EXISTS first_round_date,
  DROP COLUMN IF EXISTS second_round_date,
  DROP COLUMN IF EXISTS third_round_date,
  DROP COLUMN IF EXISTS hr_round_date,
  DROP COLUMN IF EXISTS first_round_skipped,
  DROP COLUMN IF EXISTS second_round_skipped,
  DROP COLUMN IF EXISTS third_round_skipped,
  DROP COLUMN IF EXISTS hr_round_skipped;
```
