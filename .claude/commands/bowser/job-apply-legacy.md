---
model: opus
description: (Legacy) Original single-agent Opus approach — apply to multiple jobs sequentially via Simplify links
argument-hint: <simplify links separated by newlines, commas, or spaces>
---

# Job Apply

Apply to multiple jobs sequentially using Simplify links. For each link: navigate, let Simplify auto-fill, fill gaps from resume/profile data, handle Gmail verification if needed, and submit.

## Workflow

### Step 1 — Parse Links

Extract all URLs matching `https://simplify.jobs/p/*` from `$ARGUMENTS`. Links may be separated by newlines, commas, or spaces. Deduplicate. Reject any non-Simplify URLs and warn about them.

If no valid Simplify links found, stop with: _"No valid Simplify links found. Please provide links in the format: `https://simplify.jobs/p/...`"_

### Step 2 — Pre-load Data

Read these files once upfront (avoids re-reading per application):
- `resume.md` — factual career data
- `ibrahim.md` — motivation, stories, profile data

### Step 3 — Process Each Link

For each Simplify link sequentially (one Chrome instance, no parallelism), execute the full application workflow:

**Phase 1 — Navigate:** Verify Chrome MCP tools available. Resize to 1440x900. Navigate to the Simplify link. Screenshot.

**Phase 2 — Validate Job & Check ATS Platform:**
- Read page content. Check for "no longer available" / "expired" / "already applied". SKIP if found.
- Extract COMPANY_NAME, JOB_TITLE, JOB_DESCRIPTION summary for tailoring answers.

**Phase 2.5 — Qualification Fit Check:**
- Review JOB_TITLE and JOB_DESCRIPTION against resume data.
- SKIP if any hard disqualifier is found:
  - Requires Master's or PhD degree
  - Requires 5+ years of experience
  - Role is Senior/Staff/Principal/Lead level
  - Role is clearly outside software engineering / ML / AI / full-stack
- Log skip reason: `Unqualified: {reason}` (e.g., "Unqualified: requires Master's degree")
- Do NOT skip for "preferred" qualifications, learnable tech gaps, or 2-3 year experience asks.

- Extract the ATS URL programmatically: run JavaScript to read
  `JSON.parse(document.querySelector('#__NEXT_DATA__').textContent).props.pageProps.jobPosting.url`
  This returns a redirect URL like `https://simplify.jobs/jobs/click/{id}`.
- Navigate directly to that URL in the current tab. It will redirect to the actual ATS.
- Do NOT click the Apply button (it opens a new window outside the MCP tab group).
- Check the resulting URL domain against supported ATS platforms:
  - `jobs.smartrecruiters.com`
  - `jobs.ashbyhq.com`
  - `oraclecloud.com`
- Workday (`myworkdayjobs.com`) is **not supported** — requires account creation which cannot be automated.
  - `jobs.lever.co`
  - `boards.greenhouse.io`
- If domain not supported: **SKIP** — note `Unsupported ATS: {domain}`.

**Phase 3 — Wait for Simplify Extension:**
After the ATS page loads, look for the Simplify extension sidebar (find "Autofill this page" button in the DOM). Click it to trigger autofill.
Then snapshot-polling: read accessibility tree, wait 5s, read again. If fields changed, Simplify is still working — re-check. If stable after 2 consecutive reads, done. Max 20s total.
After polling: check if Simplify actually activated (overlay, auto-filled values, Simplify UI). If not: **SKIP** — `Simplify not active on this page`.

**Phase 4 — Fill Gaps:**
Read all interactive form elements. Identify unfilled fields. Fill based on category:

| Category | Strategy | Source |
|----------|----------|--------|
| Factual (citizenship, salary, authorization, start date, links) | Direct value | `resume.md`, `ibrahim.md` |
| Free-text (motivation, challenges, "why this company?") | Tailored answer using profile Quick-Reference + company context | `ibrahim.md` + job context |
| Select/Radio/Checkbox | Best match from options | Both files |
| Job-specific questions | Answer from profile data | `ibrahim.md` |

Free-text style: paragraphs not bullets, conversational tone, no buzzwords, no em dashes, story-driven, 3-5 sentences.

Agreement/policy checkboxes (e.g., "I agree to hybrid work model") are commonly missed by Simplify. Always check and fill these.

Scroll down to check for more fields. Handle multi-page forms (repeat Phase 4 after "Next"/"Continue").

**Phase 5 — Gmail Verification (if triggered):**
Use `gmail-automation` skill (Rube MCP), NOT browser-based Gmail.
- Code verification: `RUBE_SEARCH_TOOLS` → `GMAIL_FETCH_EMAILS` (query: `is:unread newer_than:5m`) → `GMAIL_FETCH_MESSAGE_BY_MESSAGE_ID` (include_payload: true) → extract code → enter in form.
- Link verification: same fetch steps → extract URL → navigate to it in browser → return to application.

**Phase 6 — Submit:** Click submit → verify confirmation text.

**Phase 7 — Collect Result:**
```
RESULT: SUCCESS | SKIPPED | FAILED
Company: {COMPANY_NAME}
Position: {JOB_TITLE}
URL: {JOB_URL}
ATS: {domain}
Notes: {details}
```

### Step 4 — Summary Report

After all links are processed, output a summary table:

```
# Job Application Summary
**Date:** {current date}
**Total:** N | **Success:** X | **Skipped:** Y | **Failed:** Z

| # | Company | Position | ATS | Status | Notes |
|---|---------|----------|-----|--------|-------|
| 1 | Acme | SWE | SmartRecruiters | SUCCESS | Applied |
| 2 | Corp | Backend | Lever | SUCCESS | Applied |
| 3 | Startup | Frontend | — | SKIPPED | Job no longer available |
```

The ATS column and skip reasons give visibility into which platforms need future support.
