<p align="center"><img src="assets/logo.png" alt="FinWise" height="56"></p>

# FinWise

**A loan and financial-product marketplace.** Individuals and small businesses compare offers from partner lenders, apply once, and follow their application through to disbursal. Channel partners source customers and earn payouts, and staff run sales, partner management and service in the CRM.

[← All projects](../README.md) · 🔒 **Source code:** private repo [`loganathan-expertise/finwise`](https://github.com/loganathan-expertise/finwise) · [Request access](https://github.com/loganathan-expertise/project-portfolio/issues/new?template=access-request.yml)

| | |
|---|---|
| **Domain** | FinTech · lending marketplace · B2C retail loans and B2B (SME) finance |
| **Type** | Rebuild of a 2014–15 enterprise CRM implementation from its original specification, on current open-source software |
| **Status** | CRM + portal built and verified end to end with demo data |
| **Users** | Customers, channel partners (DSAs), lender credit desks, call-centre executives, field sales, regional managers, channel team, executives |

---

## What was rebuilt

The original system was a SugarCRM 7 implementation with a third-party website and CTI. Its design documents covered a 12-step customer journey, field-level module definitions, website touchpoints, channel-partner empanelment and the user hierarchy. These were enough for an end-to-end rebuild:

| Then (2014–15) | Now |
|---|---|
| SugarCRM 7 (commercial) | **SuiteCRM 8.10** (open source) |
| Third-party website | **Laravel 13 portal** for customers, partners and lenders, built here |
| Real bank integrations | **Lender credit desk** in the portal (5 fictitious lenders) standing in for FI integration |
| SMS gateway / SMTP | Notifications recorded in the CRM through a single delivery hook point, ready for a real gateway |
| 17 near-identical product modules | One application model with product-specific answers stored as structured JSON |

## Tech stack

| Layer | Technology |
|---|---|
| CRM | SuiteCRM 8.10 (Symfony + Angular shell, legacy module layer) on PHP 8.2 |
| Portal | Laravel 13 on PHP 8.5 · Blade · custom CSS |
| Database | MariaDB 12 (dedicated instance): `finwise_crm`, `finwise_portal` |
| Integration | SuiteCRM V8 REST API with custom routes · OAuth2 client credentials · HMAC-SHA256 signed webhooks |
| Web server | Apache 2.4 with FastCGI (separate PHP runtimes for CRM and portal) |
| Docs | Architecture / design / flow document, user manual, installation guide (Markdown + PDF), Mermaid diagrams |

## Architecture

```mermaid
flowchart TB
  subgraph U["Users"]
    direction LR
    C[Customer]
    P[Channel partner]
    L[Lender credit desk]
    S[Staff<br/>CCE · Sales · RM · Channel · Ops]
  end
  subgraph PT["FinWise Portal · Laravel 13 · PHP 8.5"]
    direction LR
    PW[Web UI<br/>Blade]
    PO[Offers engine<br/>EMI · FOIR]
    PS[CRM client<br/>OAuth2 token cache]
    PH[CRM webhook<br/>HMAC verified]
    PDB[(finwise_portal<br/>users · OTP · sessions)]
  end
  subgraph CR["FinWise CRM · SuiteCRM 8.10 · PHP 8.2"]
    direction LR
    API[V8 API custom routes<br/>/Api/V8/custom/fw/*]
    SVC[FWApi service]
    HK[Logic hooks<br/>IDs · rules · notifications]
    UI[SuiteCRM UI<br/>FinWise theme]
    MOD[FinWise modules<br/>+ extended standard modules]
    CDB[(finwise_crm)]
  end
  N[SMS / e-mail<br/>notification outbox]
  CTI[CTI - phase 2]
  C & P & L --> PW
  S --> UI
  PW --> PO
  PW --> PS
  PW --> PDB
  PH --> PDB
  PS -- "OAuth2 client credentials · JSON" --> API
  API --> SVC --> MOD --> CDB
  UI --> MOD
  HK --> MOD
  HK -- "partner-provision webhook" --> PH
  HK --> N
  CTI -.-> API
```

- **The CRM is the system of record.** Business rules run as logic hooks, so they apply equally to records created by staff, by the portal or by imports.
- **The portal holds only identities and short-lived data** (users, OTP codes, sessions). It reads and writes all business data through the CRM API, so there is one copy of every customer and application.
- **The CRM API enforces ownership.** An application is returned only to its own customer, partner or lender, and lender views mask the PAN.

## Customer journey

```mermaid
sequenceDiagram
  autonumber
  actor Cu as Customer
  participant Po as Portal (Laravel)
  participant Cr as CRM (SuiteCRM)
  actor Le as Lender desk
  Cu->>Po: Register (name, mobile, e-mail, pincode)
  Po->>Cu: OTP by SMS
  Cu->>Po: Enter OTP
  Po->>Cr: POST /fw/leads
  Cr-->>Po: Lead + Cust ID FWC000123
  Cu->>Po: Choose product, fill application
  Po->>Po: Offers engine (rate, EMI, fee, FOIR per lender)
  Cu->>Po: Pick an eligible offer
  Po->>Cr: POST /fw/applications (convert lead)
  Note over Cr: Lead → Customer [+ Business for SME]<br/>Loan application FW-APP-2026-00001<br/>co-applicants · guarantors · references
  Cr->>Cr: Stage → Submitted to lender · SMS + e-mail
  Le->>Po: Open credit desk queue
  Po->>Cr: GET /fw/lenders/{lender}/applications
  Le->>Po: Sanction or reject (with reason)
  Po->>Cr: POST …/decision
  alt Rejected
    Cu->>Po: Choose another lender
  else Sanctioned
    Cu->>Po: Upload documents / book a visit
    Po->>Cr: POST /fw/applications/{id}/documents
    Po->>Cr: POST /fw/appointments
    Note over Cr: Ops verifies documents → Documents verified
    Le->>Po: Mark disbursed
  end
```

### Application stages

```mermaid
stateDiagram-v2
  [*] --> Applied
  Applied --> SubmittedToLender: offer chosen
  SubmittedToLender --> InProcess: lender acknowledges
  SubmittedToLender --> Sanctioned
  InProcess --> Sanctioned: lender sanctions
  SubmittedToLender --> RejectedByLender
  InProcess --> RejectedByLender: lender rejects
  RejectedByLender --> SubmittedToLender: customer picks another lender
  Sanctioned --> DocumentsSubmitted: documents uploaded
  DocumentsSubmitted --> DocumentsVerified: ops verifies in CRM
  DocumentsVerified --> Disbursed: lender disburses
  Disbursed --> [*]
```

### Channel-partner empanelment (auto-approval)

```mermaid
flowchart TD
  A[Partner applies on the portal] --> B[CRM: Channel Partner<br/>status Submitted · code FWP00001]
  B --> T{Partner type}
  T -- Connector --> OK[Approved]
  T -- Associate --> R{Rules in master data:<br/>office area · staff · fee · payout %}
  R -- all pass --> OK
  R -- any fail --> RJ[Escalated to channel team]
  RJ --> RA{Reassess}
  RA -- yes --> OK
  RA -- no --> X[Rejected]
  OK --> AG[Agreement and commercials]
  AG --> E[Empanelled]
  E --> W[CRM → portal webhook<br/>HMAC signed]
  W --> PL[Partner login created<br/>credentials by SMS + e-mail]
```

### Appointment allocation

```mermaid
flowchart TD
  A[Visit requested for date D] --> P[Customer pincode]
  P --> S{Field-sales users<br/>serving the pincode?}
  S -- yes --> SL[Fewest appointments on D<br/>tie → round robin]
  S -- no --> C{Active partners<br/>serving the pincode?}
  C -- yes --> CL[Fewest appointments on D<br/>tie → round robin]
  C -- no --> E[No coverage → call centre follows up]
  SL --> N[Meeting created · SMS + e-mail to both sides]
  CL --> N
```

## Features

| Area | What's included |
|---|---|
| **Customer portal** | OTP registration, offers engine with EMI / FOIR across lenders, application wizard (retail and SME), tracker with stage history, document upload, appointments, service requests |
| **Partner portal** | Empanelment application, dashboard (leads, applications, appointments, earnings), partner lead entry |
| **Lender credit desk** | Queue, application detail with PAN masked, decisions (in process / sanction / reject with reason / disburse after document verification) |
| **CRM** | Leads, Customers, Businesses, Loan Applications, Appointments, Service Requests, Documents, Knowledge Base; 8 custom modules (Channel Partners, Branches, Co-Applicants, Guarantors, References, Shareholders, Empanelment Rules, Notifications) |
| **Business rules** | Auto-numbering (Cust ID, application no., partner code), stage notifications to customer and partner, stage history, empanelment checks, partner portal provisioning |
| **Catalogue** | 17 products in two segments (retail and SME), sub-products, managed from CRM dropdowns and read by the portal over the API |
| **Branding** | FinWise CRM theme; renamed modules carried through menus, quick-create, labels and "related to" lists |

## Data model

```mermaid
erDiagram
  LEADS ||--o| CONTACTS : "converts to"
  CONTACTS ||--o{ OPPORTUNITIES : "loan applications"
  ACCOUNTS ||--o{ CONTACTS : "business of"
  ACCOUNTS ||--o{ FW_SHAREHOLDERS : has
  OPPORTUNITIES ||--o{ FW_COAPPLICANTS : has
  OPPORTUNITIES ||--o{ FW_GUARANTORS : has
  OPPORTUNITIES ||--o{ FW_REFERENCES : has
  OPPORTUNITIES ||--o{ NOTES : documents
  OPPORTUNITIES ||--o{ MEETINGS : appointments
  CONTACTS ||--o{ CASES : "service requests"
  FW_CHANNELPARTNERS ||--o{ LEADS : sources
  FW_CHANNELPARTNERS ||--o{ OPPORTUNITIES : sources
  FW_BRANCHES ||--o{ FW_CHANNELPARTNERS : manages
  FW_MASTERDATA ||..|| FW_CHANNELPARTNERS : "empanelment rules"
  FW_NOTIFICATIONS }o..|| OPPORTUNITIES : "logs messages for"
```

## Security

- **Portal authentication:** bcrypt passwords, login throttling, and a 6-digit mobile OTP (stored hashed, 10-minute validity, limited attempts and resends). Partners must change their password at first sign-in.
- **Authorisation:** role middleware (customer / partner / lender) on every route; record-ownership checks in the CRM API.
- **Integration:** OAuth2 client credentials (portal → CRM), HMAC-SHA256 signed webhooks (CRM → portal), CSRF on all forms.
- **Input handling:** server-side validation of PAN, GSTIN, IFSC, pincode and mobile numbers; MIME sniffing on uploads. Only `public/` is web-visible.

## Screens

<p>
<img src="assets/portal-customer-dashboard.jpg" alt="Customer dashboard" width="49%">
<img src="assets/portal-application.jpg" alt="Application" width="49%">
</p>
<p>
<img src="assets/portal-partner-dashboard.jpg" alt="Partner dashboard" width="49%">
<img src="assets/portal-lender-desk.jpg" alt="Lender credit desk" width="49%">
</p>
<p><img src="assets/crm-loan-applications.jpg" alt="CRM loan applications" width="98%"></p>

## Next phase

CTI integration, real SMS gateway and SMTP, real lender APIs, partner sub-users per branch, a report pack.

---

[← All projects](../README.md) · 🔒 [Request code access](https://github.com/loganathan-expertise/project-portfolio/issues/new?template=access-request.yml)
