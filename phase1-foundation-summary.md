# AI-Powered Loan Application Assessment Platform — Phase 1 Foundation

**Date:** 2026-05-24
**Author:** Documentation Agent
**Status:** Completed — Awaiting Post-Deployment Activation Steps
**API Version:** 66.0

---

## Overview

### Original Request

Build Phase 1 of an AI-Powered Loan Application Assessment Platform on Salesforce. The platform will ultimately automate the end-to-end loan document assessment lifecycle — from broker document submission through to underwriter recommendation — using Agentforce, Einstein Document AI, Data Cloud, Flow Automation, and Lightning Web Components.

Phase 1 establishes the data model and broker-facing document upload experience. AI analysis and underwriter tooling are delivered in later phases.

### Business Objective

Underwriters and credit analysts currently spend significant time reading document packs manually, extracting borrower information, writing summaries, identifying missing documents, and following up brokers. This slows approval cycles and creates inconsistency across assessors.

Phase 1 solves the intake problem: it gives brokers a structured, guided way to submit loan applications and upload supporting documents directly into Salesforce, while creating the data structures that later phases will populate with AI-extracted intelligence.

### Summary

Phase 1 delivers four custom Salesforce objects (the core data model), two permission sets (broker and underwriter access tiers), an Apex controller, a test class, a Lightning Web Component for broker document upload, a record-triggered flow that links uploaded files to loan document records, and a manual runbook for Einstein Document AI model configuration in Setup. The broker-facing LWC is deployed on both the Loan Application record page and the Experience Cloud broker portal.

---

## Data Model

### Object Relationship Diagram

```
                          ┌─────────────────────────────┐
                          │      Loan_Application__c     │
                          │  (Master — Auto Number LA-#) │
                          └──────────────┬──────────────┘
                                         │
               ┌─────────────────────────┼──────────────────────┐
               │ Master-Detail           │ Master-Detail         │ Lookup
               ▼                         ▼                        ▼
  ┌─────────────────────┐  ┌──────────────────────┐  ┌───────────────────────────┐
  │  Loan_Document__c   │  │ Underwriting_Note__c │  │ Extracted_Financial_Data__c│
  │  (Auto Number LD-#) │  │  (Auto Number UN-#)  │  │   (Auto Number EFD-#)     │
  └──────────┬──────────┘  └──────────────────────┘  └───────────────────────────┘
             │                                                    ▲
             │ Lookup                                             │ Lookup
             └────────────────────────────────────────────────────┘
```

### Relationship Summary

| Child Object | Relationship Type | Parent | Relationship Name |
|---|---|---|---|
| `Loan_Document__c` | Master-Detail | `Loan_Application__c` | Loan_Documents |
| `Underwriting_Note__c` | Master-Detail | `Loan_Application__c` | Underwriting_Notes |
| `Extracted_Financial_Data__c` | Lookup | `Loan_Application__c` | Extracted_Financials |
| `Extracted_Financial_Data__c` | Lookup | `Loan_Document__c` | Extracted_Financials |

---

## Components Created

### Admin Components (Declarative)

#### Custom Objects

| Object API Name | Label | Name Field | Auto-Number Format | Sharing Model |
|---|---|---|---|---|
| `Loan_Application__c` | Loan Application | Application_Number__c | LA-{0000} | ReadWrite |
| `Loan_Document__c` | Loan Document | Loan_Document_Name | LD-{0000} | ControlledByParent |
| `Extracted_Financial_Data__c` | Extracted Financial Data | Record_Name | EFD-{0000} | ReadWrite |
| `Underwriting_Note__c` | Underwriting Note | Note_Name | UN-{0000} | ControlledByParent |

#### Custom Fields — Loan_Application__c

| Field API Name | Type | Detail | Notes |
|---|---|---|---|
| `Applicant_Name__c` | Text(255) | — | Broker-editable |
| `Loan_Amount__c` | Currency(16,2) | — | Broker-editable |
| `Loan_Purpose__c` | Picklist | Home, Investment, Refinance, Personal | Broker-editable |
| `Application_Status__c` | Picklist | New (default), Documents Received, AI Processing, Under Review, Conditional Approval, Approved, Declined | Drives stage tracker in LWC |
| `Risk_Rating__c` | Picklist | Low, Medium, High, Critical | Underwriter-set |
| `Assigned_Underwriter__c` | Lookup(User) | Relationship: Assigned_Loan_Applications | — |
| `Broker__c` | Lookup(Account) | Relationship: Broker_Loan_Applications | Broker-editable |
| `AI_Processing_Status__c` | Picklist | Pending, In Progress, Completed, Failed | Set by Phase 2 AI |
| `AI_Summary__c` | Long Text Area(32768) | — | Hidden from Brokers |
| `AI_Risk_Alerts__c` | Long Text Area(32768) | — | Hidden from Brokers |
| `AI_Recommended_Action__c` | Text Area(255) | — | Visible to Underwriters |
| `Documents_Complete__c` | Checkbox | Default: false | — |
| `Missing_Documents__c` | Long Text Area(32768) | — | Displayed as alert banner in LWC |
| `Submission_Date__c` | DateTime | — | Set by flow on first document upload |
| `SLA_Due_Date__c` | Formula(DateTime) | `Submission_Date__c + 2` | Blanks treated as zero |
| `Predicted_Default_Risk__c` | Percent(16,2) | — | Hidden from Brokers |
| `Fraud_Risk_Score__c` | Percent(16,2) | — | Hidden from Brokers and Underwriters |
| `Estimated_Borrowing_Capacity__c` | Currency(16,2) | — | Set by Phase 2 AI |

#### Custom Fields — Loan_Document__c

| Field API Name | Type | Detail | Notes |
|---|---|---|---|
| `Loan_Application__c` | Master-Detail | Required; reparenting disabled | Links to parent application |
| `Document_Type__c` | Picklist | Payslip, Bank Statement, Tax Return, ID Document, BAS, Loan Form, Other | Set by LWC/Apex after upload |
| `Document_Status__c` | Picklist | Pending, Received (default), Verified, Rejected | Set to Received by flow on creation |
| `ContentDocument_Id__c` | Text(18) | External ID, Unique (case-sensitive) | Links to Salesforce Files; used by Apex to match records |
| `AI_Extracted_Data__c` | Long Text Area(131072) | — | Raw JSON from Einstein Document AI; hidden from Underwriters |
| `AI_Confidence_Score__c` | Percent(16,2) | — | Populated by Phase 2 AI |
| `Document_Date__c` | Date | — | — |
| `Anomaly_Detected__c` | Checkbox | Default: false | Set by Phase 2 AI |
| `Anomaly_Description__c` | Text Area(255) | — | Set by Phase 2 AI |

#### Custom Fields — Extracted_Financial_Data__c

| Field API Name | Type | Detail |
|---|---|---|
| `Loan_Application__c` | Lookup(Loan_Application__c) | Relationship: Extracted_Financials |
| `Document__c` | Lookup(Loan_Document__c) | Relationship: Extracted_Financials |
| `Employer_Name__c` | Text(255) | — |
| `Annual_Income__c` | Currency(16,2) | — |
| `PAYG_Status__c` | Checkbox | Default: false |
| `Monthly_Expenses__c` | Currency(16,2) | — |
| `Monthly_Liabilities__c` | Currency(16,2) | — |
| `Gambling_Indicators__c` | Checkbox | Default: false |
| `Applicant_DOB__c` | Date | — |
| `Address_Match__c` | Checkbox | Default: false |
| `Tax_Declared_Income__c` | Currency(16,2) | — |
| `Employment_Duration_Years__c` | Number(16,2) | — |

#### Custom Fields — Underwriting_Note__c

| Field API Name | Type | Detail |
|---|---|---|
| `Loan_Application__c` | Master-Detail(Loan_Application__c) | Required; relationship: Underwriting_Notes |
| `Note_Type__c` | Picklist | AI Generated, Manual, System |
| `Note_Content__c` | Long Text Area(32768) | — |
| `Author__c` | Lookup(User) | Relationship: Authored_Underwriting_Notes |
| `Is_AI_Generated__c` | Checkbox | Default: false; distinguishes AI vs manual notes |

#### Permission Sets

| Permission Set API Name | Label | Purpose |
|---|---|---|
| `TFS_Broker` | TFS Broker | Broker users — create and read loan applications and documents; sensitive AI fields hidden |
| `TFS_Underwriter` | TFS Underwriter | Underwriter users — read/edit all objects; raw JSON and fraud score hidden |

##### TFS_Broker — Object Permissions

| Object | Create | Read | Edit | Delete |
|---|---|---|---|---|
| `Loan_Application__c` | Yes | Yes | No | No |
| `Loan_Document__c` | Yes | Yes | No | No |
| `Extracted_Financial_Data__c` | No | Yes | No | No |
| `Underwriting_Note__c` | No | Yes | No | No |

##### TFS_Broker — Field-Level Security (Loan_Application__c)

| Field | Access |
|---|---|
| `AI_Risk_Alerts__c` | Hidden (no read, no edit) |
| `Fraud_Risk_Score__c` | Hidden (no read, no edit) |
| `Predicted_Default_Risk__c` | Hidden (no read, no edit) |
| `AI_Summary__c` | Hidden (no read, no edit) |
| `Applicant_Name__c`, `Loan_Amount__c`, `Loan_Purpose__c`, `Broker__c` | Read + Edit |
| All other custom fields | Read only |

##### TFS_Underwriter — Object Permissions

| Object | Create | Read | Edit | Delete |
|---|---|---|---|---|
| `Loan_Application__c` | No | Yes | Yes | No |
| `Loan_Document__c` | No | Yes | Yes | No |
| `Extracted_Financial_Data__c` | No | Yes | Yes | No |
| `Underwriting_Note__c` | Yes | Yes | Yes | No |

##### TFS_Underwriter — Field-Level Security

| Object | Field | Access |
|---|---|---|
| `Loan_Application__c` | `Fraud_Risk_Score__c` | Hidden (no read, no edit) |
| `Loan_Application__c` | `SLA_Due_Date__c` | Read only (formula — not editable) |
| `Loan_Application__c` | All other custom fields | Read + Edit |
| `Loan_Document__c` | `AI_Extracted_Data__c` | Hidden (no read, no edit) |
| `Loan_Document__c` | All other custom fields | Read + Edit |
| `Extracted_Financial_Data__c` | All custom fields | Read + Edit |
| `Underwriting_Note__c` | All custom fields | Read + Edit |

#### Flows

| Flow API Name | Label | Type | Trigger | Status |
|---|---|---|---|---|
| `TFS_ContentDocumentLink_LoanDoc_Create` | TFS ContentDocumentLink - Create Loan Document | AutoLaunchedFlow (Record-Triggered) | ContentDocumentLink After Insert | **Inactive** — must be activated manually after updating the key prefix |

---

### Development Components (Code)

#### Apex Classes

| Class API Name | Modifier | Description |
|---|---|---|
| `TFS_BrokerDocumentUploadController` | `public with sharing` | Controller for the brokerDocumentUpload LWC. Provides two AuraEnabled methods for document type assignment and document list retrieval. All SOQL/DML run with USER_MODE. |
| `TFS_BrokerDocumentUploadControllerTest` | `@isTest private` | Test class for TFS_BrokerDocumentUploadController. 10 test methods. Expected coverage ~93%. |

#### Apex Methods — TFS_BrokerDocumentUploadController

**setDocumentType**
```
@AuraEnabled
public static void setDocumentType(
    Id loanApplicationId,
    String contentDocumentId,
    String documentType
)
```
- Called by the LWC after each file upload completes.
- Queries `Loan_Document__c` WHERE `Loan_Application__c = loanApplicationId` AND `ContentDocument_Id__c = contentDocumentId` WITH USER_MODE LIMIT 1.
- Sets `Document_Type__c` to the broker's selected value.
- Uses `Database.update(..., AccessLevel.USER_MODE)` for DML.
- Throws `AuraHandledException` on null inputs, no-match, or DML/query failure.

**getUploadedDocuments**
```
@AuraEnabled(cacheable=true)
public static List<Loan_Document__c> getUploadedDocuments(Id loanApplicationId)
```
- Called via `@wire` by the LWC to populate the uploaded documents datatable.
- Returns `Id, Name, Document_Type__c, Document_Status__c, ContentDocument_Id__c, CreatedDate` ordered by `CreatedDate DESC`, LIMIT 200.
- Throws `AuraHandledException` on null input or query failure.

#### Test Class — TFS_BrokerDocumentUploadControllerTest

| Test Method | Scenario |
|---|---|
| `testGetUploadedDocuments_returnsDocuments` | Returns correct count of documents for an application |
| `testGetUploadedDocuments_emptyList` | Returns empty list (never null) when no documents exist |
| `testGetUploadedDocuments_bulkVolume` | Handles 50 documents without hitting governor limits |
| `testGetUploadedDocuments_nullId_throwsException` | Null loanApplicationId throws AuraHandledException |
| `testSetDocumentType_success` | Updates Document_Type__c to the specified value |
| `testSetDocumentType_nullInputs` | Null loanApplicationId throws AuraHandledException |
| `testSetDocumentType_nullContentDocumentId` | Null contentDocumentId throws AuraHandledException |
| `testSetDocumentType_blankDocumentType` | Blank documentType throws AuraHandledException |
| `testSetDocumentType_blankContentDocumentId` | Blank contentDocumentId throws AuraHandledException |
| `testSetDocumentType_noMatchingRecord` | Non-existent contentDocumentId throws AuraHandledException |
| `testSetDocumentType_targetsSingleRecord` | Only the targeted document is updated; sibling documents unchanged |

Test setup uses `@TestSetup` to create one `Loan_Application__c` and two `Loan_Document__c` records shared across applicable test methods.

#### Lightning Web Components

| Component Name | Master Label | API Version |
|---|---|---|
| `brokerDocumentUpload` | Broker Document Upload | 66.0 |

Component files:
- `brokerDocumentUpload.js` — controller logic, wire adapters, event handlers
- `brokerDocumentUpload.html` — five-section template (stage tracker, missing docs alert, type selector, file upload, documents table)
- `brokerDocumentUpload.css` — SLDS-based layout styles
- `brokerDocumentUpload.js-meta.xml` — targets and design property configuration

**LWC Public Property:**

| Property | Type | Source |
|---|---|---|
| `@api recordId` | String / Id | Auto-bound on record page; set via community page design attribute on Experience Cloud |

**LWC Sections (rendered in order):**

1. Stage Tracker — `lightning-progress-indicator` (path variant) showing `Application_Status__c` across 7 steps. Only renders when status has loaded.
2. Missing Documents Alert — Warning banner with `lightning-icon` (utility:warning) displaying `Missing_Documents__c` content. Only renders when the field is populated.
3. Document Type Combobox — `lightning-combobox` (required). Upload section is hidden until a type is selected.
4. File Upload — `lightning-file-upload` supporting `.pdf .png .jpg .jpeg .tiff`, multi-file, native drag-and-drop. Only renders when `canUpload` is true (type selected AND recordId present).
5. Uploaded Documents Table — `lightning-datatable` showing Name, Type, Status. Only renders when at least one document exists.

**Upload Finished Handler:**
- Uses `Promise.allSettled` to fire all `setDocumentType` Apex calls in parallel.
- Counts fulfilled vs rejected results.
- Dispatches a Success toast (all succeeded), Error toast (all failed), or Warning toast with counts (partial failure).
- Calls `refreshApex` on the wired documents result after each upload cycle.

**Wire Error Handling:**
- Wire errors on `getRecord` (application data) and `getUploadedDocuments` both dispatch Error toasts via `ShowToastEvent`.
- `reduceError` helper normalises `AuraHandledException` body, wired-error arrays, and plain JS errors into a single readable string.

**LWC Targets (js-meta.xml):**

| Target | Configuration |
|---|---|
| `lightning__RecordPage` | Restricted to `Loan_Application__c` object pages |
| `lightningCommunity__Page` | No additional configuration |
| `lightningCommunity__Default` | Design property: `recordId` (String) — "Loan Application ID" |

---

## Data Flow

### End-to-End Document Upload Flow

```
Broker selects Document Type in LWC combobox
           │
           ▼
Broker selects / drags files into lightning-file-upload
           │
           ▼ (Salesforce platform creates ContentVersion + ContentDocument)
ContentDocumentLink record created (file linked to Loan Application)
           │
           ▼
TFS_ContentDocumentLink_LoanDoc_Create flow fires (After Insert)
           │
           ├─ Decision: Is LinkedEntityId a Loan Application? (key prefix check)
           │  No → flow terminates (file was linked to a different object)
           │  Yes ↓
           ├─ Get Loan Application record
           ├─ Create Loan_Document__c (Status = Received, ContentDocument_Id__c set)
           └─ Is Application_Status__c = "New"?
              No → done
              Yes ↓
              ├─ Is Submission_Date__c blank?
              │  Yes → set Application_Status__c = "Documents Received"
              │         set Submission_Date__c = now
              │  No  → set Application_Status__c = "Documents Received" only
              └─ Update Loan Application record
           │
           ▼ (control returns to LWC onuploadfinished event)
LWC calls setDocumentType Apex for each uploaded file
           │
           ├─ Apex queries Loan_Document__c by ContentDocument_Id__c
           └─ Updates Document_Type__c with broker's selected value
           │
           ▼
LWC refreshes wired documents query → datatable updates
```

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│             Experience Cloud / Record Page               │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │           brokerDocumentUpload LWC                │   │
│  │                                                   │   │
│  │  [Stage Tracker]  [Missing Docs Alert]            │   │
│  │  [Document Type Combobox]                         │   │
│  │  [File Upload Area — drag or browse]              │   │
│  │  [Uploaded Documents Table]                       │   │
│  └──────────────┬────────────────────────────────────┘   │
└─────────────────┼────────────────────────────────────────┘
                  │ @wire / imperative Apex
                  ▼
┌─────────────────────────────────────────────────────────┐
│          TFS_BrokerDocumentUploadController             │
│                                                         │
│  getUploadedDocuments(loanApplicationId)  [cacheable]   │
│  setDocumentType(appId, contentDocId, type)             │
└──────────────────┬──────────────────────────────────────┘
                   │ SOQL/DML WITH USER_MODE
                   ▼
┌──────────────────────────────────────────────────────────┐
│                    Salesforce Data Layer                  │
│                                                          │
│  Loan_Application__c ◄── Loan_Document__c               │
│                      ◄── Underwriting_Note__c           │
│                      ◄── Extracted_Financial_Data__c    │
│                                                          │
│  ContentDocumentLink ─► TFS_ContentDocumentLink_        │
│                          LoanDoc_Create (Flow)          │
└──────────────────────────────────────────────────────────┘
```

---

## File Locations

| Component | Path |
|---|---|
| Loan_Application__c object | `force-app/main/default/objects/Loan_Application__c/` |
| Loan_Document__c object | `force-app/main/default/objects/Loan_Document__c/` |
| Extracted_Financial_Data__c object | `force-app/main/default/objects/Extracted_Financial_Data__c/` |
| Underwriting_Note__c object | `force-app/main/default/objects/Underwriting_Note__c/` |
| TFS_Broker permission set | `force-app/main/default/permissionsets/TFS_Broker.permissionset-meta.xml` |
| TFS_Underwriter permission set | `force-app/main/default/permissionsets/TFS_Underwriter.permissionset-meta.xml` |
| Apex controller | `force-app/main/default/classes/TFS_BrokerDocumentUploadController.cls` |
| Apex test class | `force-app/main/default/classes/TFS_BrokerDocumentUploadControllerTest.cls` |
| LWC component folder | `force-app/main/default/lwc/brokerDocumentUpload/` |
| Flow | `force-app/main/default/flows/TFS_ContentDocumentLink_LoanDoc_Create.flow-meta.xml` |
| Einstein runbook | `docs/einstein-document-ai-setup.md` |

---

## Security Model

### Summary Table

| User Type | Permission Set | Can Create | Can Read | Can Edit | Hidden Fields |
|---|---|---|---|---|---|
| Broker | TFS_Broker | Loan Applications, Loan Documents | All objects (limited fields) | Applicant_Name, Loan_Amount, Loan_Purpose, Broker | AI_Risk_Alerts, Fraud_Risk_Score, Predicted_Default_Risk, AI_Summary |
| Underwriter | TFS_Underwriter | Underwriting Notes only | All objects | All objects/fields (with exceptions) | Loan_Document.AI_Extracted_Data, Loan_Application.Fraud_Risk_Score |

### Apex Security

- Controller class declared `public with sharing` — sharing rules are enforced at runtime.
- All SOQL uses `WITH USER_MODE` — field-level security and object permissions are enforced at the query level (API 66.0 feature).
- All DML uses `AccessLevel.USER_MODE` via `Database.update(..., AccessLevel.USER_MODE)`.
- `AuraHandledException` is used for all user-facing errors so stack traces are never exposed.

### Org-Wide Defaults

OWD for the four custom objects was not explicitly set in Phase 1. The current default is ReadWrite (public). If the business requires Brokers to see only their own applications, OWD for `Loan_Application__c` should be changed to Private and criteria-based sharing rules added. This is flagged for confirmation before go-live.

---

## Post-Deployment Activation Checklist

Work through these steps after the Phase 1 metadata has been deployed to the target org.

### Step 1 — Confirm Loan_Application__c Key Prefix

The flow `TFS_ContentDocumentLink_LoanDoc_Create` contains a placeholder key prefix `"a0P"` in the formula `fx_IsLoanApplication`. The real key prefix is org-specific and must be updated before the flow is activated.

1. Open the target org.
2. Go to Setup > Object Manager > Loan Application.
3. In the URL, locate the 15-character object ID. The first 3 characters after the last `/` are the key prefix. Alternatively, run this in Anonymous Apex:
   ```apex
   System.debug(Loan_Application__c.SObjectType.getDescribe().getKeyPrefix());
   ```
4. Note the 3-character key prefix (e.g., `a0Q`).

### Step 2 — Update the Flow Formula

1. Go to Setup > Flows.
2. Open `TFS ContentDocumentLink - Create Loan Document`.
3. Click the `fx_IsLoanApplication` formula element.
4. Replace `"a0P"` with the actual key prefix found in Step 1:
   ```
   LEFT({!$Record.LinkedEntityId}, 3) = "a0Q"
   ```
5. Save the flow.

### Step 3 — Activate the Flow

1. In the flow canvas, click **Activate**.
2. Confirm activation.
3. Verify the flow status shows **Active** in Setup > Flows.

### Step 4 — Assign Permission Sets

Assign the appropriate permission set to each user before they access the platform:

```
Setup > Users > [select user] > Permission Set Assignments > Edit Assignments
```

- Brokers: assign `TFS_Broker`
- Underwriters: assign `TFS_Underwriter`

### Step 5 — Add LWC to Loan Application Record Page

1. Open a Loan Application record.
2. Click the gear icon > **Edit Page** (Lightning App Builder).
3. Locate `Broker Document Upload` in the component panel.
4. Drag it onto the page layout in the desired position.
5. Click **Save** and then **Activate**.
6. The `recordId` property is automatically bound from the record context — no additional configuration is required.

### Step 6 — Add LWC to Experience Cloud Broker Portal

1. Open Experience Builder for the broker portal.
2. Navigate to the page where brokers access loan applications.
3. Drag the `Broker Document Upload` component onto the page.
4. In the component properties panel, set **Loan Application ID** to the record ID of the relevant loan application (typically bound via a URL parameter or page variable in the community page template).
5. Save and publish the page.

### Step 7 — Einstein Document AI Setup (Optional — Phase 2 Prerequisite)

Follow the full runbook at `docs/einstein-document-ai-setup.md` to:
- Create and train the Payslip Extraction model.
- Create and train the Bank Statement Extraction model.
- Activate both models and record the Model IDs.

This step is not required for Phase 1 functionality but must be completed before Phase 2 AI integration work begins.

### Step 8 — Smoke Test

After completing steps 1-6:

1. Log in as a broker user (with TFS_Broker permission set).
2. Create a new Loan Application record. Confirm `Application_Status__c` defaults to "New".
3. Open the record page — confirm the `brokerDocumentUpload` LWC loads and shows the stage tracker at "New".
4. Select a Document Type (e.g., Payslip) and upload a test PDF.
5. Confirm the flow creates a `Loan_Document__c` record linked to the application.
6. Confirm `Application_Status__c` changes to "Documents Received".
7. Confirm `Submission_Date__c` is stamped.
8. Confirm the uploaded document appears in the LWC datatable with the correct type.
9. Log in as an underwriter user (with TFS_Underwriter permission set) and confirm the hidden fields (AI_Extracted_Data, Fraud_Risk_Score) are not visible.

---

## Testing

### Test Coverage Summary

| Class | Test Methods | Expected Coverage | Status |
|---|---|---|---|
| `TFS_BrokerDocumentUploadController` | 10 (in test class) | ~93% | Pending deployment |
| `TFS_BrokerDocumentUploadControllerTest` | N/A (is the test class) | 100% | Pending deployment |

### Key Test Scenarios Covered

| Category | Scenarios |
|---|---|
| Positive — retrieval | Returns correct document list for a valid application |
| Positive — update | Updates Document_Type__c on the correct record only |
| Negative — null inputs | Null loanApplicationId, null contentDocumentId, blank documentType, blank contentDocumentId each throw AuraHandledException |
| No-match | Non-existent contentDocumentId throws AuraHandledException with descriptive message |
| Isolation | Updating one document does not affect sibling documents on the same application |
| Bulk volume | 50 documents returned without exceeding governor limits |
| Empty state | Empty list returned (never null) when no documents exist |

---

## Known Limitations and Phase 2 Additions

### Known Limitations (Phase 1)

| Limitation | Detail |
|---|---|
| Flow key prefix is a placeholder | The `fx_IsLoanApplication` formula uses `"a0P"` as a placeholder. This MUST be updated to the real org key prefix before the flow is activated. See Post-Deployment Activation Checklist Step 2. |
| Document type set asynchronously | The flow creates the `Loan_Document__c` record, then the LWC calls Apex to set the Document_Type__c. There is a brief window where the type is blank. If the Apex call fails, the type will remain blank. |
| No push notifications | The missing documents banner in the LWC reads a static field value. There are no real-time push notifications, emails, or Chatter posts when documents are requested. |
| OWD is Public | Sharing is not restricted to record owners. Brokers can see each other's applications unless OWD is changed to Private and sharing rules are added. |
| No page layout configuration | Page layouts for the four objects use Salesforce defaults. Custom layouts are not part of Phase 1. |
| Einstein AI models not integrated | The Einstein Document AI runbook documents how to create the models, but Phase 1 does not call the API or populate AI fields. |
| getUploadedDocuments LIMIT 200 | The documents query returns at most 200 records. Applications with more than 200 documents will not show all uploads in the LWC. |

### What Phase 2 Will Add

| Feature | Description |
|---|---|
| Einstein Document AI integration | Apex service to call the activated Einstein models via API; populates AI_Extracted_Data__c and AI_Confidence_Score__c on Loan_Document__c |
| AI field population | Populates AI_Summary__c, AI_Risk_Alerts__c, AI_Recommended_Action__c, Predicted_Default_Risk__c, Estimated_Borrowing_Capacity__c on Loan_Application__c |
| Extracted_Financial_Data__c population | Structured financial data parsed from AI JSON output and written to the Extracted_Financial_Data__c child records |
| Agentforce Underwriting Assistant | Agentforce agent for underwriters to query the data and receive recommendations |
| Anomaly detection | Logic to set Anomaly_Detected__c and Anomaly_Description__c on Loan_Document__c |
| Data Cloud integration | Unified borrower profile, historical data enrichment, predictive scoring |
| Underwriter LWC | Dedicated component for underwriters showing AI outputs, risk scores, and recommended actions |

---

## Notes and Considerations

### Design Decisions

- **Permission sets over profiles** — Modern Salesforce best practice. Profiles are not modified by this project. Users need both their profile and the relevant permission set.
- **Flow over Apex trigger** — The ContentDocumentLink automation uses a Record-Triggered Flow rather than an Apex trigger, as requested. This keeps the logic declarative and visible in Flow Builder without requiring code deployment for minor changes.
- **lightning-file-upload over custom drag zone** — The standard base component provides multi-file, drag-and-drop, and accessibility compliance out of the box. A fully custom HTML5 drop zone was explicitly out of scope.
- **recordId as community design property** — The LWC uses `recordId` as both its auto-bound record page property and its community design attribute. Brokers must be on a page that has a Loan Application context (via URL parameter or page binding) when using Experience Cloud.
- **Promise.allSettled for upload handling** — The upload finished handler fires all Apex calls in parallel and waits for all to settle before reporting. This gives the broker an accurate count of successes vs failures rather than failing fast on the first error.

### Dependencies

| Dependency | Detail |
|---|---|
| Loan_Application__c must be deployed first | Loan_Document__c, Extracted_Financial_Data__c, and Underwriting_Note__c all reference it |
| Flow requires key prefix update | Cannot be activated with the placeholder value |
| Einstein models required for Phase 2 | Model IDs must be recorded from the runbook before Phase 2 development begins |
| Experience Cloud license | Required for the broker portal target of the LWC |

---

## Change History

| Date | Author | Description |
|---|---|---|
| 2026-05-24 | Documentation Agent | Initial creation — Phase 1 foundation complete |
