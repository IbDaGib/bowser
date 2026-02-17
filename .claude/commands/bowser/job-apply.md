---
model: sonnet
description: Apply to multiple jobs via job links (Simplify or direct ATS URLs) — multi-agent workflow dispatching to specialized research, writer, and form-filler agents for cost-optimized applications
argument-hint: <job links separated by newlines, commas, or spaces>
---

# Job Apply (v2 — Multi-Agent)

Apply to multiple jobs using job links. Accepts both Simplify links (`https://simplify.jobs/p/...`) and direct ATS URLs (Greenhouse, Lever, Ashby, etc.). Dispatches to specialized agents: a Sonnet research agent screens jobs, a Haiku form filler handles mechanical work, and an Opus writer handles only tailored free-text answers. ~60-75% cheaper than the legacy single-agent approach.

## Workflow

### Step 1 — Parse Links

Extract all URLs from `$ARGUMENTS`. Links may be separated by newlines, commas, or spaces. Deduplicate.

Classify each link:
- **Simplify link:** matches `https://simplify.jobs/p/*` — research agent will extract ATS URL from Simplify page
- **Direct ATS link:** any other job URL (e.g., `boards.greenhouse.io`, `jobs.lever.co`, `jobs.ashbyhq.com`, `*.icims.com`, `myworkdayjobs.com`, company career pages with `gh_jid` params, etc.) — research agent navigates directly

If no valid URLs found, stop with: _"No valid links found. Please provide Simplify links or direct ATS URLs."_

### Step 2 — Pre-load Answer Bank

Read the answer bank file once upfront:
- `answer-bank.md` — static field-answer map + pre-cached free-text answers

Do NOT read `resume.md` or `ibrahim.md` at this level. The specialized agents load what they need via their own skills.

### Step 3 — Process Each Link

For each link sequentially (one Chrome instance, no parallelism):

**Phase A — Research (via @job-research-agent)**

Spawn a `@job-research-agent` with the job URL (Simplify or direct ATS) and the current tab ID.

The research agent will:
1. Navigate to the URL
2. Read the job posting (no screenshots)
3. Run qualification fit check
4. If Simplify link: extract the ATS URL and navigate to it
5. If direct ATS link: already on the ATS page
6. Verify the ATS domain is supported

It returns a structured payload:
```
STATUS: PROCEED | SKIP | NEEDS_VERIFICATION
COMPANY: {name}
TITLE: {title}
JOB_SUMMARY: {summary}
ATS_URL: {url}
SKIP_REASON: {reason, if SKIP}
VERIFICATION_URL: {url, if NEEDS_VERIFICATION}
VERIFICATION_REASON: {reason, if NEEDS_VERIFICATION}
VERIFICATION_ACTION: {instructions for the user, if NEEDS_VERIFICATION}
```

If STATUS is SKIP: log the result and continue to next link.

**Phase A.5 — User Verification (if NEEDS_VERIFICATION)**

If STATUS is `NEEDS_VERIFICATION`:
1. Display a clear pause message to the user:
   ```
   ⚠️  Manual verification required for {COMPANY} ({TITLE})
   ATS: {ATS domain}
   Reason: {VERIFICATION_REASON}

   Please open Chrome and go to: {VERIFICATION_URL}
   Action: {VERIFICATION_ACTION}

   Type "continue" when you're past the login and on the application form.
   Type "skip" to skip this job.
   ```
2. Wait for the user to type "continue" or "skip" in the chat.
3. If "skip": log `RESULT: SKIPPED` with `Notes: User skipped at verification step` and continue to next link.
4. If "continue": proceed to Phase B and C as normal, passing `ATS_URL` to the form filler. The form filler will operate from the current browser state (user is already logged in — do NOT navigate back to the login page).

**Phase B — Pre-generate Tailored Answers (via @job-writer-agent, conditional)**

Review the JOB_SUMMARY. If it suggests the application will have company-specific free-text questions (e.g., "why this company?", role-specific motivation questions), spawn a `@job-writer-agent` with:
- QUESTION_TEXT: "Why are you interested in this role at {COMPANY}?"
- COMPANY, TITLE, JOB_SUMMARY from Phase A

Merge the writer's response with the pre-cached answers from the answer bank. This pre-generation step is optional and only runs when company-specific tailoring is clearly needed.

**Phase C — Fill & Submit (via @job-form-filler-agent)**

Spawn a `@job-form-filler-agent` with:
- ATS_URL from Phase A
- TAB_ID (current tab)
- COMPANY, TITLE from Phase A
- ANSWER_BANK: the full answer-bank.md content
- TAILORED_ANSWERS: any answers from Phase B (may be empty)
- IS_DIRECT_ATS: true if the original link was not a Simplify link

The form filler will:
1. Navigate to the ATS page
2. Attempt Simplify autofill (best-effort — may not be available for direct ATS links)
3. Fill gaps from the answer bank
4. Check agreement checkboxes
5. Handle multi-page forms
6. Handle Gmail verification if needed
7. Submit the application

It returns one of:
- `RESULT: SUCCESS/SKIPPED/FAILED` with details
- `RESULT: NEEDS_ANSWERS` with `UNFILLED_QUESTIONS` list

**Phase C.1 — Handle Unfilled Questions (if NEEDS_ANSWERS)**

If the form filler returns UNFILLED_QUESTIONS:
1. For each unfilled question, spawn a `@job-writer-agent` with the question text, COMPANY, TITLE, and JOB_SUMMARY.
2. Collect the tailored answers.
3. Send the answers back to the form filler agent to continue filling and submitting.

**Phase D — Log Result**

Record the result for each application:
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

## Agent Summary

| Agent | Model | Purpose | Skills |
|-------|-------|---------|--------|
| @job-research-agent | Sonnet | Screen job, qualify, extract ATS URL | claude-bowser |
| @job-form-filler-agent | Haiku | Click, type, check boxes, submit | claude-bowser, gmail-automation |
| @job-writer-agent | Opus | Tailored free-text answers ONLY | resume, ibrahim-profile |

This architecture ensures Opus tokens are only spent on creative writing, while mechanical navigation and form filling run on cheaper models.
