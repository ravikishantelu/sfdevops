# Solution Design: AI-Powered Loan Application Assessment Assistant

**Version:** 1.0  
**Date:** 2026-05-24  
**Author:** Ravi K Telu  
**Contact:** ravikishantelu@gmail.com  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Problem](#2-business-problem)
3. [POC Vision](#3-poc-vision)
4. [Architecture Overview](#4-architecture-overview)
5. [Salesforce Data Model](#5-salesforce-data-model)
6. [Data Cloud Layer](#6-data-cloud-layer)
7. [Einstein Document AI](#7-einstein-document-ai)
8. [Agentforce Underwriting Assistant](#8-agentforce-underwriting-assistant)
9. [Salesforce Flow Automation](#9-salesforce-flow-automation)
10. [Lightning Web Components](#10-lightning-web-components)
11. [Integration Architecture](#11-integration-architecture)
12. [Security & Sharing Model](#12-security--sharing-model)
13. [POC Delivery Phases](#13-poc-delivery-phases)
14. [Demo Scenario](#14-demo-scenario)
15. [Key Success Metrics](#15-key-success-metrics)

---

## 1. Executive Summary

This solution leverages **Agentforce**, **Einstein Document AI**, **Salesforce Data Cloud**, **Flow Automation**, and **Lightning Web Components** to automate the end-to-end loan document assessment lifecycle — from broker document submission through to underwriter recommendation.

Data Cloud enriches the solution with a **360° unified borrower profile**, historical loan behaviour, predictive AI scoring, and real-time calculated insights — transforming the POC from a document processing tool into a full borrower intelligence platform.

---

## 2. Business Problem

Underwriters and credit analysts currently spend significant time:

- Reading large document packs manually
- Extracting borrower information
- Writing underwriting summaries
- Identifying missing documents
- Checking policy exceptions
- Following up brokers repeatedly

This slows approval cycles and creates inconsistency across assessors.

---

## 3. POC Vision

> **"Upload documents → AI extracts, analyses, summarises, and recommends next actions automatically."**

With Data Cloud added:

> **"AI knows this borrower's entire financial history, detects that their income has grown 23% since last year, flags that they had a prior decline which is now resolved, and recommends approval with confidence — all in under 60 seconds."**

---

## 4. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          SOLUTION ARCHITECTURE                               │
│                                                                              │
│  ┌────────────────┐     ┌───────────────────┐     ┌────────────────────┐   │
│  │  BROKER        │     │  SALESFORCE        │     │  UNDERWRITER       │   │
│  │  PORTAL        │────▶│  CORE PLATFORM    │────▶│  WORKSPACE         │   │
│  │  (LWC /        │     │                   │     │  (LWC Console)     │   │
│  │  Exp. Cloud)   │     │                   │     │                    │   │
│  └────────────────┘     └─────────┬─────────┘     └────────────────────┘   │
│                                   │                                          │
│           ┌───────────────────────┼───────────────────────┐                 │
│           ▼                       ▼                       ▼                 │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────────────┐     │
│  │ Einstein Doc AI │   │   Agentforce     │   │  Salesforce Flow     │     │
│  │ (Classification │   │   AI Agent       │   │  Automation          │     │
│  │  + Extraction)  │   │   (Underwriting  │   │  (Tasks / Routing /  │     │
│  │                 │   │   Assistant)     │   │   Notifications)     │     │
│  └─────────────────┘   └────────┬─────────┘   └──────────────────────┘     │
│                                  │                                           │
│  ┌───────────────────────────────▼──────────────────────────────────────┐   │
│  │                     DATA CLOUD LAYER                                  │   │
│  │                                                                       │   │
│  │  ┌────────────┐ ┌─────────────┐ ┌──────────────┐ ┌───────────────┐  │   │
│  │  │ Unified    │ │ Calculated  │ │ Predictive   │ │  Activation   │  │   │
│  │  │ Borrower   │ │ Insights    │ │ AI Scoring   │ │  → CRM        │  │   │
│  │  │ Profile    │ │ (DTI, etc.) │ │ (Risk/Fraud) │ │               │  │   │
│  │  └────────────┘ └─────────────┘ └──────────────┘ └───────────────┘  │   │
│  └───────────────────────────────────────────────────────────────────── ┘   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Salesforce Data Model

### 5.1 `Loan_Application__c` — Master Record

| Field | Type | Description |
|-------|------|-------------|
| `Application_Number__c` | Auto Number | Unique ID (LA-{0000}) |
| `Applicant_Name__c` | Text(255) | Primary borrower |
| `Loan_Amount__c` | Currency | Requested loan amount |
| `Loan_Purpose__c` | Picklist | Home, Investment, Refinance, Personal |
| `Application_Status__c` | Picklist | New → Documents Received → AI Processing → Under Review → Conditional Approval → Approved / Declined |
| `Risk_Rating__c` | Picklist | Low, Medium, High, Critical |
| `Assigned_Underwriter__c` | Lookup(User) | Responsible assessor |
| `Broker__c` | Lookup(Account) | Submitting broker |
| `AI_Processing_Status__c` | Picklist | Pending, In Progress, Completed, Failed |
| `AI_Summary__c` | Long Text | Agentforce-generated borrower summary |
| `AI_Risk_Alerts__c` | Long Text | Detected risk indicators |
| `AI_Recommended_Action__c` | Text Area | Next action recommendation |
| `Documents_Complete__c` | Checkbox | All required docs received |
| `Missing_Documents__c` | Long Text | AI-detected missing items |
| `Submission_Date__c` | DateTime | When broker submitted |
| `SLA_Due_Date__c` | DateTime | Formula: Submission + 2 business days |
| `Predicted_Default_Risk__c` | Percent | Einstein AI — default probability |
| `Fraud_Risk_Score__c` | Percent | Data Cloud fraud model score |
| `Estimated_Borrowing_Capacity__c` | Currency | Einstein serviceability model |

### 5.2 `Loan_Document__c` — Document Tracking

| Field | Type | Description |
|-------|------|-------------|
| `Loan_Application__c` | Master-Detail | Parent loan application |
| `Document_Type__c` | Picklist | Payslip, Bank Statement, Tax Return, ID Document, BAS, Loan Form, Other |
| `Document_Status__c` | Picklist | Pending, Received, Verified, Rejected |
| `ContentDocument_Id__c` | Text | Salesforce File ID |
| `AI_Extracted_Data__c` | Long Text | JSON payload from Document AI |
| `AI_Confidence_Score__c` | Percent | Extraction confidence (0–100%) |
| `Document_Date__c` | Date | Document date (extracted) |
| `Anomaly_Detected__c` | Checkbox | AI flagged anomaly |
| `Anomaly_Description__c` | Text Area | Description of anomaly |

### 5.3 `Extracted_Financial_Data__c` — Structured AI Output

| Field | Type | Description |
|-------|------|-------------|
| `Loan_Application__c` | Lookup | Parent application |
| `Document__c` | Lookup(Loan_Document__c) | Source document |
| `Employer_Name__c` | Text | From payslip |
| `Annual_Income__c` | Currency | Declared / extracted income |
| `PAYG_Status__c` | Checkbox | PAYG confirmed |
| `Monthly_Expenses__c` | Currency | From bank statements |
| `Monthly_Liabilities__c` | Currency | Loan / debt repayments |
| `Gambling_Indicators__c` | Checkbox | Flagged transactions |
| `Applicant_DOB__c` | Date | From ID document |
| `Address_Match__c` | Checkbox | ID vs application address match |
| `Tax_Declared_Income__c` | Currency | From tax return |
| `Employment_Duration_Years__c` | Number | Tenure extracted |

### 5.4 `Underwriting_Note__c` — Audit Trail

| Field | Type | Description |
|-------|------|-------------|
| `Loan_Application__c` | Master-Detail | Parent |
| `Note_Type__c` | Picklist | AI Generated, Manual, System |
| `Note_Content__c` | Long Text | Full note text |
| `Author__c` | Lookup(User) | Human or AI |
| `Is_AI_Generated__c` | Checkbox | True for Agentforce notes |

---

## 6. Data Cloud Layer

### 6.1 Why Data Cloud?

Without Data Cloud, the AI only knows what's in the uploaded documents for the current application. With Data Cloud, Agentforce can answer:

- *"Has this borrower applied before and what was the outcome?"*
- *"Is their declared income consistent with prior applications?"*
- *"What does their 24-month spending pattern look like?"*
- *"Are there other open applications from the same applicant?"*

### 6.2 Data Streams (Ingestion Sources)

| Data Stream | Source | Frequency | Key Fields |
|-------------|--------|-----------|------------|
| CRM Contacts | Salesforce CRM | Real-time | Name, DOB, Email, Phone, Address |
| Loan Application History | `Loan_Application__c` | Real-time | Amount, Status, Risk Rating, Outcome |
| Extracted Financial Data | `Extracted_Financial_Data__c` | Real-time | Income, Expenses, Employer, Liabilities |
| Bank Transactions | External bank feed / Plaid API | Batch (daily) | Transactions, categories, merchant names |
| Credit Bureau | Equifax / Experian API | On-demand | Credit score, defaults, enquiries |
| Broker Submissions | Experience Cloud events | Real-time | Submission timestamps, document types |
| Web / App Behaviour | Experience Cloud tracking | Real-time | Pages visited, time spent, doc views |

### 6.3 Identity Resolution Rules

```
Match Rule Priority:
1. Email (exact match)            → High confidence merge
2. Full Name + DOB                → Medium confidence merge
3. Full Name + Address            → Medium confidence merge
4. Phone Number                   → Supporting match
5. Driver Licence Number          → High confidence (from Document AI)
```

This ensures one applicant who has applied multiple times across different brokers is recognised as the **same person**, and their entire history is surfaced to Agentforce instantly.

### 6.4 Unified Data Model Objects (DMOs)

| DMO | Type | Purpose |
|-----|------|---------|
| `Individual` | Standard | Golden borrower profile |
| `Contact_Point_Email` | Standard | All email addresses |
| `Contact_Point_Phone` | Standard | All phone numbers |
| `Contact_Point_Address` | Standard | All addresses (ID, application, bureau) |
| `Loan_Application_DMO` | Custom | All loan applications unified |
| `Financial_Profile_DMO` | Custom | Aggregated income, expenses, liabilities |
| `Bank_Transaction_DMO` | Custom | All transaction-level data |
| `Credit_Enquiry_DMO` | Custom | Bureau enquiries and scores |
| `Document_Submission_DMO` | Custom | Document submission history |

### 6.5 Calculated Insights

| Calculated Insight | Formula | Business Value |
|-------------------|---------|----------------|
| `Avg_Monthly_Income_6M` | Avg of last 6 months bank credits | More reliable than a single payslip |
| `Avg_Monthly_Expenses_6M` | Avg of last 6 months debits | True living expense baseline |
| `Debt_To_Income_Ratio` | Total liabilities / Annual income × 100 | Core serviceability metric |
| `Income_Consistency_Score` | StdDev of monthly income (lower = stable) | Employment stability indicator |
| `Gambling_Transaction_Pct` | Gambling spend / Total spend × 100 | Risk policy check |
| `Prior_Application_Count` | Count of all prior loan apps | Repeat applicant indicator |
| `Prior_Default_Flag` | Any prior declined / defaulted loans | Critical risk flag |
| `Address_Consistency_Score` | Match % across ID, apps, bureau | Fraud signal |
| `Credit_Score_Trend` | Change in credit score over 12 months | Trajectory indicator |
| `Broker_Submission_Volume` | Apps submitted by same broker this month | Broker behaviour insight |

### 6.6 Predictive AI Models (Einstein on Data Cloud)

#### Model 1: `Loan_Default_Probability`
- **Type:** Classification (0–100% default probability)
- **Training Data:** Historical approved loans + outcomes (12+ months)
- **Features:** DTI ratio, income consistency, credit score, employment tenure, loan purpose, prior applications
- **Output:** Populates `Predicted_Default_Risk__c` on Loan Application

#### Model 2: `Document_Fraud_Score`
- **Type:** Anomaly detection
- **Signals:** Address mismatches, income vs spending gap, multiple applications same identity, document metadata inconsistencies
- **Output:** `Fraud_Risk_Score__c` (0–100)

#### Model 3: `Serviceability_Score`
- **Type:** Regression
- **Calculates:** Predicted sustainable repayment capacity
- **Output:** `Estimated_Borrowing_Capacity__c`

### 6.7 Data Activation (Insights → CRM)

| Activated Insight / Segment | Destination | How Used |
|-----------------------------|-------------|----------|
| `Predicted_Default_Risk` | `Loan_Application__c` field | Displayed on underwriter console |
| `Unified_Income_Verified` | `Extracted_Financial_Data__c` | Replaces single-payslip income |
| `Prior_Decline_Flag` | `Loan_Application__c` | Triggers senior underwriter routing |
| `High_Fraud_Risk` segment | Flow trigger | Auto-escalates to fraud team |
| `Low_Risk_Pre_Approved` segment | Flow trigger | Fast-tracks assessment |
| `VIP_Borrower` segment | Console alert | Flags existing premium customers |

---

## 7. Einstein Document AI

### 7.1 Document AI Models (Per Document Type)

| Model | Extracts | Confidence Threshold |
|-------|----------|----------------------|
| Payslip Model | Employer, Gross Income, Net Pay, Pay Period, PAYG status | 85% |
| Bank Statement Model | Account holder, Balance, Credits, Debits, Expense categories, Gambling flags | 80% |
| ID Document Model | Full name, DOB, Address, Expiry date | 90% |
| Tax Return Model | Declared income, ABN, Tax year, Lodgement date | 85% |
| BAS Model | GST collected, GST credits, Period | 80% |

### 7.2 Document Classification Flow

```
File uploaded to ContentVersion
        │
        ▼
Einstein Document AI → Classify document type
        │
        ├── Known type  → Route to correct extraction model
        └── Unknown     → Flag for manual classification
        │
        ▼
Field Extraction → Store in Loan_Document__c.AI_Extracted_Data__c
        │
        ▼
Confidence < threshold? → Flag for human review
        │
        ▼
Write structured data → Extracted_Financial_Data__c
        │
        ▼
Data Cloud ingests → Updates Unified Borrower Profile
```

---

## 8. Agentforce Underwriting Assistant

### 8.1 Agent Configuration

| Setting | Value |
|---------|-------|
| Agent Name | Loan Underwriting Assistant |
| Agent Type | Internal (Underwriter-facing) |
| Channel | Salesforce Console (Service Cloud) |
| Grounded With | Data Cloud Unified Borrower Profile |

### 8.2 Agent Topics & Instructions

#### Topic 1: Unified Borrower Profile Retrieval (Data Cloud)
```
Instructions:
- Query Data Cloud for unified Individual matching applicant name + DOB
- Retrieve all Calculated Insights (DTI, income avg, gambling %, etc.)
- Retrieve prior application history and outcomes
- Retrieve credit score trend
- Surface any Prior_Default_Flag or Fraud_Risk_Score
- Use this context to enrich ALL subsequent topic responses
```

#### Topic 2: Borrower Summary Generation
```
Instructions:
- Retrieve all Extracted_Financial_Data__c for the Loan Application
- Combine with Data Cloud Unified Profile (6-month income avg vs payslip)
- Summarise employer, income, employment tenure, and PAYG status
- Reference prior application history if applicable
- Generate a 3-5 sentence professional borrower summary
- Store output in Loan_Application__c.AI_Summary__c
```

#### Topic 3: Risk Assessment
```
Instructions:
- Analyse Calculated Insights: DTI ratio, gambling %, expense ratio
- Apply lender policy thresholds (expense ratio > 70% = High Risk)
- Cross-reference declared income vs Data Cloud 6-month bank average
- Check address consistency using Address_Consistency_Score
- Incorporate Einstein Predicted_Default_Risk score
- Flag Prior_Default_Flag from Data Cloud history
- Generate Risk_Rating__c (Low / Medium / High / Critical)
- List all risk alerts in AI_Risk_Alerts__c
```

#### Topic 4: Document Completeness Check
```
Instructions:
- Compare received document types against required checklist:
  - Employed:       Payslip x2, Bank Statements x3 months, Tax Return, ID, Loan Form
  - Self-Employed:  All above + BAS x2, Financial Statements
- List all missing documents in Missing_Documents__c
- Set Documents_Complete__c = true only when all required docs received
```

#### Topic 5: Recommended Action
```
Instructions:
- Based on risk rating, completeness, and Data Cloud profile:
  - ALL DOCS + Low Risk          → "Proceed to formal assessment"
  - ALL DOCS + Medium Risk       → "Proceed with conditions: [list]"
  - ALL DOCS + High Risk         → "Escalate to senior assessor"
  - MISSING DOCS                 → "Request outstanding documents"
  - Critical Risk / Fraud Flag   → "Decline — policy exception required"
  - Prior Decline (resolved)     → "Proceed — note prior decline resolved"
- Write recommendation to AI_Recommended_Action__c
```

#### Topic 6: Draft Underwriting Notes
```
Instructions:
- Compile structured note combining summary, risks, and recommendation
- Format: [Date] | AI Assessment | [Summary] | [Risks] | [Action]
- Reference Data Cloud history where relevant
- Create Underwriting_Note__c with Is_AI_Generated__c = true
- Notes must be factual, objective, reference source documents
```

### 8.3 Agentforce Actions (Apex-backed)

| Action | Description | Apex Class |
|--------|-------------|------------|
| `GetUnifiedBorrowerProfile` | Queries Data Cloud for full borrower profile + insights | `DataCloudProfileAction` |
| `GetLoanApplicationData` | Retrieves full application + related records | `LoanAssessmentAgentAction` |
| `TriggerDocumentAI` | Initiates Document AI processing | `DocumentAITriggerAction` |
| `UpdateRiskRating` | Writes risk score to application | `RiskRatingUpdateAction` |
| `CreateUnderwritingNote` | Creates audit note record | `UnderwritingNoteAction` |
| `CheckDocumentCompleteness` | Validates required documents | `DocumentCompletenessAction` |
| `RequestMissingDocuments` | Triggers broker notification | `BrokerNotificationAction` |

### 8.4 Data Cloud as Agent Grounding — Example

```
Underwriter: "Give me the full borrower profile"
        │
        ▼
Agentforce queries Data Cloud Unified Profile
        │
        ├── Calculated Insights: DTI 54%, income avg $12,083/month, gambling 0.2%
        ├── Loan history: 3 prior apps, 1 declined 2022 (DTI was 78%)
        ├── Credit score trend: +42 points over 18 months (improving)
        └── Address consistency: 1 mismatch flagged (minor — suburb abbreviation)
        │
        ▼
Agentforce response:
"Borrower has applied 3 times since 2020. Previous decline in 2022 
 due to high DTI (78%). Current DTI has improved to 54% following 
 salary increase. Credit score up 42 points over 18 months — positive 
 trajectory. One minor address inconsistency — confirm with applicant. 
 Recommend conditional approval."
```

---

## 9. Salesforce Flow Automation

### 9.1 Flow: Document Upload Trigger
**Type:** Record-Triggered Flow (After Save on `ContentDocumentLink`)

```
1. Get parent Loan Application
2. Create Loan_Document__c record (status = Received)
3. Update Application_Status__c → "Documents Received"
4. Subflow: Check document completeness
5. If all required docs present → Trigger AI Processing Flow
```

### 9.2 Flow: AI Processing Orchestrator
**Type:** Autolaunched Flow

```
1. Set AI_Processing_Status__c = "In Progress"
2. Call External Service: Einstein Document AI (per document)
3. Await Platform Event: AI_Processing_Complete__e
4. Trigger: Data Cloud identity resolution + Calculated Insights refresh
5. Invoke Agentforce Agent via API
6. Set AI_Processing_Status__c = "Completed"
7. Update Application_Status__c = "Under Review"
8. Trigger: Underwriter Assignment Flow
```

### 9.3 Flow: Underwriter Routing
**Type:** Autolaunched Flow

```
Decision: Risk_Rating__c + Fraud_Risk_Score__c
├── Critical / High / Fraud Score > 70  → Senior Underwriter Queue
├── Medium                              → Standard Underwriter (Round Robin)
└── Low + Pre_Approved segment          → Fast-track queue

Actions:
- Create Task: "Review AI Assessment for [Application Number]"
- Send Email to assigned underwriter
- Update SLA_Due_Date__c
```

### 9.4 Flow: Broker Notification
**Type:** Triggered on `Missing_Documents__c` update

```
1. Generate personalised email listing missing documents
2. Send via Salesforce OrgWideEmail
3. Update Broker Portal Experience Cloud status
4. Create Task: "Follow up: Missing Documents"
5. Log notification in Underwriting_Note__c
```

### 9.5 Flow: Customer Status Notification
**Type:** Record-Triggered on `Application_Status__c` change

```
Status → Customer Message:
- "Documents received — under review"
- "Additional documents required"
- "Conditional approval — pending BAS"
- "Application approved / declined"
```

### 9.6 Platform Events

| Event | Purpose |
|-------|---------|
| `AI_Processing_Complete__e` | Signals Document AI finished |
| `Document_Requested__e` | Triggers broker notification |
| `Risk_Threshold_Breach__e` | Alerts senior team |
| `Application_Status_Change__e` | Drives customer notifications |
| `DataCloud_Profile_Updated__e` | Signals Data Cloud activation complete |

---

## 10. Lightning Web Components

### 10.1 `loanApplicationWorkspace` — Underwriter Console
- AI Summary Panel — Agentforce-generated borrower summary
- Data Cloud Profile Card — Unified history, prior apps, credit trend
- Risk Indicator Badges — Colour-coded risk alerts (Red / Amber / Green)
- Document Checklist — Live status with AI confidence scores
- Missing Documents Panel — One-click broker request
- AI Recommendation Card — Highlighted next action
- Agentforce Chat Panel — Embedded conversation interface

### 10.2 `brokerDocumentUpload` — Broker Portal (Experience Cloud)
- Multi-file drag-and-drop upload
- Document type selector with AI auto-classification fallback
- Real-time upload progress indicators
- Application stage tracker
- Notification centre for document requests

### 10.3 `loanDocumentViewer` — Document Preview + AI Overlay
- Side-by-side document preview with AI-extracted field highlights
- Confidence score indicators per field
- Underwriter override capability for extracted values
- Anomaly flags with visual callouts

### 10.4 `borrowerInsightsDashboard` — Management View (Data Cloud)
- Portfolio Risk Distribution — Live breakdown of all open applications
- Average Processing Time — SLA performance tracker
- AI Accuracy Rate — AI recommendation vs final decision
- Fraud Alerts Today — Applications with Fraud_Risk_Score > 70
- Broker Performance — Submission volume and document quality
- Income Verification Gap — Declared vs Data Cloud calculated income delta

---

## 11. Integration Architecture

```
Salesforce Platform
├── Einstein Document AI        (Native — no external API required)
├── Agentforce                  (Native — Agent Builder)
├── Data Cloud                  (Native — Data Streams + Identity Resolution)
├── Experience Cloud            (Broker Portal)
└── External Integrations (Optional):
    ├── Credit Bureau API       → Pull credit score during assessment
    ├── ATO API                 → Verify tax return lodgement status
    ├── ASIC API                → Verify company / ABN status
    └── Open Banking / Plaid    → Direct bank transaction feed
```

---

## 12. Security & Sharing Model

### 12.1 Profiles & Access

| Profile | Access Level |
|---------|-------------|
| Broker | Create / Read own Loan Applications; upload documents only |
| Underwriter | Read / Edit assigned applications; view all AI output |
| Senior Underwriter | All underwriter permissions + high-risk / fraud applications |
| Underwriting Manager | Full access; reporting; AI configuration |

### 12.2 Sharing Rules
- Loan Applications shared with the broker who submitted
- High-risk applications visible to Senior Underwriter Queue
- AI-generated notes are read-only for all users (system-written)
- Data Cloud profile data visible to Underwriter and above only

### 12.3 Field-Level Security

| Field | Restricted From |
|-------|----------------|
| `AI_Risk_Alerts__c` | Broker profile |
| `Assigned_Underwriter__c` | Broker profile |
| `AI_Extracted_Data__c` (raw JSON) | Underwriter and below |
| `Fraud_Risk_Score__c` | Broker profile |
| `Predicted_Default_Risk__c` | Broker profile |

---

## 13. POC Delivery Phases

### Phase 1 — Foundation (Week 1–2)
- [ ] Create all custom objects and fields
- [ ] Configure Document AI models (Payslip + Bank Statement first)
- [ ] Build broker document upload LWC
- [ ] Basic record-triggered flow on file upload

### Phase 2 — AI Core (Week 3–4)
- [ ] Configure Agentforce agent with all 6 topics
- [ ] Build Apex actions backing the agent
- [ ] Document completeness checker
- [ ] Risk rating logic with policy thresholds

### Phase 3 — Data Cloud (Week 4–5)
- [ ] Configure Data Streams (CRM + Loan App + Financial Data)
- [ ] Set up Identity Resolution rules
- [ ] Build all Calculated Insights
- [ ] Train and activate Einstein predictive models
- [ ] Configure Data Activation back to CRM
- [ ] Ground Agentforce agent with Data Cloud profile action

### Phase 4 — Automation (Week 5–6)
- [ ] All 5 Flows built and tested
- [ ] Platform Events wired up
- [ ] Underwriter routing logic
- [ ] Broker and customer notifications

### Phase 5 — UX & Polish (Week 6–7)
- [ ] Underwriter Console LWC (including Data Cloud profile card)
- [ ] Experience Cloud broker portal
- [ ] Document viewer with AI overlay
- [ ] Management insights dashboard
- [ ] End-to-end demo scenario rehearsal

---

## 14. Demo Scenario

### Enhanced Demo Script (with Data Cloud)

| Step | Action | What the Audience Sees |
|------|--------|------------------------|
| 1 | Broker logs into Experience Cloud portal | Clean upload interface |
| 2 | Broker uploads 5 loan documents | Real-time classification labels appear |
| 3 | Document AI processes each file | Extracted fields highlighted on document |
| 4 | Data Cloud resolves identity — recognises returning applicant | "Known customer — 2 prior applications found" |
| 5 | Agentforce generates full assessment | Summary, risk alerts, missing docs, recommendation |
| 6 | Data Cloud context enriches assessment | "Prior decline 2022 (DTI 78%) — now resolved at 54%" |
| 7 | Flow routes to correct underwriter queue | Underwriter receives task + email |
| 8 | Broker portal updates in real-time | "Application under review" |
| 9 | Underwriter opens console | Full AI assessment + Data Cloud profile card visible |
| 10 | Underwriter approves with one click | Customer notification triggered automatically |

**Total elapsed time: under 60 seconds from upload to underwriter notification.**

---

## 15. Key Success Metrics

| Metric | Before | Target (Post-POC) |
|--------|--------|-------------------|
| Time to process document pack | 45–90 mins | < 3 minutes |
| Underwriter summary writing time | 20–30 mins | Eliminated |
| Missing document detection accuracy | ~70% (manual) | 95%+ (AI) |
| Broker follow-up cycle time | 2–3 days | < 4 hours |
| Assessment consistency | Variable per assessor | Standardised |
| Repeat applicant recognition | Manual CRM lookup | Automatic (Data Cloud) |
| Default prediction accuracy | N/A | 80%+ (Einstein model) |
| Fraud detection rate | ~40% manual | 85%+ (Data Cloud AI) |

---

## Appendix: Technology Stack Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Document Processing | Einstein Document AI | Classification + field extraction |
| AI Agent | Agentforce | Underwriting assessment + recommendations |
| Customer Intelligence | Salesforce Data Cloud | Unified profile + predictive scoring |
| Automation | Salesforce Flow | Routing, notifications, orchestration |
| Broker Interface | Experience Cloud + LWC | Document upload portal |
| Underwriter Interface | Service Cloud Console + LWC | Assessment workspace |
| Reporting | CRM Analytics + Data Cloud | Management dashboards |
| Identity | Data Cloud Identity Resolution | Unified borrower across sources |
| Predictive AI | Einstein on Data Cloud | Default risk + fraud scoring |

---

*Document prepared by Ravi K Telu — ravikishantelu@gmail.com*
*Solution design based on Salesforce API v65.0 / Agentforce GA / Data Cloud*
