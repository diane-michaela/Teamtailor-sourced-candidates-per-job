# DISCLAIMER — Privacy & Customization 
 =============================================================================
 
 To preserve privacy, the pipeline stages used in this script ("TA Screening",# "HM Interview", "Testing Stage", "Final Interview", "Offer", "Hired") 
 reflect a CLASSIC, GENERIC hiring process. They do NOT match the exact stages or# naming conventions used at the author's current company.
 Before using this script in your own setup, you MUST replace these stage# names with the EXACT names of the stages you have created in your own
 TeamTailor (or other ATS) pipeline. Stage names are matched as plain strings,# so any mismatch (typos, extra spaces, different casing) will silently result# in a count of 0 for that stage.## Places to update if your stage names differ:
 - STAGE_ORDER (the master list)
 - INTERVIEW_STAGES (stages you want to highlight with candidate names)
 - The funnel and alert sections (display labels)

=============================================================================

# Teamtailor Hiring Report — Make Automation

Automated weekly hiring pipeline reports pulled from the Teamtailor ATS and posted to Slack, powered by Make.com

=============================================================================


## Purpose

For each active job opening, this automation:
- Fetches all applications and their pipeline stage from the Teamtailor API
- Calculates pipeline health metrics (conversion rates, sourced vs inbound, days open)
- Detects anomalies and fires alerts (backlog, low quality, high fail rate, offer drop-off)
- Posts a formatted weekly report to the relevant Slack channel

---

## Make Architecture

```
Scheduler (weekly)
    │
    ▼
[Module 1 — Config]          Python Code module
    │  Returns array of       No API calls, no inputs
    │  {job_id, channel}      Edit here to add/remove jobs
    │
    ▼
[Iterator]                   Built-in Make module
    │  Loops once per job     Feeds job_id + channel downstream
    │
    ▼
[Module 2 — Report]          Python Code module  ← master-code-teamtailor-extraction.py
    │  Inputs: job_id,        Calls Teamtailor API, builds Slack message
    │          channel        Returns slack_message + metrics
    │
    ▼
[Slack — Send a Message]     Native Make Slack module
       Channel: {{channel}}
       Message: {{slack_message}}
```

### Why this architecture saves credits

| Old setup | New setup |
|---|---|
| Router → 5× Code → 5× Slack | Config → Iterator → 1× Code → 1× Slack |
| Code module runs 5 times independently | Code module logic runs once per iteration, reused |
| ~10 operations per run | ~4 modules, scales automatically with job count |
| Adding a job = duplicating a branch | Adding a job = one line in Module 1 |

The key saving: the Router fan-out pattern duplicates the heavy Code module for every job. The Iterator pattern runs a single module instance per item — fewer operations billed and a much smaller scenario to maintain.

---

## Files

| File | Purpose |
|---|---|
| `master-code-teamtailor-extraction.py` | Make Module 2 — full report logic. Copy/paste into Make's Python Code module. Module 1 config is included as a comment at the top. |
| `teamtailor_hiring_report.py` | Standalone Python version — run locally with env variables, no Make dependency. |
| `.env.example` | Template for local environment variables. Copy to `.env` and fill in your token. |

---

## Make Setup

### Module 1 — Config
- Type: Code (Python)
- No input variables
- Paste the array from the top comment of `master-code-teamtailor-extraction.py`
- To add a job: add `{"job_id": "XXXXXXX", "channel": "#channel-name"}` to the array
- To change a Slack channel: edit only this module

### Iterator
- Source array: `{{1.[]}}` (output of Module 1)

### Module 2 — Report
- Type: Code (Python)
- Input variables to declare:
  - `job_id` → mapped from `{{2.job_id}}`
  - `channel` → mapped from `{{2.channel}}`
- Paste the full content of `master-code-teamtailor-extraction.py` (below the comment block)

### Slack
- Channel: `{{3.channel}}`
- Message: `{{3.slack_message}}`

---

## Alerts

The report fires the following alerts automatically inside the Slack message:

| Alert | Condition |
|---|---|
| 🚨 Urgent — role open too long | ≥ 90 days since job created |
| ⚠️ Approaching critical delay | 60–89 days since job created |
| ⚠️ Inbox backlog | > 20 candidates active in Inbox + Reviewing |
| ⚠️ Low pipeline quality | < 20% of screened candidates pass TA Screening |
| ⚠️ High use-case fail rate | > 60% of decided candidates fail the use-case interview |
| ⚠️ Low offer acceptance | < 80% of offers result in a hire |

---

## Local Usage

```bash
cp .env.example .env
# Fill in TEAMTAILOR_API_TOKEN and TEAMTAILOR_JOB_ID in .env

export $(cat .env | xargs)
python teamtailor_hiring_report.py
```

---

## API

- **ATS:** [Teamtailor API v1](https://docs.teamtailor.com)
- **API version header:** `20240904`
- **Authentication:** `Token token=YOUR_TOKEN`
- **Endpoints used:** `/v1/jobs/{id}`, `/v1/job-applications`

 labels)
# =============================================================================
