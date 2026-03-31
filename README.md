# gcc-job-scraper-n8n

**Automated GCC job search pipeline built with n8n, Ollama (local LLM), and the Anthropic Claude API.**

Scrapes 60+ Global Capability Center career pages and job boards across Pune, Bangalore, and Hyderabad — filters by role family, scores each listing against a target candidate profile using Claude, generates a tailored ATS-optimised resume, and delivers a match report via Gmail with the resume saved to Google Drive.

Part of the [Open Mind Lab](https://github.com/itsolutions413544-boop) applied AI automation research series.

---

## What it does

Two parallel workflows handle job discovery:

| Path | Trigger | Description |
|---|---|---|
| **Path A** | Daily cron — 7AM weekdays | Auto-scrapes 60+ GCC career pages and job boards |
| **Path B** | HTTP POST webhook | Manual JD paste — analyse any single job description on demand |

Both paths run the same downstream pipeline:

```
JD Input
  → Ollama llama3.2      — parse JD structure into JSON (local, zero API cost)
  → Claude Sonnet        — score match against candidate profile (0–100)
  → IF score ≥ 70
      → Claude Sonnet    — generate tailored ATS-optimised resume
      → [parallel]
          Google Drive   — save resume as Google Doc
          Gmail          — send match report with score, strengths, gaps, cover letter opening
          Gotenberg      — convert resume to DOCX
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  PATH A — Auto Scraper                                          │
│                                                                 │
│  Cron 7AM                                                       │
│    │                                                            │
│    ▼                                                            │
│  Build URLs (62 searches)                                       │
│    │  Tier 1: Pune GCCs    — Shell, Barclays, Deutsche Bank,   │
│    │                          Cummins, VWITS, Eaton,           │
│    │                          Honeywell, HSBC, BP, Maersk      │
│    │  Tier 2: Blr/Hyd GCCs — Goldman, Bosch, Siemens, EY GDS, │
│    │                          JPMorgan, Deloitte, Novartis,    │
│    │                          Fidelity, Unilever, Cargill      │
│    │  Tier 3: Aggregators  — Naukri.com (8 role searches)      │
│    │                          LinkedIn (14 company filters)     │
│    │                                                            │
│    ▼                                                            │
│  HTTP Scrape (parallel per URL)                                 │
│    │                                                            │
│    ▼                                                            │
│  Parse HTML/JSON → extract title, company, location, URL        │
│    │                                                            │
│    ▼                                                            │
│  IF valid listing?  ──── NO ──→ [drop]                         │
│    │ YES                                                        │
│    ▼                                                            │
│  6-Family Keyword Filter                                        │
│    │  ✓ SIAM / Service Integration                             │
│    │  ✓ IT Change Manager / Change Analyst                     │
│    │  ✓ IT Service Delivery Manager                            │
│    │  ✓ IT Operations Manager                                  │
│    │  ✓ Workforce / Capacity Planning                          │
│    │  ✓ ServiceNow Specialist / Consultant                     │
│    │                                                            │
│    ▼                                                            │
│  IF relevant?  ──── NO ──→ [drop — saves API credits]          │
│    │ YES                                                        │
│    │                                                            │
│    └──────────────────────────────────────────────────────┐    │
│                                                           │    │
└───────────────────────────────────────────────────────────│────┘
                                                            │
┌───────────────────────────────────────────────────────────│────┐
│  PATH B — Manual JD Paste                                 │    │
│                                                           │    │
│  Webhook POST { title, company, jd_text, url }           │    │
│    │                                                      │    │
│    ▼                                                      │    │
│  Normalise payload                                        │    │
│    │                                                      │    │
│    └──────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
           ┌──────────────────────────────────┐
           │  SHARED PIPELINE                 │
           │                                  │
           │  Ollama llama3.2 (local)         │
           │    └─ Parse JD → JSON structure  │
           │                                  │
           │  Claude Sonnet (API)             │
           │    └─ Score 0–100 vs profile     │
           │                                  │
           │  IF score < 70 → skip            │
           │                                  │
           │  IF score ≥ 70:                  │
           │    Claude Sonnet (API)           │
           │      └─ Generate tailored resume │
           │                                  │
           │    ┌─────────────────────────┐   │
           │    │ Google Drive  │  Gmail  │   │
           │    │ Save resume   │  Alert  │   │
           │    │               │         │   │
           │    │    Gotenberg (DOCX)     │   │
           │    └─────────────────────────┘   │
           └──────────────────────────────────┘
```

---

## GCC sources covered

### Tier 1 — Pune (direct career pages)

| Company | Role families targeted |
|---|---|
| Shell Business Operations | ITSM, Change Manager |
| Barclays India | Change Analyst, ITSM, Service Delivery |
| Deutsche Bank Tech Centre | IT Change Manager, ITSM |
| Cummins GCC | Service Delivery, Change Management, ServiceNow |
| Volkswagen Group Technology Solutions (VWITS) | IT Service Manager, ServiceNow |
| Eaton India Innovation Centre | IT Operations, Change Manager |
| Honeywell Technology Solutions | ITSM, Change Manager, ServiceNow |
| HSBC GCC India | IT Change Manager, ITSM |
| BP GCC India | IT Service Manager, Change Manager |
| Maersk GCC India | ITSM, Service Delivery |
| Mercedes-Benz Tech India | IT Operations |

### Tier 2 — Bangalore + Hyderabad (direct career pages)

| Company | Role families targeted |
|---|---|
| Goldman Sachs India | Change Manager, IT Service Manager |
| Bosch India | IT Service Manager, Change Manager |
| Siemens GCC India | ITSM, ServiceNow, Change Manager |
| EY GDS India | IT Change Manager, ITSM, ServiceNow |
| Unilever GCC | IT Service Manager, Workforce Planning |
| Cargill India | IT Operations, Change Manager |
| Telstra India | IT Operations, Service Delivery |
| American Express India | IT Operations, Change Manager |
| Nokia India GCC | IT Service Manager |
| AstraZeneca India | IT Change Manager, ITSM |
| JPMorgan Chase India | IT Change Manager, ITSM |
| Deloitte USI | IT Change Manager, ITSM, ServiceNow |
| Novartis India | IT Governance, Change Manager |
| Standard Chartered GCC | ITSM |
| 3M India GCC | IT Change Manager |
| Fidelity Investments India | Change Management, ITSM |
| DP World India | IT Operations |

### Tier 3 — Aggregators (Naukri + LinkedIn)

| Source | Searches |
|---|---|
| Naukri.com | 8 role-family searches across Pune, Bangalore, Hyderabad |
| LinkedIn (guest API) | 14 company-filtered searches with 24h recency filter |

---

## Role families — keyword filter

All 6 families are checked against job title + description before any API call is made:

| Family | Keywords matched |
|---|---|
| SIAM / Service Integration | siam, service integration, multi-vendor governance |
| IT Change Manager / Analyst | change manager, change analyst, CAB, ITIL change |
| IT Service Delivery Manager | service delivery manager, SDM, managed services delivery |
| IT Operations Manager | IT operations manager, ITSM manager, IT service manager |
| Workforce / Capacity Planning | workforce planning, capacity planning, resource manager, rebadging |
| ServiceNow Specialist | servicenow, now platform, SNOW consultant |

Roles matching software engineer, data scientist, recruiter, and other irrelevant titles are excluded before Ollama runs — saving local compute and API credits.

---

## Tech stack

| Component | Purpose |
|---|---|
| **n8n** | Workflow orchestration — self-hosted |
| **Ollama + llama3.2** | Local LLM — JD parsing (zero API cost) |
| **Anthropic Claude Sonnet** | Scoring + resume generation |
| **Google Drive** | Resume storage |
| **Gmail** | Match alert delivery |
| **Gotenberg** | Markdown → DOCX conversion |
| **Docker** | Container runtime for n8n |
| **Linux / systemd / UFW** | Host configuration and service management |

---

## Setup

### Prerequisites

- n8n (self-hosted via Docker)
- Ollama running locally with `llama3.2` pulled (`ollama pull llama3.2`)
- Anthropic API key (set up as `Anthropic API` credential in n8n)
- Google OAuth2 credential in n8n (for Drive + Gmail)
- Gotenberg running locally on port 3000 (optional — for DOCX output)

### Install

```bash
# Clone the repo
git clone https://github.com/itsolutions413544-boop/gcc-job-scraper-n8n.git

# Import workflows into n8n
# Settings → Import workflow → select the JSON files from /workflows/
```

### Configuration

Before running Path A, create a folder named **"Job Applications GCC"** in your Google Drive — the workflow saves resumes there.

To trigger Path B manually:

```bash
curl -X POST https://your-n8n-instance/webhook/manual-jd \
  -H "Content-Type: application/json" \
  -d '{
    "title": "IT Change Manager",
    "company": "Barclays India",
    "jd_text": "Full job description text here...",
    "url": "https://search.jobs.barclays/..."
  }'
```

---

## Cost estimate

| Operation | Model | Cost per run |
|---|---|---|
| JD parsing | Ollama llama3.2 (local) | $0.00 |
| Match scoring | Claude Sonnet | ~$0.002 per JD scored |
| Resume generation | Claude Sonnet | ~$0.015 per resume generated |

With 60 searches/day, 10–20 relevant listings filtered through, and a 70% score threshold reducing resume generation to 3–5 per day: **estimated daily API cost under $0.10**.

---

## Related

- [open-mind-lab-ai-research](https://github.com/itsolutions413544-boop/open-mind-lab-ai-research) — Automated YouTube content research agent using n8n + Ollama + PostgreSQL

---

*Built at Open Mind Lab — applied AI automation for IT operations.*
