<p align="center"><img src="assets/logo.svg" alt="CamCon" height="64"></p>

# CamCon — Campus Connect

**One platform where colleges teach, companies post real work and hire, and students go from learning to a job without leaving it.**

[← All projects](../README.md) · 🔒 **Source code:** private repo [`loganathan-expertise/camcon`](https://github.com/loganathan-expertise/camcon) · [Request access](https://github.com/loganathan-expertise/project-portfolio/issues/new?template=access-request.yml)

| | |
|---|---|
| **Domain** | EdTech · campus recruitment · corporate induction |
| **Type** | Greenfield platform built on Moodle LMS |
| **Status** | Release 0.5: core flow end to end, verified for every role |
| **Users** | Students, faculty, placement officers (TPO), recruiters, hiring managers, mentors, platform admin |

---

## The problem

Colleges, students and recruiters work in separate systems. Colleges run an LMS, companies run an ATS, and placements move through spreadsheets and email. CamCon puts learning, real project work, hiring and onboarding in one place:

- **Colleges** publish courses and learning paths, approve which openings their students see, and track placement readiness.
- **Companies** post projects, internships and jobs, run interviews, make offers and onboard new hires with an induction programme.
- **Students** learn, build a verified profile and resume, take part in challenges, apply, interview, sign offers electronically and complete onboarding.

## Tech stack

| Layer | Technology |
|---|---|
| Platform | Moodle LMS 5.3 (LTS) |
| Language | PHP 8.4 / 8.5 |
| Custom code | 1 Boost child theme + 7 local plugins (Moodle public APIs only, **no core changes**) |
| Database | MariaDB (PostgreSQL supported) |
| Mobile | Moodle app with custom CamCon pages (Ionic templates via `db/mobile.php`) |
| Integrations | Jitsi video rooms, WhatsApp (Meta Cloud API, pluggable), HR exports (CSV / Excel / JSON), signed webhooks, REST web services, optional AI match explanations |
| Ops | Moodle cron (every minute), scheduled tasks, scripted install, upgrade and demo-data setup |

## Architecture

```mermaid
flowchart TB
  U[Students · faculty · placement officers · recruiters · mentors · platform admin<br/>web browser and Moodle app]
  subgraph Site[CamCon site · Moodle 5.3 LTS · PHP 8.4/8.5]
    T[theme_camcon<br/>branding, home page, organisation colours]
    O[local_camconorg<br/>organisations, approval, members, renewals, partnerships]
    P[local_camconplacement<br/>openings, applications, interviews, offers, onboarding]
    TL[local_camcontalent<br/>profiles, readiness, talent search]
    EN[local_camconengage<br/>challenges, learning paths, content library, WhatsApp]
    HD[local_camconhelp<br/>help desk, help articles]
    IN[local_camconinsights<br/>dashboards and exports]
    MB[local_camconmobile<br/>Moodle app pages]
    C[Moodle core<br/>courses, Book, Quiz, Assignment, SCORM, Forum, rubrics, completion, cohorts, roles]
  end
  DB[(MariaDB / PostgreSQL)]
  FS[(moodledata files)]
  CR[cron every minute]
  U --> Site
  T --- C
  O --> C
  P --> C
  P --> O
  TL --> O
  TL --> P
  EN --> P
  EN --> O
  HD --> O
  IN --> P
  IN --> TL
  MB --> P
  MB --> HD
  Site --> DB
  Site --> FS
  CR --> Site
```

| Component | Responsibility |
|---|---|
| `theme_camcon` | Design tokens, navigation, home page (banner slider, numbers, audiences, open projects), organisation colours inside organisation spaces |
| `local_camconorg` | Organisation applications and approval, provisioning, branding, members and verification, bulk upload, subscriptions and renewals, partnerships (MoU), offboarding with data export |
| `local_camconplacement` | Openings and eligibility, college approval per opening, applications, interview rounds, slots, panels and scorecards, offers with hiring-manager approval and e-signature, onboarding, HR integration, project-course builder |
| `local_camcontalent` | Career profile, talent-pool opt-in, six-part readiness score, PDF resume, talent search |
| `local_camconengage` | Challenges and hackathons, verifiable certificates, learning paths, content library, WhatsApp alerts |
| `local_camconhelp` | Help desk with routing, SLAs, escalation and ratings; help articles per organisation |
| `local_camconinsights` | Dashboards for the platform, each college and each company; CSV / Excel export |
| `local_camconmobile` | CamCon screens in the Moodle app (My CamCon, openings, challenges, help) |

### Tenancy model: soft tenancy

A marketplace needs companies to see students from every college, so CamCon uses **one site with organisation-scoped spaces** instead of hard isolation:

```mermaid
flowchart LR
  A[Organisation approved] --> B[Course category<br/>camcon-org-id]
  A --> C[Cohort<br/>camcon-org-id]
  A --> D[Organisation manager role<br/>category level]
  B --> E[College courses ·<br/>company project courses ·<br/>induction]
  C --> F[Enrolment and reporting]
```

Visibility rules are enforced in code:
- A student sees an opening only after their college approves it.
- A profile is visible to recruiters only if the student opted in to the talent pool or applied to that company.
- Applying requires explicit consent to share the profile.

## End-to-end flow

```mermaid
flowchart TB
  A[College or company applies] --> B{Platform admin}
  B -- approve --> C[Space provisioned:<br/>category, cohort, admin]
  B -- reject --> X[Reason recorded]
  C --> D[College adds faculty and students;<br/>publishes courses]
  C --> E[Company posts an opening]
  E --> F{Each college's<br/>placement officer}
  F -- approve --> G[Verified students see it and apply]
  G --> H0[Shortlist]
  H0 --> IV[Interview rounds:<br/>book slot, scorecard]
  IV -- recommended --> H[Select]
  H -- project --> I[Project course: milestones,<br/>rubric grading by mentors]
  H --> J[Offer]
  I --> J
  J --> HM{Hiring manager<br/>if the company has one}
  HM -- send back --> J
  HM -- approve --> J2[Student sees the offer]
  J2 -- accept and e-sign --> K[Employee: induction course,<br/>checklist, buddy, 30/60/90 check-ins,<br/>sign-off]
  J2 -- decline --> L[Closed]
```

### Offer e-signature

```mermaid
sequenceDiagram
  autonumber
  actor S as Student
  participant C as CamCon
  S->>C: Accept offer
  C->>S: 6-digit code (email + notification, valid 10 min, 5 tries)
  S->>C: Code + typed full name + drawn signature + consent
  C->>C: Store code hash only · record name, signature, IP, browser
  C->>C: SHA-256 fingerprint of offer terms
  C-->>S: Signed offer letter PDF with audit trail
  Note over C: PDF warns if terms no longer match the fingerprint
```

## Features

| Area | What's included |
|---|---|
| **Organisations** | Application and approval, provisioning, branding, member verification, bulk upload; yearly subscriptions with reminders (30/7/0 days), grace period and read-only mode when lapsed; offboarding with ZIP export; college–company partnerships |
| **Learning** | Books, pages, assignments with rubrics, quizzes (11 question types), SCORM, forums, completion; learning paths with verifiable certificates; one-click course templates |
| **Openings and challenges** | Projects, internships and jobs with eligibility (branch, year, CGPA, one-offer policy); challenges and hackathons with teams, judges and a leaderboard |
| **Interviews** | Rounds, interviewer slots, self-booking with calendar events, Jitsi video rooms, panels with one scorecard per member, company question banks |
| **Hiring** | Shortlist, select, offer, hiring-manager approval, built-in e-signature, offer letter PDF; HR exports, signed webhook on hire, REST API |
| **Projects** | An auto-built project course per opening: brief, forum, three milestones, rubric (code 25 · functionality 35 · docs 20 · demo 20) |
| **Onboarding** | Induction course on acceptance, joining checklist, buddy, 30/60/90-day check-ins, hiring-manager sign-off |
| **Readiness** | Profile, PDF resume, aptitude tests, mock interviews, six-part readiness score, rule-based matching with reasons, optional AI explanation |
| **Support** | Help desk routed to the college, the company or the CamCon team; priorities and SLAs (urgent 4 h / 24 h … low 48 h / 240 h); escalation; help articles |
| **Insights** | Platform, college and company dashboards (funnel, time to hire, placements by branch, readiness); period filters; CSV / Excel |
| **Mobile** | Moodle app pages for My CamCon, openings, challenges and help, with CamCon styling |

## Data model (core placement tables)

```mermaid
erDiagram
  local_camconorg ||--o{ local_camconorg_member : has
  local_camconorg ||--o{ local_camconplacement_open : posts
  local_camconplacement_open ||--o{ local_camconplacement_appr : "approved by colleges"
  local_camconplacement_open ||--o{ local_camconplacement_app : receives
  local_camconplacement_app ||--o| local_camconplacement_offer : leads_to
  local_camconplacement_open ||--o{ local_camconplacement_round : "interview rounds"
  local_camconplacement_round ||--o{ local_camconplacement_slot : "time slots"
  local_camconplacement_slot ||--o| local_camconplacement_score : scorecard
  local_camconplacement_offer ||--o| local_camconplacement_onb : onboarding
  user ||--o{ local_camconorg_member : is
  user ||--o| local_camcontalent_profile : has
```

## Engineering highlights

- **Zero core modifications.** Everything is standard Moodle plugins and public APIs (course and activity builder, question engine, completion, charts, mobile delegates), so upgrades stay routine.
- **Released in increments:** 0.1 → 0.1.1 → 0.2 → 0.3 → 0.4 → 0.5. Each release was verified in the browser for every role, using seeded demo organisations, students and recruiters.
- **Repeatable setup.** Numbered setup scripts handle install, configuration, demo data and publishing. A demo-data seed produces realistic states, such as backdated applications and pending interviews.
- **Privacy built in:** Moodle Privacy API, explicit consent to share profiles, scoped visibility.

## Roadmap

Coding labs (VPL / CodeRunner on Docker), Git repositories and a PR review board, billing and licensing (Razorpay), and production hardening (HTTPS, SMTP, backups).

---

[← All projects](../README.md) · 🔒 [Request code access](https://github.com/loganathan-expertise/project-portfolio/issues/new?template=access-request.yml)
