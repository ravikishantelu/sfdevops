# AI-Powered Loan Application Assessment Platform — Phase 2 AI Core

**Date:** 2026-05-25
**Author:** Documentation Agent
**Status:** Completed — Awaiting Named Credential Secret, Flow Activation, and Agentforce Setup
**API Version:** 66.0

---

## Overview

### Original Request

Phase 2 — AI Core builds on the Phase 1 data model and broker upload experience. The scope delivered in this phase:

1. Six Apex `@InvocableMethod` action classes that back the Agentforce Underwriting Assistant topics
2. A shared policy constants class containing all hard-coded business rule thresholds
3. An AI Processing Orchestrator Flow triggered when a Loan Application's `AI_Processing_Status__c` is set to "In Progress"
4. Named Credential and External Credential metadata for the Einstein Document AI callout endpoint
5. Test classes for every new Apex class
6. A post-deployment admin setup guide for configuring the Agentforce agent in Setup (see `docs/agentforce-setup-guide.md`)

Out of scope for Phase 2 (deferred to Phase 3 or not requested):
- Data Cloud unified borrower profile (Topic 1 returns CRM data only)
- Einstein Prediction Builder for `Predicted_Default_Risk__c` and `Fraud_Risk_Score__c`
- New permission sets (Phase 1 sets cover all fields added here)
- Custom Metadata Type for policy thresholds (user explicitly requested hard-coded values)

### Business Objective

Phase 1 established the data model and gave brokers a way to upload documents into Salesforce. Phase 2 connects those documents to AI. When an underwriter or automated process marks a loan application as "In Progress", the Orchestrator Flow sends each uploaded document to Einstein Document AI, extracts financial data into structured records, and makes that data available to an Agentforce agent. The Agentforce Underwriting Assistant can then answer underwriter questions, compute risk ratings against policy thresholds, confirm document completeness, draft underwriting notes, and request missing documents from the broker — all without the underwriter manually reading raw document files.

### Summary

Phase 2 delivers one constants class, six invocable action classes, one orchestrating Flow, two Named Credential metadata files, and seven matching test classes. No new Salesforce objects, fields, permission sets, validation rules, or page layouts are created — all Apex writes target Phase 1 fields that already exist on the four Phase 1 objects. The Agentforce agent itself is configured manually by a human admin in Setup after deployment, guided by `docs/agentforce-setup-guide.md`.

---

## Data Model Context

Phase 2 writes to existing Phase 1 fields only. No schema changes were made. The relevant fields written by Phase 2 actions are:

| Object | Field API Name | Written By |
|--------|----------------|------------|
| `Loan_Application__c` | `AI_Summary__c` | Agentforce agent (via record update step in Setup) |
| `Loan_Application__c` | `AI_Risk_Alerts__c` | `TFS_RiskRatingUpdateAction` |
| `Loan_Application__c` | `Risk_Rating__c` | `TFS_RiskRatingUpdateAction` |
| `Loan_Application__c` | `Documents_Complete__c` | `TFS_DocumentCompletenessAction` |
| `Loan_Application__c` | `Missing_Documents__c` | `TFS_DocumentCompletenessAction` / `TFS_BrokerNotificationAction` |
| `Loan_Application__c` | `AI_Recommended_Action__c` | Agentforce agent (via record update step in Setup) |
| `Loan_Application__c` | `AI_Processing_Status__c` | Orchestrator Flow (sets to Completed or Failed) |
| `Loan_Application__c` | `Application_Status__c` | Orchestrator Flow (sets to Under Review on success) |
| `Loan_Document__c` | `AI_Confidence_Score__c` | `TFS_DocumentAITriggerAction` |
| `Loan_Document__c` | `AI_Extracted_Data__c` | `TFS_DocumentAITriggerAction` |
| `Loan_Document__c` | `Document_Status__c` | `TFS_DocumentAITriggerAction` (sets to Verified) |
| `Extracted_Financial_Data__c` | All financial fields | `TFS_DocumentAITriggerAction` (creates new record) |
| `Underwriting_Note__c` | All fields | `TFS_UnderwritingNoteAction` (creates new record) |

---

## Components Created

### Admin Components (Declarative)

#### Named Credential

| Component | API Name | Label | URL | Auth Protocol |
|-----------|----------|-------|-----|---------------|
| Named Credential | `TFS_Einstein_Document_AI` | TFS Einstein Document AI | `https://api.salesforce.com/einstein/document-ai` | Custom (via External Credential) |
| External Credential | `TFS_Einstein_Document_AI_Ext` | TFS Einstein Document AI Ext | — | Custom (Named Principal; token set post-deploy) |

The Named Credential shell is deployed as metadata. A human admin must set the principal token in Setup after deployment and before any callout can succeed. The secret is never committed to source control.

Apex references the endpoint as: `callout:TFS_Einstein_Document_AI`

#### Flows

| Flow API Name | Label | Type | Trigger Object | Trigger Event | Run Mode | Status on Deploy |
|---------------|-------|------|----------------|---------------|----------|-----------------|
| `TFS_AI_Processing_Orchestrator` | TFS AI Processing Orchestrator | AutoLaunchedFlow (Record-Triggered) | `Loan_Application__c` | After Update | AsyncAfterCommit | Draft — must be activated manually |

---

### Development Components (Code)

#### Apex Classes — Action Classes

| Class API Name | InvocableMethod Label | Agentforce Topic | Description |
|----------------|-----------------------|------------------|-------------|
| `TFS_LoanPolicyConstants` | N/A (constants only) | All topics | Hard-coded DTI thresholds, gambling flag percentage, income multiplier, expense ratio, required document checklists, risk rating strings |
| `TFS_LoanAssessmentAgentAction` | Get Loan Application Data | Topics 1, 2, 5 | Returns the unified loan + borrower profile (loan fields + most-recent EFD snapshot + document count) as individual `@InvocableVariable` fields and as a JSON string |
| `TFS_DocumentAITriggerAction` | Trigger Document AI | Orchestrator Flow | Calls Einstein Document AI via Named Credential, creates `Extracted_Financial_Data__c`, updates `Loan_Document__c` status and confidence score |
| `TFS_RiskRatingUpdateAction` | Update Risk Rating | Topic 3 | Computes DTI from EFD, applies policy thresholds, writes `Risk_Rating__c` and `AI_Risk_Alerts__c` |
| `TFS_UnderwritingNoteAction` | Create Underwriting Note | Topic 6 | Inserts an AI-generated `Underwriting_Note__c` authored by the running user |
| `TFS_DocumentCompletenessAction` | Check Document Completeness | Topic 4 | Compares uploaded document counts against the employment-type checklist, writes `Documents_Complete__c` and `Missing_Documents__c` |
| `TFS_BrokerNotificationAction` | Request Missing Documents | Topic 4 / 5 follow-up | Emails the broker's account owner a missing-documents list via `Messaging.SingleEmailMessage` |

#### Method Signatures

**TFS_LoanAssessmentAgentAction**
```apex
@InvocableMethod(label='Get Loan Application Data'
                 description='Returns the unified loan + borrower profile for an Agentforce topic'
                 category='TFS Loan Assessment')
public static List<Response> getData(List<Request> requests)
```
Input: `Id loanApplicationId` (required)
Output: `String applicantName`, `Decimal loanAmount`, `String loanPurpose`, `String applicationStatus`, `String aiProcessingStatus`, `Decimal annualIncome`, `Decimal monthlyExpenses`, `Decimal monthlyLiabilities`, `Boolean paygStatus`, `String employerName`, `Decimal employmentDurationYears`, `Integer documentCount`, `String summaryJson`

**TFS_DocumentAITriggerAction**
```apex
@InvocableMethod(label='Trigger Document AI'
                 description='Calls Einstein Document AI, parses response, creates Extracted_Financial_Data__c'
                 category='TFS Loan Assessment'
                 callout=true)
public static List<Response> run(List<Request> requests)
```
Input: `Id loanDocumentId` (required)
Output: `Boolean success`, `String message`, `Id extractedFinancialDataId`

**TFS_RiskRatingUpdateAction**
```apex
@InvocableMethod(label='Update Risk Rating'
                 description='Applies policy thresholds; sets Risk_Rating__c and AI_Risk_Alerts__c'
                 category='TFS Loan Assessment')
public static List<Response> run(List<Request> requests)
```
Input: `Id loanApplicationId` (required)
Output: `String riskRating`, `String alerts`, `Decimal computedDtiPct`

**TFS_UnderwritingNoteAction**
```apex
@InvocableMethod(label='Create Underwriting Note'
                 description='Inserts an AI-generated Underwriting_Note__c'
                 category='TFS Loan Assessment')
public static List<Response> run(List<Request> requests)
```
Input: `Id loanApplicationId` (required), `String noteContent` (required)
Output: `Id underwritingNoteId`, `Boolean success`

**TFS_DocumentCompletenessAction**
```apex
@InvocableMethod(label='Check Document Completeness'
                 description='Validates uploaded docs against the required checklist'
                 category='TFS Loan Assessment')
public static List<Response> run(List<Request> requests)
```
Input: `Id loanApplicationId` (required)
Output: `Boolean documentsComplete`, `String missingDocuments`, `List<String> missingDocList`

**TFS_BrokerNotificationAction**
```apex
@InvocableMethod(label='Request Missing Documents'
                 description='Emails the broker a list of missing documents'
                 category='TFS Loan Assessment')
public static List<Response> run(List<Request> requests)
```
Input: `Id loanApplicationId` (required), `String missingDocumentsSummary` (optional)
Output: `Boolean emailSent`, `String message`

#### Apex Test Classes

| Test Class | Tests For | Key Scenarios |
|------------|-----------|---------------|
| `TFS_LoanPolicyConstantsTest` | `TFS_LoanPolicyConstants` | All constant values, threshold ordering, distinct risk strings, both employment checklists |
| `TFS_LoanAssessmentAgentActionTest` | `TFS_LoanAssessmentAgentAction` | Single + bulk positive, with/without EFD, no documents, null input, unmatched Id, 50-record bulk |
| `TFS_DocumentAITriggerActionTest` | `TFS_DocumentAITriggerAction` | Null/empty input, batch cap breach, missing ContentDocument_Id, missing ContentVersion, HTTP non-200, callout success with full EFD, 10-document bulk |
| `TFS_RiskRatingUpdateActionTest` | `TFS_RiskRatingUpdateAction` | Low/Medium/High/Critical DTI bands, gambling flag, income multiplier alert, expense ratio upgrade, null income guard, no EFD |
| `TFS_UnderwritingNoteActionTest` | `TFS_UnderwritingNoteAction` | Valid insert, null Id, blank content, mixed valid/invalid batch |
| `TFS_DocumentCompletenessActionTest` | `TFS_DocumentCompletenessAction` | Complete set (Employed), complete set (Self-Employed), partial missing, no EFD default, no documents |
| `TFS_BrokerNotificationActionTest` | `TFS_BrokerNotificationAction` | Email sent, no broker, no email, blank summary, custom summary overrides stored, DML failure handling |

---

## Policy Thresholds (TFS_LoanPolicyConstants)

All values are hard-coded as Apex constants per the user's request. To change thresholds without redeployment, migrate to a Custom Metadata Type in a future phase.

### DTI Thresholds

| Constant | Value | Meaning |
|----------|-------|---------|
| `DTI_CRITICAL` | 70 | Annualised liabilities / annual income >= 70% -> Critical rating |
| `DTI_HIGH` | 50 | >= 50% and < 70% -> High rating |
| `DTI_MEDIUM` | 30 | >= 30% and < 50% -> Medium rating |
| (below DTI_MEDIUM) | — | < 30% -> Low rating |

DTI formula: `(Monthly_Liabilities__c * 12) / Annual_Income__c * 100`

### Non-DTI Alert Rules

| Constant | Value | Alert Condition |
|----------|-------|-----------------|
| `GAMBLING_SPEND_FLAG_PCT` | 5 | If `Gambling_Indicators__c` is true on the EFD record, append "Gambling spend flag detected" |
| `INCOME_LOAN_MULTIPLIER` | 3 | If annual income < loan amount * 3, append "Income below 3x loan amount" |
| `EXPENSE_RATIO_HIGH_PCT` | 70 | If monthly expenses / monthly income > 70%, upgrade rating to at least High and append "Expense ratio exceeds 70%" |

Note on gambling flag: `Gambling_Indicators__c` is a boolean checkbox on `Extracted_Financial_Data__c`. Phase 2 treats `true` as the flag because no per-transaction spend data is available. If a numeric `Gambling_Spend__c` field is added in a future phase, the rule should compute `(gamblingSpend / monthlyIncome) * 100 > GAMBLING_SPEND_FLAG_PCT` instead.

### Risk Rating String Constants

| Constant | Value |
|----------|-------|
| `RISK_LOW` | `'Low'` |
| `RISK_MEDIUM` | `'Medium'` |
| `RISK_HIGH` | `'High'` |
| `RISK_CRITICAL` | `'Critical'` |

### Required Document Checklists (REQUIRED_DOCS_BY_EMPLOYMENT)

Employment type is derived from `Extracted_Financial_Data__c.PAYG_Status__c`: true = Employed, false = Self-Employed. If no EFD record exists, defaults to Employed.

| Document Type | Employed | Self-Employed |
|---------------|----------|---------------|
| Payslip | 2 | 2 |
| Bank Statement | 3 | 3 |
| Tax Return | 1 | 1 |
| ID Document | 1 | 1 |
| Loan Form | 1 | 1 |
| BAS | — | 2 |

---

## Data Flow

### How a Document Moves from Upload to Completed AI Processing

```
1. Broker uploads a file via brokerDocumentUpload LWC (Phase 1)
   -> Salesforce creates ContentDocument + ContentVersion
   -> TFS_ContentDocumentLink_LoanDoc_Create flow creates Loan_Document__c
      (Document_Status__c = 'Received')

2. Underwriter (or process) sets AI_Processing_Status__c = 'In Progress'
   on the Loan_Application__c record (manual edit or automated trigger)

3. TFS_AI_Processing_Orchestrator flow fires (After Update, AsyncAfterCommit)
   Entry condition: ISCHANGED(AI_Processing_Status__c) AND value = 'In Progress'
   -> Gets all Loan_Document__c WHERE Document_Status__c = 'Received'
   -> Loops through each document

4. For each Loan_Document__c in the loop:
   -> Calls TFS_DocumentAITriggerAction (callout=true)
   -> Action reads Loan_Document__c and its ContentVersion (binary file)
   -> Builds HTTP POST to callout:TFS_Einstein_Document_AI
      Body: { "documentType": "<type>", "content": "<base64>" }
   -> On HTTP 200:
      a. Parses JSON response
      b. Creates Extracted_Financial_Data__c with financial fields
      c. Updates Loan_Document__c:
         - AI_Confidence_Score__c = response.confidenceScore
         - AI_Extracted_Data__c   = raw JSON response
         - Document_Status__c     = 'Verified'
   -> On any failure: returns success=false, message; flow continues to next doc

5. After the loop completes:
   -> Flow sets Loan_Application__c.AI_Processing_Status__c = 'Completed'
   -> Flow sets Loan_Application__c.Application_Status__c   = 'Under Review'

6. On any fault in the flow:
   -> Flow sets Loan_Application__c.AI_Processing_Status__c = 'Failed'

7. Underwriter opens Agentforce Underwriting Assistant and asks a question.
   The agent routes the query to the appropriate topic, which calls the
   relevant Apex action to read or write data.
```

### Agentforce Actions Data Flow (Post-Extraction)

```
Underwriter query: "What is the risk profile for application <Id>?"
           |
           v
Agentforce routes to Topic 3 (Risk Assessment)
           |
           v
TFS_RiskRatingUpdateAction.run([Request(loanApplicationId)])
    Queries Loan_Application__c (Loan_Amount__c)
    Queries newest Extracted_Financial_Data__c (income, expenses, liabilities, gambling flag)
    Computes DTI -> applies TFS_LoanPolicyConstants thresholds
    Evaluates non-DTI alert rules
    Writes Risk_Rating__c + AI_Risk_Alerts__c (WITH USER_MODE)
    Returns riskRating, alerts, computedDtiPct
           |
           v
Agent displays computed rating and alerts to underwriter
```

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Salesforce Org                                   │
│                                                                         │
│  ┌──────────────────────┐      ┌────────────────────────────────────┐   │
│  │  Loan_Application__c │      │  TFS_AI_Processing_Orchestrator    │   │
│  │                      │─────>│  Flow (Record-Triggered, Async)    │   │
│  │  AI_Processing_      │      │                                    │   │
│  │  Status__c =         │      │  Get Loan_Document__c (Received)   │   │
│  │  "In Progress"       │      │  Loop each document                │   │
│  └──────────────────────┘      │    └> TFS_DocumentAITriggerAction  │   │
│                                │         HTTP POST (callout)        │   │
│                                │         -> Einstein Document AI    │   │
│                                │         <- JSON response           │   │
│                                │         Insert EFD record          │   │
│                                │         Update Loan_Document__c    │   │
│                                │  Set Status = Completed / Failed   │   │
│                                └────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────┐      ┌────────────────────────────────────┐   │
│  │  Agentforce          │      │  Apex Action Classes               │   │
│  │  Underwriting        │─────>│                                    │   │
│  │  Assistant           │      │  TFS_LoanAssessmentAgentAction     │   │
│  │  (6 Topics)          │      │  TFS_RiskRatingUpdateAction        │   │
│  │                      │      │  TFS_DocumentCompletenessAction    │   │
│  │  Underwriter UI      │      │  TFS_UnderwritingNoteAction        │   │
│  └──────────────────────┘      │  TFS_BrokerNotificationAction      │   │
│                                └────────────────────────────────────┘   │
│                                             |                           │
│                                             v                           │
│                          ┌──────────────────────────────────────────┐   │
│                          │           Salesforce Data Layer           │   │
│                          │                                           │   │
│                          │  Loan_Application__c                      │   │
│                          │  Loan_Document__c                         │   │
│                          │  Extracted_Financial_Data__c              │   │
│                          │  Underwriting_Note__c                     │   │
│                          └──────────────────────────────────────────┘   │
│                                                                         │
│                          ┌──────────────────────────────────────────┐   │
│                          │  External: Einstein Document AI           │   │
│                          │  callout:TFS_Einstein_Document_AI         │   │
│                          │  (Named Credential)                       │   │
│                          └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Agentforce Agent Overview

The Agentforce Underwriting Assistant is configured manually in Setup after deployment. It is an Internal (Employee) type agent. It has six topics, each backed by one or more of the Apex action classes in the `TFS Loan Assessment` InvocableMethod category.

| # | Topic Name | Apex Action(s) Wired | Purpose |
|---|------------|----------------------|---------|
| 1 | Unified Borrower Profile Retrieval | `TFS_LoanAssessmentAgentAction` | Returns the full loan application data and financial snapshot so the agent can answer factual questions about a borrower |
| 2 | Borrower Summary Generation | `TFS_LoanAssessmentAgentAction` | Same data retrieval; agent prompt composes a narrative summary and writes `AI_Summary__c` via an agent record-update step |
| 3 | Risk Assessment | `TFS_RiskRatingUpdateAction` | Computes DTI and applies policy thresholds; writes `Risk_Rating__c` and `AI_Risk_Alerts__c` |
| 4 | Document Completeness Check | `TFS_DocumentCompletenessAction` + optional `TFS_BrokerNotificationAction` | Checks required vs uploaded documents; optionally emails the broker about gaps |
| 5 | Recommended Action | `TFS_LoanAssessmentAgentAction` + `TFS_RiskRatingUpdateAction` | Gathers profile and risk data; agent composes recommendation text and writes `AI_Recommended_Action__c` |
| 6 | Draft Underwriting Notes | `TFS_UnderwritingNoteAction` | Creates an AI-generated `Underwriting_Note__c` from agent-composed text |

For detailed step-by-step instructions to configure each topic in Setup, see `docs/agentforce-setup-guide.md`.

---

## File Locations

| Component | Path |
|-----------|------|
| Named Credential | `force-app/main/default/namedCredentials/TFS_Einstein_Document_AI.namedCredential-meta.xml` |
| External Credential | `force-app/main/default/externalCredentials/TFS_Einstein_Document_AI_Ext.externalCredential-meta.xml` |
| Orchestrator Flow | `force-app/main/default/flows/TFS_AI_Processing_Orchestrator.flow-meta.xml` |
| Policy Constants | `force-app/main/default/classes/TFS_LoanPolicyConstants.cls` |
| Loan Assessment Action | `force-app/main/default/classes/TFS_LoanAssessmentAgentAction.cls` |
| Document AI Action | `force-app/main/default/classes/TFS_DocumentAITriggerAction.cls` |
| Risk Rating Action | `force-app/main/default/classes/TFS_RiskRatingUpdateAction.cls` |
| Underwriting Note Action | `force-app/main/default/classes/TFS_UnderwritingNoteAction.cls` |
| Document Completeness Action | `force-app/main/default/classes/TFS_DocumentCompletenessAction.cls` |
| Broker Notification Action | `force-app/main/default/classes/TFS_BrokerNotificationAction.cls` |
| Policy Constants Test | `force-app/main/default/classes/TFS_LoanPolicyConstantsTest.cls` |
| Loan Assessment Action Test | `force-app/main/default/classes/TFS_LoanAssessmentAgentActionTest.cls` |
| Document AI Action Test | `force-app/main/default/classes/TFS_DocumentAITriggerActionTest.cls` |
| Risk Rating Action Test | `force-app/main/default/classes/TFS_RiskRatingUpdateActionTest.cls` |
| Underwriting Note Action Test | `force-app/main/default/classes/TFS_UnderwritingNoteActionTest.cls` |
| Document Completeness Action Test | `force-app/main/default/classes/TFS_DocumentCompletenessActionTest.cls` |
| Broker Notification Action Test | `force-app/main/default/classes/TFS_BrokerNotificationActionTest.cls` |
| Agentforce setup guide | `docs/agentforce-setup-guide.md` |

---

## Security

### Apex Security Model

All Phase 2 Apex classes follow the same pattern as Phase 1:

| Practice | Implementation |
|----------|----------------|
| Sharing enforcement | All classes declared `public with sharing` — org sharing rules are enforced at runtime |
| FLS enforcement on reads | All SOQL uses `WITH USER_MODE` — field-level security and object permissions are enforced at query level (API 66.0) |
| FLS enforcement on writes | All DML uses `AccessLevel.USER_MODE` via `Database.insert/update(..., AccessLevel.USER_MODE)` |
| Callout isolation | `TFS_DocumentAITriggerAction` uses `callout=true` on `@InvocableMethod`. Callouts run per-document in a loop managed by the Flow's `NewTransaction` model, which keeps each callout in its own transaction context |
| Savepoints | `TFS_DocumentAITriggerAction` sets a `Database.Savepoint` before the callout so DML can be rolled back cleanly on a partial failure within a single document's processing |
| Error surfacing | All action classes catch exceptions and return them in the Response (`success=false`, `message=...`) rather than re-throwing. This allows the calling Flow or Agentforce topic to continue with remaining items |
| No stack trace exposure | Errors are returned as plain string messages, not propagated as system exceptions to the UI |

### Named Credential Security

The `TFS_Einstein_Document_AI` Named Credential uses an External Credential (`TFS_Einstein_Document_AI_Ext`) with a Custom authentication protocol. The principal token (API key or bearer token) is never committed to source control. A human admin sets it post-deployment via:

Setup > Named Credentials > External Credentials > TFS Einstein Document AI Ext > Principals

### Broker Email Resolution

`TFS_BrokerNotificationAction` resolves the broker email via `Account.Owner.Email` (the account owner of the Account record pointed to by `Loan_Application__c.Broker__c`). If the org uses Person Accounts and the broker is stored as a Person Account, the query should instead use `Account.PersonEmail`. This assumption is documented in the class header and requires admin confirmation before production use.

---

## Post-Deployment Checklist

Complete these steps in order after deploying Phase 2 metadata to the target org.

### Step 1 — Set the Named Credential Secret

1. Go to Setup > Named Credentials > External Credentials.
2. Click **TFS Einstein Document AI Ext**.
3. In the Principals section, click the pencil icon next to the named principal.
4. Enter the Einstein Document AI API token in the **Token** field.
5. Save.

The callout endpoint (`callout:TFS_Einstein_Document_AI`) will fail with an authentication error until this step is complete.

### Step 2 — Activate the Orchestrator Flow

1. Go to Setup > Flows.
2. Open **TFS AI Processing Orchestrator**.
3. The flow is deployed in **Draft** status.
4. Click **Activate**.
5. Confirm the status shows **Active**.

### Step 3 — Configure the Agentforce Agent

Follow all steps in `docs/agentforce-setup-guide.md` to:
- Enable Agentforce in Setup
- Create the TFS Underwriting Assistant agent
- Configure each of the 6 topics with classifications, instructions, and action wiring
- Test the agent in Agent Builder preview
- Activate the agent and assign the Agent User to the `TFS_Underwriter` permission set

### Step 4 — Smoke Test the AI Flow

1. Create or select a Loan Application that has at least one `Loan_Document__c` in `Document_Status__c = 'Received'` status and a linked file.
2. Manually change `AI_Processing_Status__c` to `In Progress` and save.
3. Wait 30–60 seconds (async flow).
4. Verify `AI_Processing_Status__c` changes to `Completed` and `Application_Status__c` changes to `Under Review`.
5. Verify at least one `Extracted_Financial_Data__c` record is created and linked to the application.
6. Verify the `Loan_Document__c` record shows `Document_Status__c = 'Verified'` and a confidence score.

### Step 5 — Smoke Test the Agentforce Agent

1. Open the TFS Underwriting Assistant in Agent Builder.
2. In the preview pane, type: `Assess loan application <paste a valid Loan Application Id>`.
3. Verify the agent responds with borrower profile data (Topic 1).
4. Type: `What is the risk rating?` to trigger Topic 3 and verify `Risk_Rating__c` is updated on the record.
5. Type: `Check document completeness` to trigger Topic 4 and verify `Documents_Complete__c` is updated.

---

## Testing

### Test Coverage Summary

| Class | Test Class | Expected Coverage |
|-------|------------|-------------------|
| `TFS_LoanPolicyConstants` | `TFS_LoanPolicyConstantsTest` | ~100% |
| `TFS_LoanAssessmentAgentAction` | `TFS_LoanAssessmentAgentActionTest` | ~95% |
| `TFS_DocumentAITriggerAction` | `TFS_DocumentAITriggerActionTest` | ~90% |
| `TFS_RiskRatingUpdateAction` | `TFS_RiskRatingUpdateActionTest` | ~95% |
| `TFS_UnderwritingNoteAction` | `TFS_UnderwritingNoteActionTest` | ~95% |
| `TFS_DocumentCompletenessAction` | `TFS_DocumentCompletenessActionTest` | ~95% |
| `TFS_BrokerNotificationAction` | `TFS_BrokerNotificationActionTest` | ~90% |

`TFS_DocumentAITriggerActionTest` uses `HttpCalloutMock` implementations (SuccessMock and a failure mock) to simulate the Einstein Document AI HTTP response without making live callouts.

### Bulkification

All six action classes are bulkified: SOQL queries run once across all input IDs collected from the full request list, and DML is batched into single operations. `TFS_DocumentAITriggerAction` is the exception — callouts are issued per-document inside the Flow loop because each has a distinct response payload, and the Flow uses `NewTransaction` to comply with callout-before-DML rules.

---

## Known Limitations

| Limitation | Detail |
|------------|--------|
| Orchestrator Flow deployed as Draft | Must be activated manually in Setup after the Named Credential secret is set |
| Named Credential secret not in source | The API token must be entered manually in Setup; it cannot be deployed from metadata |
| Batch cap of 10 on Document AI action | `TFS_DocumentAITriggerAction` rejects invocations with more than 10 documents per call. The Flow feeds one document per loop iteration so this cap is a safety net. Applications with more than 10 documents in a single AI processing run would need a Queueable refactor |
| Gambling flag is a checkbox | No per-transaction spend data is available in Phase 2. The `Gambling_Indicators__c` boolean is treated as the flag itself. Future phases should add a numeric spend field for more precise evaluation |
| Broker email via Account.Owner.Email | If the org uses Person Accounts for brokers, the query must switch to `Account.PersonEmail`. Admin must confirm the data model before production use |
| AI_Summary__c and AI_Recommended_Action__c | These fields are populated by the Agentforce agent's built-in record-update step, not by an Apex action. The admin must configure this step in each relevant topic in Setup. The fields will remain blank until the agent is configured and invoked |
| No real-time push notifications | The missing-documents email is sent on demand by `TFS_BrokerNotificationAction`. There are no automated triggers that email brokers when AI processing completes |

### What Phase 3 Adds

| Feature | Description |
|---------|-------------|
| Data Cloud integration | Unified borrower profile enriched with historical data, not just CRM fields |
| Einstein Prediction Builder | Populates `Predicted_Default_Risk__c` and `Fraud_Risk_Score__c` with model-driven scores |
| Underwriter LWC | Dedicated Lightning Web Component for underwriters showing AI outputs, risk scores, alerts, and recommended actions in one panel |
| Anomaly detection | Logic to evaluate extracted data for anomalies and set `Anomaly_Detected__c` and `Anomaly_Description__c` on `Loan_Document__c` |
| Custom Metadata for thresholds | Migrates hard-coded constants from `TFS_LoanPolicyConstants` to Custom Metadata so thresholds can be changed without redeployment |

---

## Notes and Considerations

### Design Decisions

- **Hard-coded thresholds over Custom Metadata** — The user explicitly requested that Phase 2 policy thresholds be hard-coded in Apex. `TFS_LoanPolicyConstants` is the single source of truth for all threshold values. This makes them easy to review but requires redeployment to change. Phase 3 can migrate them to Custom Metadata if needed.
- **Record-Triggered Flow over standalone autolaunched flow** — The Orchestrator Flow is record-triggered on `Loan_Application__c` after update, running on the `AsyncAfterCommit` path. This is the Salesforce-recommended pattern for async callouts from flows because the async path runs after the record commit, satisfying the callout-before-DML rule without requiring a separate invocation mechanism.
- **Failure isolation in the document loop** — `TFS_DocumentAITriggerAction` never re-throws exceptions. Each document is processed independently. A failed callout on document 2 does not prevent document 3 from being processed. The Flow's fault connector on the action call also routes back to the loop continuation rather than terminating, ensuring maximum documents are attempted.
- **Savepoint rollback on partial failure** — Each document's Apex processing uses a `Database.Savepoint` before the callout. If the EFD insert or document update fails after a successful callout, the DML is rolled back cleanly for that document.
- **summaryJson on TFS_LoanAssessmentAgentAction** — In addition to individual `@InvocableVariable` outputs, the action serialises the full Response to a JSON string (`summaryJson`). The Agentforce topic prompt can pass this single variable to the LLM without enumerating every field, making the topic configuration simpler to maintain.

### Dependencies

| Dependency | Detail |
|------------|--------|
| Phase 1 must be deployed first | All four objects and their fields must exist before Phase 2 can deploy |
| Einstein Document AI model must be active | Follow `docs/einstein-document-ai-setup.md` to train and activate the models before testing callouts |
| Named Credential secret must be set | Callouts fail with HTTP 401 or connection error until the admin sets the principal token post-deploy |
| Orchestrator Flow must be activated | The Flow deploys in Draft status; document AI processing will not trigger until it is activated |
| Agentforce license required | The TFS Underwriting Assistant requires an Agentforce for Salesforce license assigned to underwriter users |

---

## Change History

| Date | Author | Description |
|------|--------|-------------|
| 2026-05-25 | Documentation Agent | Initial creation — Phase 2 AI Core complete |
