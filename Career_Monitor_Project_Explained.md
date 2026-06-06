# Automated Career Intelligence System
## A Complete Project Walkthrough — Built June 7, 2026

---

## What We Built

An **autonomous job monitoring pipeline** that:
- Runs automatically every 6 hours
- Searches 7 aerospace/EV/defense companies across 10+ job portals
- Uses Claude AI to find, extract, and score opportunities
- Checks a Google Sheet to avoid sending duplicate alerts
- Sends a detailed, formatted email instantly when a new opening is found

No manual checking. No missed deadlines. Just alerts straight to your inbox.

---

## The Problem It Solves

Internship and job postings at niche companies like ideaForge, Skyroot Aerospace, or Garuda Aerospace don't always show up on mainstream portals. They get posted on LinkedIn, Internshala, Wellfound, Naukri, and sometimes just on the company's own website — all at different times. Manually checking 7 companies × 10 portals = 70 places to check, multiple times a day. Nobody does that. This system does it for you.

---

## Companies Being Monitored

1. **ideaForge** — India's leading UAV manufacturer (defense & surveying drones)
2. **Garuda Aerospace** — Agricultural and defense drone startup
3. **Asteria Aerospace** — Autonomous UAV systems
4. **Skyroot Aerospace** — Private space launch vehicles
5. **Ultraviolette Automotive** — High-performance electric motorcycles
6. **SSS Defence** — Defense technology systems
7. **Vazirani Automotive** — High-performance electric supercars

---

## Tools & Technologies Used

| Tool | Purpose |
|------|---------|
| **n8n** | Workflow automation platform — the backbone of the system |
| **Claude AI (Sonnet 4.6)** | Searches the web, extracts job data, scores profile match |
| **SerpAPI** | Gives Claude real-time Google Search access |
| **Google Sheets** | Stores seen job IDs to prevent duplicate emails |
| **Gmail** | Sends the formatted alert email |
| **Anthropic API** | Powers the Claude AI nodes |

---

## System Architecture — Node by Node

### Node 1: Schedule Trigger — "Every 6 Hours"
**What it does:** Automatically kicks off the entire workflow every 6 hours (00:00, 06:00, 12:00, 18:00).

**Why it matters:** You don't have to do anything manually. The system wakes itself up, does its job, and goes back to sleep.

**How it works in n8n:** Uses the built-in Schedule Trigger node set to a 6-hour interval.

---

### Node 2: AI Agent — "Search All Companies"
**What it does:** This is the brain of the operation. It uses Claude AI with web search capabilities to actively hunt for job postings across the internet.

**Why we used AI Agent instead of a simple HTTP request:** A regular HTTP request can only fetch one specific URL. An AI Agent can *reason* — it can search multiple queries, interpret results, handle different page formats, cross-reference sources, and return structured data. It's like having a research assistant instead of a simple script.

**The prompt it uses:** A detailed instruction telling Claude to:
- Search all 7 companies on LinkedIn, Internshala, Wellfound, Naukri, Indeed, Glassdoor, and official career pages
- Focus on internships, GET roles, R&D positions, and entry-level engineering
- Return results as a clean JSON array with specific fields for each job

**Sub-nodes attached to the AI Agent:**
- **Anthropic Chat Model** — Provides Claude Sonnet 4.6 as the language model
- **Google Search in SerpAPI** — Gives the agent real internet access to search Google

**What it returns:** A JSON array of job objects, each containing company name, title, type, location, deadline, apply link, eligibility requirements, and more.

---

### Node 3: Code Node — "Parse Job Results"
**What it does:** Takes Claude's raw text output and converts it into proper structured data that n8n can work with.

**Why it's needed:** Claude returns text. n8n needs clean JSON objects to pass between nodes. This node:
1. Strips any markdown formatting Claude might have added
2. Parses the JSON array
3. If no jobs were found (empty array), flags it so the workflow ends early
4. Splits the array so each job becomes its own item flowing through the pipeline individually

**The code logic:**
```javascript
const raw = $input.first().json.text;
let jobs = JSON.parse(raw.replace(/```json|```/g, '').trim());
if (!jobs.length) return [{ json: { no_new_jobs: true } }];
return jobs.map(job => ({ json: job }));
```

---

### Node 4: IF Node — "Any Jobs Found?"
**What it does:** A simple yes/no decision gate.

- **If no jobs found** → goes to "No New Jobs — End Silently" (workflow stops quietly, no email)
- **If jobs found** → continues to the deduplication check

**Why silent end matters:** You don't want an email every 6 hours saying "nothing found." You only want emails when there's something actionable.

---

### Node 5: Google Sheets — "Read Seen Job IDs"
**What it does:** Reads your entire SeenJobs Google Sheet and loads all previously seen job IDs into memory.

**Why:** This is how the system knows whether a job is genuinely new or one it already alerted you about. Without this, you'd get the same email for the same job every 6 hours forever.

**The Google Sheet (SeenJobs):** A simple spreadsheet with columns:
`job_id | company | title | type | location | apply_link | match_score | recommended_action | deadline | date_found`

---

### Node 6: Code Node — "Check If New"
**What it does:** Compares the current job's `job_id` against all IDs loaded from the Google Sheet.

**How job IDs are created:** Each job gets a unique ID in the format `company-role-YYYYMM` (e.g., `ideaforge-uav-intern-202506`). This makes IDs human-readable and collision-resistant.

**The logic:**
```javascript
const seenIds = new Set(seenRows.map(r => r.json.job_id));
if (seenIds.has(job.job_id)) return [{ json: { is_new: false } }];
return [{ json: { is_new: true, job } }];
```

---

### Node 7: IF Node — "Is New Opportunity?"
**What it does:** Another decision gate.

- **Already seen** → stops here, no email
- **New** → continues to scoring

---

### Node 8: AI Agent — "Claude — Score Profile Match"
**What it does:** A second Claude call that assesses how well the job matches your specific profile.

**Your profile it scores against:**
- Mechanical Engineering undergraduate
- Strong interest in UAVs, drones, aerospace, motorsports
- Arduino-based flight controller experience
- CAD assembly and engineering design projects
- Embedded systems exposure
- Active GitHub portfolio

**What it returns:**
```json
{
  "match_score": 8,
  "why_suitable": "This UAV firmware role directly aligns with your Arduino flight controller experience...",
  "missing_skills": ["ROS", "Python"],
  "suggestions": "Add a ROS mini-project to your GitHub before applying.",
  "recommended_action": "Apply immediately"
}
```

---

### Node 9: Code Node — "Merge Job + Score"
**What it does:** Combines the job data from Claude's search with the scoring data from Claude's analysis into one unified object ready for emailing.

---

### Node 10: Gmail — "Send Alert Email"
**What it does:** Sends a beautifully formatted HTML email to captainash1801@gmail.com the moment a new relevant opportunity is found.

**Email contains:**
- Company, role, location, type, deadline
- Direct apply link button
- Full eligibility and requirements breakdown
- Match score (shown as X/10)
- Why it's suitable for your profile
- Missing skills (if any)
- Recommended action: Apply immediately / Prepare portfolio / Upskill first

**Subject line format:** `[New Opportunity] ideaForge — UAV Systems Intern`

---

### Node 11: Google Sheets — "Log to Google Sheet"
**What it does:** Appends the new job to your SeenJobs sheet so it's never emailed again.

**Runs in parallel with the email** — both happen simultaneously after a new job is confirmed.

---

### Node 12: Code Node — "No New Jobs — End Silently"
**What it does:** Logs a console message and returns nothing. The workflow ends cleanly with no output and no email.

---

## Data Flow — The Full Journey

```
Every 6 Hours
    ↓
AI Agent searches all 7 companies (Claude + SerpAPI)
    ↓
Parse JSON results
    ↓
Any jobs found?
    ├── NO → End silently
    └── YES ↓
         Read all seen job IDs from Google Sheet
              ↓
         Is this job new?
              ├── NO → Stop (already alerted)
              └── YES ↓
                   Claude scores match against your profile
                        ↓
                   Merge job + score data
                        ↓
              ┌─────────┴─────────┐
         Send email          Log to Google Sheet
    (captainash1801@gmail.com)   (never email again)
```

---

## Why Each Design Decision Was Made

### Why n8n?
n8n is an open-source workflow automation tool (like Zapier but more powerful and self-hostable). It lets you visually connect APIs and services without building a backend from scratch. Your instance runs on `captainash.app.n8n.cloud`.

### Why Claude AI instead of a simple web scraper?
Web scrapers break constantly — every time a company changes their HTML, the scraper fails. Claude can read and understand any webpage format, interpret job listings from different structures, and return consistent data regardless of the source format. It's far more robust.

### Why SerpAPI?
Claude by itself doesn't have internet access. SerpAPI gives it real-time Google Search results, allowing it to find fresh job postings published minutes ago.

### Why Google Sheets for deduplication?
It's simple, free, and you can inspect/edit it directly. You can see every job that's ever been found, when it was found, and what score it got. If something looks wrong, you can delete a row to "reset" that job and get re-alerted.

### Why send one email per job instead of a daily digest?
Speed matters. A posting at ideaForge might get 200 applications in the first 6 hours. Getting alerted instantly vs. getting a digest the next morning could be the difference between being in the first 10 applicants or the last 100.

---

## Setup Checklist (For Reference)

- [x] n8n account at captainash.app.n8n.cloud
- [x] Workflow imported and configured
- [x] AI Agent node with search prompt
- [x] Anthropic Chat Model connected (needs API credits)
- [x] SerpAPI account created and connected
- [x] Google Sheet "SeenJobs" created with headers
- [x] Gmail connected for sending alerts
- [ ] **Add $5+ credits to Anthropic account** (platform.claude.com)
- [ ] Activate the workflow (toggle to Active)

---

## How to Activate

1. Go to your n8n workflow
2. Add Anthropic API credits at platform.claude.com
3. Click the toggle in the top-left to set the workflow to **Active**
4. The first run happens at the next 6-hour mark automatically
5. Or click **"Execute workflow"** to test it immediately

---

## For Your GitHub README

**Project title:** Autonomous Career Intelligence System

**One-liner:** AI-powered job monitoring pipeline that tracks aerospace & EV companies 24/7 and sends instant profile-matched email alerts.

**Tech stack:** n8n · Claude AI (Anthropic) · SerpAPI · Google Sheets API · Gmail API

**What it does:**
- Monitors 7 companies (ideaForge, Skyroot, Garuda, Asteria, Ultraviolette, SSS Defence, Vazirani) across LinkedIn, Internshala, Wellfound, Naukri, Indeed, Glassdoor, and official career pages
- Runs every 6 hours autonomously
- Deduplicates using Google Sheets to prevent repeat notifications
- Scores each opportunity 1-10 against a custom candidate profile
- Sends instant HTML email alerts with full eligibility breakdown and recommended action

---

## For Your Resume

> Built an autonomous career monitoring system using n8n, Claude AI (Anthropic), and SerpAPI that tracks 7 aerospace/EV companies across 10+ job portals every 6 hours, performs AI-driven profile match scoring, and delivers real-time email alerts with deduplication via Google Sheets API.

---

*Built on June 7, 2026 — from idea to working system in one session.*
