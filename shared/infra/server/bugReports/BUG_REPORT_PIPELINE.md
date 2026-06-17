# Bug Report Pipeline

This folder contains the server-only implementation for turning Tally bug reports into Supabase records and GitHub issues.

```txt
Tally form
  -> app/api/webhooks/tally/bug-report
  -> Supabase bug_reports row
  -> immediate processing scheduled with Next.js after()
  -> DeepSeek cleanup
  -> Supabase Storage attachment copy
  -> GitHub issue
  -> app/api/process-bug-reports cron retry safety net
```

The Tally webhook is push-based. Tally sends a request immediately when a user submits the form. The 5-minute Vercel Cron retry is not polling Tally; it only picks up reports that were safely stored but not fully processed.

## Manual Setup Checklist

### Supabase

Create or select the KanaDojo Supabase project.

Create a private Storage bucket:

```txt
bug-report-attachments
```

Run this SQL:

```sql
create table bug_reports (
  id uuid primary key default gen_random_uuid(),
  source text not null default 'tally',
  source_submission_id text unique,
  status text not null default 'received',
  raw_payload jsonb not null,
  normalized_payload jsonb,
  cleaned_payload jsonb,
  title text,
  description text,
  severity text,
  labels text[] not null default '{}',
  github_issue_number integer,
  github_issue_url text,
  attempts integer not null default 0,
  last_error text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table bug_report_attachments (
  id uuid primary key default gen_random_uuid(),
  bug_report_id uuid not null references bug_reports(id) on delete cascade,
  original_url text,
  original_name text,
  storage_path text not null,
  mime_type text,
  size_bytes integer,
  created_at timestamptz not null default now()
);
```

Keep the tables and bucket private for v1. The server uses `SUPABASE_SERVICE_ROLE_KEY`; do not expose it to client code.

### GitHub

Create a fine-grained personal access token for `lingdojo/kana-dojo` with:

```txt
Issues: Read and write
Metadata: Read
```

Store it in Vercel as `GITHUB_PAT`.

### DeepSeek

Create an API key and store it as `DEEPSEEK_API_KEY`.

Optional model override:

```txt
DEEPSEEK_BUG_REPORT_MODEL=deepseek-chat
```

### Vercel

Add these env vars to Production and Preview:

```txt
TALLY_WEBHOOK_SECRET=
TALLY_WEBHOOK_TOKEN=
BUG_REPORT_PROCESSOR_SECRET=

DEEPSEEK_API_KEY=
DEEPSEEK_BUG_REPORT_MODEL=deepseek-chat

SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
SUPABASE_BUG_REPORT_BUCKET=bug-report-attachments

GITHUB_PAT=
GITHUB_REPO_OWNER=lingdojo
GITHUB_REPO_NAME=kana-dojo
```

`CRON_SECRET` is already used by existing cron routes. The processor route accepts either `BUG_REPORT_PROCESSOR_SECRET` or `CRON_SECRET`.

### Tally

In the Tally bug report form:

1. Open `Integrations`.
2. Choose `Webhooks`.
3. Add endpoint URL:

```txt
https://kanadojo.com/api/webhooks/tally/bug-report
```

4. Add custom HTTP header:

```txt
Authorization: Bearer <TALLY_WEBHOOK_TOKEN>
```

5. Add the signing secret matching `TALLY_WEBHOOK_SECRET`.
6. Submit one test report without an attachment.
7. Submit one test report with a screenshot.

## Module Responsibilities

`auth.ts`
: Verifies bearer tokens and Tally HMAC signatures. Tally signatures are checked by calculating HMAC SHA-256 over `JSON.stringify(payload)` and comparing the base64 digest to `Tally-Signature`.

`tally.ts`
: Converts Tally's flexible field payload into a stable `NormalizedBugReport`. Field labels are matched with forgiving aliases so the Tally form can evolve without requiring immediate code changes.

`supabaseAdmin.ts`
: Creates the server-only Supabase client with the service role key and provides the bug report Storage bucket name.

`attachments.ts`
: Downloads Tally file upload URLs, validates MIME type and size, copies accepted images to Supabase Storage, inserts `bug_report_attachments` metadata, and creates signed URLs for GitHub issue links.

`deepseek.ts`
: Sends normalized report data to DeepSeek, validates the JSON response with `zod`, clamps labels/title/severity, and falls back to raw normalized fields when DeepSeek is unavailable or returns invalid JSON.

`github.ts`
: Formats the final GitHub issue body and creates the issue using the GitHub REST API.

`processor.ts`
: Orchestrates the full processing transaction: load pending reports, mark attempts, copy attachments, format with DeepSeek, create the GitHub issue, and update Supabase status.

## Status Lifecycle

```txt
received
processing
github_created
retryable_error
failed
```

Rules:

- `received`: raw report stored, not yet fully processed.
- `processing`: an immediate or cron processor run is working on the report.
- `github_created`: GitHub issue was created and stored back on the row.
- `retryable_error`: GitHub or another retryable processing step failed.
- `failed`: the report reached 5 attempts.

## Attachment Policy

Accepted MIME types:

```txt
image/png
image/jpeg
image/webp
```

Limits:

```txt
max files per report: 3
max file size: 10 MB
storage path: bug-reports/<report-id>/<uuid>-<safe-file-name>
```

Attachment failures are non-fatal. The GitHub issue still gets created, and the attachment error is included in triage notes.

## DeepSeek Contract

DeepSeek is used as a formatter, not as the source of truth. The code still enforces all important constraints.

The prompt asks DeepSeek to return JSON with:

```json
{
  "title": "Short GitHub issue title, max 90 characters",
  "summary": "Clear one-paragraph summary of the bug",
  "stepsToReproduce": ["Step 1", "Step 2"],
  "expectedBehavior": "What the user expected, or null",
  "actualBehavior": "What happened instead, or null",
  "environment": {
    "pageUrl": "URL or null",
    "feature": "Feature area or null",
    "device": "Device or null",
    "browser": "Browser or null",
    "locale": "Locale/language or null"
  },
  "severity": "low | medium | high | critical",
  "labels": ["bug", "user-report", "needs-triage"],
  "triageNotes": "Brief maintainer-facing notes, or null",
  "missingInfo": ["Question that would help debugging"]
}
```

The code enforces:

```txt
title <= 90 characters
labels are whitelist-only
bug, user-report, needs-triage are always present
severity is low, medium, high, or critical
invalid JSON falls back to normalized raw fields
```

Allowed labels:

```txt
bug
user-report
needs-triage
accessibility
mobile
desktop
regression
```

## GitHub Issue Output

Each issue includes:

```txt
Summary
Steps To Reproduce
Expected Behavior
Actual Behavior
Environment
Attachments
Triage Notes
Missing Info
Source
```

The source section includes the Supabase report ID and Tally submission ID so maintainers can trace a GitHub issue back to the stored raw payload.

## Rollout Checklist

- [ ] Create Supabase tables.
- [ ] Create private Supabase Storage bucket.
- [ ] Add Vercel env vars.
- [ ] Deploy to Preview.
- [ ] Temporarily point Tally webhook at Preview if desired.
- [ ] Submit one Tally report without an attachment.
- [ ] Confirm a `bug_reports` row is created.
- [ ] Confirm a GitHub issue is created.
- [ ] Submit one report with a screenshot.
- [ ] Confirm `bug_report_attachments` metadata is created.
- [ ] Confirm the GitHub issue includes the attachment link.
- [ ] Deploy to Production.
- [ ] Point Tally webhook at production.
- [ ] Submit a production smoke-test report.
- [ ] Monitor Vercel logs and Supabase statuses for 24 hours.

## Verification

Use:

```powershell
npx tsc --noEmit --incremental --pretty
npx eslint app\api\webhooks\tally\bug-report\route.ts app\api\process-bug-reports\route.ts shared\infra\server\bugReports\*.ts
```

Do not use `npm run build` for verification.
