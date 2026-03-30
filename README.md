# HubSpot GTM Engine with Multi-Channel Outreach
## Architectural Overview

---

## 👥 Who This Is For

- Sales teams running outbound at scale using HubSpot as their primary CRM
- Companies that need automated lead scoring, multi-channel routing, and progressive engagement tracking without manual SDR overhead
- Operators who want real-time pipeline visibility with dashboards that update as leads move through the funnel

---

## 🗺️ Architecture at a Glance

The engine is a 4-workflow system spanning Power Automate, HubSpot, and HeyReach. Power Automate handles ingestion and AI enrichment, pushing Apollo-sourced leads into HubSpot with OpenAI-generated ICP scores, reasoning, and personalized icebreakers. HubSpot orchestrates all downstream logic: intent-based routing across three tiers, lifecycle stage management, automated email outreach, and progressive engagement tracking. HeyReach handles LinkedIn outreach for high-fit leads, receiving contacts via webhook from HubSpot's routing workflow. A real-time dashboard surfaces pipeline distribution, engagement funnel metrics, and enrichment source analytics.

---

## 🔁 Example: One Lead's Journey

1. **Apollo/Clay Export:** A CSV of ICP-targeted executives is exported from Apollo or Clay with firmographic data (name, title, company, industry, employee count, technologies, revenue).
2. **Power Automate Ingestion:** The CSV lands in a OneDrive folder. Power Automate triggers, reads each row from the Excel table, and sends the lead's data to OpenAI GPT-4o-mini.
3. **AI Scoring & Personalization:** OpenAI returns a structured JSON object with three fields: a personalized 2-sentence icebreaker, an ICP score (1-10) based on company fit signals (Microsoft 365 usage, employee count, regulated industry, revenue), and a one-sentence reasoning statement. Power Automate parses the JSON and pushes the contact to HubSpot via REST API with custom properties: AI Icebreaker, ICP Score, ICP Reasoning, Enrichment Source, and Ingestion Timestamp.
4. **Intent-Based Routing:** A HubSpot workflow triggers when ICP Score is set. A three-way branch evaluates the score:
   - **Score ≥ 8 (High Fit):** Lifecycle Stage set to Sales Qualified Lead. Internal email notification fires to the sales rep with full lead context. A webhook pushes the contact to a HeyReach LinkedIn campaign. An automated high-fit outreach email is sent.
   - **Score ≥ 4 (Medium Fit):** Lifecycle Stage set to Marketing Qualified Lead. A nurture email sequence is triggered.
   - **Score < 4 (Low Fit):** Lifecycle Stage set to Other. No outreach.
5. **LinkedIn Outreach (High Fit):** HeyReach receives the lead via API and executes a LinkedIn sequence: View Profile (warm-up) → 3-hour delay → Connection Request with personalized icebreaker → If Accepted: follow-up message with meeting CTA → If Not Accepted: Follow Profile (soft engagement).
6. **Engagement Tracking:** Four HubSpot workflows progressively track engagement: Email Opened → Email Clicked → Email Replied → Meeting Booked. Each updates a custom Engagement Status property. Reply and Meeting Booked workflows additionally fire internal email notifications for immediate sales rep action. Meeting Booked also promotes the contact to Lifecycle Stage: Opportunity.
7. **Dashboard Visibility:** The GTM Engine Performance dashboard displays four real-time reports: Pipeline by Lifecycle Stage (bar chart), ICP Score by Lifecycle Stage (sortable table), Engagement Funnel (bar chart), and Leads by Enrichment Source (pie chart).

---

## Workflow 1: Lead Ingestion & AI Enrichment (Power Automate)

**Trigger:** New file created in OneDrive folder (/Documents/GTM Lead Drops).

<img src="./Assets/PowerAutomate_Ingestion_Flow.png" width="300" alt="Power Automate Ingestion Flow">

**Processing Pipeline:**
- **Excel Parsing:** "List rows present in a table" reads all rows from the uploaded Apollo export.
- **Loop (Apply to each):** For each row, the following executes sequentially:
  - **OpenAI Chat Completion (GPT-4o-mini):** Receives the lead's name, title, company, employee count, industry, technologies, annual revenue, and keywords. Returns a structured JSON object containing icebreaker, score (1-10), and reasoning. The prompt includes a strict scoring rubric: Microsoft 365 technology signals (+3), employee count thresholds (+1 to +2), regulated industry (+2), revenue over $50M (+1), Google Workspace presence (cap at 3).
  - **Compose:** Strips markdown code fences from the OpenAI response to produce clean JSON.
  - **Parse JSON:** Extracts icebreaker, score, and reasoning into discrete fields.
  - **HTTP POST:** Pushes the contact to HubSpot's CRM API (https://api.hubspot.com/crm/v3/objects/contacts) with mapped properties: firstname, lastname, email, company, jobtitle, ai_icebreaker, icp_score, icp_reasoning, enrichment_source, ingestion_timestamp.

**Authentication:** OpenAI via Bearer token. HubSpot via Private App access token.

**AI Cost:** GPT-4o-mini at < $0.001 per lead for combined scoring and personalization.

---

## Workflow 2: ICP-Based Routing & Multi-Channel Outreach (HubSpot)

**Trigger:** Contact property "ICP Score" is known.

<img src="./Assets/HubSpot_Routing_Workflow.png" width="100%" alt="HubSpot ICP Score Routing Workflow">

**Branch Logic (evaluated top-down):**

### Branch 1: High Fit (ICP Score ≥ 8)
1. **Edit Record:** Set Lifecycle Stage to "Sales Qualified Lead"
2. **Send Internal Email Notification:** Alerts sales rep with lead name, title, company, email, ICP Score, ICP Reasoning, and AI Icebreaker
3. **Send Webhook (POST):** Pushes contact to HeyReach campaign (https://api.heyreach.io/api/v1/campaigns/378065/leads) with firstName, lastName, email, company, and icebreaker as custom fields. Authenticated via HeyReach API key.
4. **Send Email:** "SQL - High Fit Outreach" — direct, personalized email using the AI Icebreaker token, focused on M365 governance pain points with a 15-minute call CTA.

### Branch 2: Medium Fit (ICP Score ≥ 4)
1. **Edit Record:** Set Lifecycle Stage to "Marketing Qualified Lead"
2. **Send Email:** "MQL - Nurture Outreach" — educational email about M365 governance challenges, softer CTA offering to share resources.

### Branch 3: Low Fit (Default)
1. **Edit Record:** Set Lifecycle Stage to "Other"
2. No outreach triggered.

---

## HeyReach LinkedIn Campaign: High-Fit ICP Outreach

**Campaign Structure (conditional multi-branch sequence):**

<img src="./Assets/HeyReach_Campaign_Sequence.png" width="100%" alt="HeyReach LinkedIn Campaign Sequence">

1. **View Profile** — Warm-up engagement before connection request
2. **3-Hour Delay** — Natural pacing
3. **Send Connection Request** — Personalized note using {FIRST_NAME} and {icebreaker} custom variable mapped from HubSpot webhook. Fallback message for missing variables.
4. **If Connection Condition:**
   - **Accepted → 1-Day Delay → Send Message:** Follow-up referencing {Company}, M365 governance, and 15-minute meeting CTA. If replied → End. If no reply → 1-Day Delay → No Reply Yet → End.
   - **Not Accepted → 5-Day Wait → Like Post** (soft engagement) **→ 5-Day Delay → End.**

**Schedule:** Monday-Friday, 09:00-19:00 ET. Auto-withdraw unanswered connection requests after 25 days.

---

## Workflow 3: Progressive Engagement Tracking (HubSpot — 4 Workflows)

Each workflow updates a custom dropdown property "Engagement Status" to track progressive lead engagement.

<img src="./Assets/HubSpot_Email_Opened.png" width="49%" alt="Email Opened Workflow"> <img src="./Assets/HubSpot_Email_Clicked.png" width="49%" alt="Email Clicked Workflow">
<img src="./Assets/HubSpot_Email_Replied.png" width="49%" alt="Email Replied Workflow"> <img src="./Assets/HubSpot_Meeting_Booked.png" width="49%" alt="Meeting Booked Workflow">

### 3a: Email Opened
- **Trigger:** Property "Marketing emails opened" value changed
- **Action:** Set Engagement Status to "Opened"

### 3b: Email Clicked
- **Trigger:** Property "Marketing emails clicked" value changed
- **Action:** Set Engagement Status to "Clicked"

### 3c: Email Replied
- **Trigger:** "Replied to marketing email" event completed
- **Action 1:** Set Engagement Status to "Replied"
- **Action 2:** Send internal email notification — "Reply Received" alert with lead details and ICP Score

### 3d: Meeting Booked
- **Trigger:** "Meeting booked" event completed
- **Action 1:** Set Engagement Status to "Goal Reached"
- **Action 2:** Set Lifecycle Stage to "Opportunity"
- **Action 3:** Send internal email notification — "Meeting Booked" alert with lead details and ICP Score

**Engagement Funnel Progression:** Delivered → Opened → Clicked → Replied → Goal Reached

---

## Workflow 4: GTM Engine Performance Dashboard (HubSpot)

Real-time dashboard with four reports, all sourced from Contact data:

<img src="./Assets/HubSpot_GTM_Dashboard.png" width="100%" alt="GTM Engine Performance Dashboard">

| Report | Type | Displaying | Measured By |
|--------|------|-----------|-------------|
| Pipeline by Lifecycle Stage | Vertical Bar Chart | Lifecycle Stage | Count of Contacts |
| ICP Score by Lifecycle Stage | Sortable Table | Create Date, First Name, Company, ICP Score, Lifecycle Stage, Engagement Status | — |
| Engagement Funnel | Vertical Bar Chart | Engagement Status | Count of Contacts |
| Leads by Enrichment Source | Pie Chart | Enrichment Source | Count of Contacts |

---

## Custom HubSpot Properties

| Property | Type | Purpose |
|----------|------|---------|
| ICP Score | Number | AI-generated company fit score (1-10) |
| ICP Reasoning | Multi-line text | One-sentence explanation of the score |
| AI Icebreaker | Single-line text | Personalized 2-sentence cold outreach opener |
| Enrichment Source | Single-line text | Origin of lead data (Apollo, Clay, LinkedIn, Manual) |
| Ingestion Timestamp | Single-line text | When the lead was processed by the ingestion pipeline |
| Engagement Status | Dropdown | Progressive engagement tracking (Delivered, Opened, Clicked, Replied, Goal Reached) |

---

## 🏗️ Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Lead Sourcing | Apollo.io, Clay | ICP-targeted executive contact data with firmographic and technographic enrichment |
| Ingestion & AI | Power Automate + OpenAI GPT-4o-mini | CSV parsing, AI scoring, icebreaker generation, JSON parsing, HubSpot API push |
| CRM & Orchestration | HubSpot (Sales + Marketing Hub) | Contact management, workflow routing, lifecycle stages, email sequences, engagement tracking, reporting dashboards |
| LinkedIn Outreach | HeyReach | Multi-step LinkedIn sequences with conditional logic and custom variable personalization |
| File Storage | OneDrive | Apollo CSV staging for Power Automate trigger |

---

## 🔧 System Topology

**7 Active Workflows:**

1. **Velocyt GTM - Ingestion Engine (Power Automate):** OneDrive trigger → Excel parsing → OpenAI scoring/icebreaker → JSON parsing → HubSpot API push
2. **ICP Score Routing (HubSpot):** ICP Score trigger → 3-way branch → lifecycle stages → internal notification → HeyReach webhook → automated emails
3. **Email Opened (HubSpot):** Marketing emails opened trigger → Engagement Status: Opened
4. **Email Clicked (HubSpot):** Marketing emails clicked trigger → Engagement Status: Clicked
5. **Email Replied (HubSpot):** Marketing email replied trigger → Engagement Status: Replied → internal notification
6. **Meeting Booked (HubSpot):** Meeting booked trigger → Engagement Status: Goal Reached → Lifecycle Stage: Opportunity → internal notification
7. **High-Fit ICP Outreach (HeyReach):** LinkedIn campaign — View Profile → Connection Request → Conditional follow-up

---

## 🚀 What I'd Improve in Production

- **Closed-Loop Scoring Optimization:** Add a weekly audit workflow (similar to the n8n Outbound Engine's GPT-4o rubric analysis) that aggregates conversion rates by ICP Score bucket and proposes rubric refinements based on actual outcomes.
- **Shadow Ledger:** Implement an immutable audit trail logging every workflow execution, API cost, and error across all systems — the same pattern proven in the n8n engine.
- **Database-First Vault Capture:** Add a pre-processing step that writes raw lead data to HubSpot before AI scoring begins, ensuring zero data loss even if OpenAI times out mid-batch.
- **Anti-Duplicate Logic:** Implement email-based deduplication in the ingestion pipeline to prevent re-processing leads already in HubSpot.
- **Multi-Source Enrichment:** Extend the ingestion pipeline to accept leads from Clay, LinkedIn, and web forms in addition to Apollo, with source-specific scoring adjustments.
- **Reply Sentiment Classification:** Add OpenAI-powered sentiment analysis on email replies (Positive, Negative, Meeting Request) to auto-triage inbound responses — already built and proven in the n8n Outbound Engine.

---

*Built by Rizwan Ahmed | Velocyt Consulting | velocyt.ca | github.com/RZ-Logic*
