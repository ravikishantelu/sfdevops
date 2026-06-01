# Einstein Document AI Setup Runbook

Phase 1 - Loan Origination Platform
Manual configuration checklist for Einstein Document Reader / Document AI models.

> Note: Einstein Document AI model configuration is performed entirely through the Salesforce Setup UI.
> There is no source metadata representation under force-app for these models.
> Complete these steps in the target org after Phase 1 metadata has been deployed.

---

## Prerequisites

- System Administrator profile or equivalent
- Einstein features enabled on the org (Einstein Platform licence or Financial Services Cloud with Einstein add-on)
- Sample training documents available (minimum 10-20 per model recommended by Salesforce)

---

## Step 1 - Navigate to Einstein Document AI in Setup

1. Click the **Setup** gear icon (top-right)
2. In the Quick Find box, type `Einstein`
3. Under the **Einstein** section, click **Einstein Document Reader** (may also appear as **Document AI** depending on org release)
4. If prompted, enable the feature and accept the terms of service
5. Confirm the feature is toggled **On**

---

## Step 2 - Create the "Payslip Extraction" Model

1. On the Einstein Document Reader page, click **New Model**
2. Enter the following details:
   - **Model Name**: `Payslip Extraction`
   - **Description**: Extracts structured payslip data for loan income verification
   - **Document Type**: Select or create `Payslip`
3. Click **Next**
4. In the **Upload Training Documents** step:
   - Upload a minimum of 10 sample payslip PDFs (real or anonymised)
   - Ensure samples cover varied layouts (different employers, payroll systems)
5. Click **Next** to proceed to the **Field Definition** step
6. Define the following extraction fields (add each as a custom field):

   | Field Label | Data Type | Notes |
   |---|---|---|
   | Employer | Text | Name of the employing organisation |
   | Pay Period | Text | e.g. "01/03/2025 - 31/03/2025" |
   | Gross Pay | Currency | Gross earnings for the period |
   | Net Pay | Currency | Take-home pay after tax/deductions |
   | YTD Gross | Currency | Year-to-date gross earnings |

7. Click **Save** (do not train yet - add all fields first)

---

## Step 3 - Create the "Bank Statement Extraction" Model

1. On the Einstein Document Reader page, click **New Model**
2. Enter the following details:
   - **Model Name**: `Bank Statement Extraction`
   - **Description**: Extracts key financial indicators from bank statements for serviceability assessment
   - **Document Type**: Select or create `Bank Statement`
3. Click **Next**
4. In the **Upload Training Documents** step:
   - Upload a minimum of 10 sample bank statement PDFs
   - Include statements from multiple banks where possible (ANZ, CBA, NAB, Westpac etc.)
   - Ensure samples span different date ranges and formats
5. Click **Next** to proceed to the **Field Definition** step
6. Define the following extraction fields:

   | Field Label | Data Type | Notes |
   |---|---|---|
   | Account Holder | Text | Full name on the account |
   | Statement Period | Text | e.g. "01/03/2025 - 31/03/2025" |
   | Opening Balance | Currency | Balance at start of period |
   | Closing Balance | Currency | Balance at end of period |
   | Transactions | Text (Long) | JSON or summary of transaction lines |

7. Click **Save**

---

## Step 4 - Train Both Models

Perform these steps for **each** model (Payslip Extraction, then Bank Statement Extraction):

1. Open the model from the Einstein Document Reader list
2. Click **Train Model**
3. Review the training data summary (document count, field coverage)
4. Click **Confirm and Train**
5. Wait for training to complete - this typically takes 5-30 minutes depending on document volume
6. Review the **Training Results** panel:
   - Check confidence scores per field (aim for >80%)
   - If scores are low, add more training documents and re-train before activating
7. Repeat for the second model

---

## Step 5 - Activate Both Models

For **each** trained model:

1. Open the trained model
2. Review the **Model Summary** (version, accuracy metrics)
3. Click **Activate**
4. Confirm activation in the dialog
5. The model status changes to **Active**

---

## Step 6 - Record Model IDs for Phase 2 Integration

After activation, capture the Model ID for each model. The Phase 2 Apex integration service will call these models via the Einstein Document AI API.

1. Open each active model
2. Copy the **Model ID** from the model detail page (format: a UUID string)
3. Record both IDs here and share with the development team:

   | Model | Model ID |
   |---|---|
   | Payslip Extraction | `[RECORD AFTER ACTIVATION]` |
   | Bank Statement Extraction | `[RECORD AFTER ACTIVATION]` |

4. Store the Model IDs as Custom Metadata or Custom Settings in the org for use by the Phase 2 integration (the developer agent will implement this in Phase 2)

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| Einstein Document Reader not visible in Setup | Confirm Einstein licences are provisioned; raise a Salesforce support case if missing |
| Training confidence below 80% | Add more varied training documents; ensure PDFs are text-based (not scanned images without OCR) |
| Model activation fails | Check org storage limits; ensure no duplicate active model names exist |
| Field extraction returns null | Review field definitions; confirm field names match document content patterns |

---

## Related Phase 2 Work

- Phase 2 will implement an Apex service class to call the Einstein Document AI API using the Model IDs recorded above
- The `AI_Extracted_Data__c` field on `Loan_Document__c` will receive the raw JSON output from the API
- The `AI_Confidence_Score__c` field will store the per-document confidence returned by the model
