# Complete Guide: Ingest File Attachments from Salesforce Objects to Data Cloud

## Overview
This guide covers ingesting Content Documents (file attachments) from ALL Salesforce objects into Data Cloud for search indexing and vector operations.

## Prerequisites

### 1. Required Permissions
You need to configure these permissions in Setup:

**Permission Set: Data Cloud Architect**
- Assign this permission set to users who will manage the ingestion

**Content Permissions:**
```
Setup → Permission Sets → Data Cloud Salesforce Connector → App Permissions → Content
✓ Query Non Vetoed Files
```
*If you can't find Content section, search "Manage Content Permission" in the search bar*

**Salesforce Files Settings:**
```
Setup → Salesforce Files → General Settings → Salesforce Files Settings
✓ Enable Files to be ingested into Data Cloud
```

**SSOT Package Version:**
```
Setup → Installed Packages
Verify SSOT version is 1.68 or higher
```

**Knowledge Article Support (if ingesting Knowledge attachments):**
```
Setup → Permission Sets → Data Cloud Salesforce Connector → App Permissions
✓ Allow View Knowledge
```

### 2. Required Editions
- Data Cloud must be enabled in your org
- See Data Cloud edition availability documentation

## How File Attachment Ingestion Works

Data Cloud uses three Data Model Objects (DMOs) to ingest file attachments:

1. **ContentDocument** - The document uploaded to Salesforce
2. **ContentVersion** - Specific version of the document (unstructured file content)
3. **ContentDocumentLink** - Links showing where the file is shared (users, groups, records, libraries)

### Important Behavior Note
- **ContentDocument** and **ContentDocumentLink** (structured data) ingest ALL existing records during initial ingestion
- **ContentVersion** (unstructured file content) ingests ONLY file versions created or updated AFTER you create the data stream

## Choose Your Scenario

### Scenario 1: New Setup (No Existing File Attachment Ingestion)
**What to do:**
1. Create three content data streams (see steps below)
2. Attach files to standard or custom Salesforce objects
3. Data Cloud automatically ingests them

### Scenario 2: Existing Salesforce Objects with File Attachments
**What to do:**
1. Create three content data streams (see steps below)
2. **Re-attach the files** to existing objects to trigger ingestion
   - This is required because ContentVersion only ingests files created/updated after stream creation

### Scenario 3: You Have Existing Content Data Streams (Created Before March 2025)
**What to do:**
1. **DELETE** the existing content data streams (they're incompatible)
2. Create the three new content data streams (see steps below)
3. Either:
   - Create new Salesforce objects with file attachments, OR
   - Re-attach existing files to their Salesforce objects

## Step-by-Step Setup Instructions

### Step 1: Install the Standard Content Bundle

```bash
# Using Data Cloud CLI (if available)
sf data cloud bundle install --bundle-name "Content Bundle" -o lendingapp
```

**OR via UI:**
1. Navigate to: **Data Cloud → Data Streams**
2. Click **New** → **Install Standard Bundle**
3. Select **Content Bundle**
4. Click **Install**

This deploys:
- ContentDocument DLO
- ContentVersion DLO
- ContentDocumentLink DLO
- Corresponding DMOs
- Pre-configured data streams

### Step 2: Verify Data Lake Objects

1. Navigate to: **Data Cloud → Data Lake Objects**
2. Verify these DLOs exist:
   - ✓ ContentDocument
   - ✓ ContentVersion
   - ✓ ContentDocumentLink

### Step 3: Create Data Streams (If Not Using Bundle)

If the bundle didn't create streams automatically, create them manually:

**Via UI:**
1. **Data Cloud → Data Streams → New**
2. Select **Salesforce CRM** connector
3. Create these three data streams:

**Stream 1: ContentDocument**
- Name: `ContentDocument_Stream`
- Source Object: `ContentDocument`
- Source Connection: Your Salesforce CRM connector

**Stream 2: ContentDocumentLink**
- Name: `ContentDocumentLink_Stream`
- Source Object: `ContentDocumentLink`
- Source Connection: Your Salesforce CRM connector

**Stream 3: ContentVersion**
- Name: `ContentVersion_Stream`
- Source Object: `ContentVersion`
- Source Connection: Your Salesforce CRM connector

**Important:** File attachment ingestion does NOT support full refresh or total replacement for ContentVersion data stream.

### Step 4: Configure Object Relationships

#### A. Disable Redundant ContentDocument Relationship

1. Navigate to: **Data Cloud → Data Model**
2. Search for: `ContentDocument` DMO
3. Click **Relationships**
4. **IF this relationship exists, DISABLE it:**
   ```
   Object: ContentDocument
   Field: ContentDocumentId
   Related Object: ContentDocumentVersion
   Related Field: ContentDocumentVersionId
   ```

#### B. Create ContentDocumentLink Relationships

For EACH Salesforce object you want to ingest attachments from (Case, Account, Contact, Opportunity, custom objects, etc.):

1. Navigate to: **Data Cloud → Data Model**
2. Search for: `ContentDocumentLink` DMO
3. Click **Relationships**
4. Create a relationship to each object DMO:

**Example for Case:**
```
Object: ContentDocumentLink
Field: LinkedEntityId (or RelatedEntityId)
Cardinality: ManyToOne
Related Object DMO: Case
Related Field: Id (Case primary key)
```

**Example for Account:**
```
Object: ContentDocumentLink
Field: LinkedEntityId
Cardinality: ManyToOne
Related Object DMO: Account
Related Field: Id (Account primary key)
```

**Example for Custom Object:**
```
Object: ContentDocumentLink
Field: LinkedEntityId
Cardinality: ManyToOne
Related Object DMO: YourCustomObject__c
Related Field: Id (Custom object primary key)
```

### Step 5: Create Data Streams for Parent Objects

For each Salesforce object that will have file attachments:

1. **Data Cloud → Data Streams → New**
2. Select **Salesforce CRM** connector
3. Choose the object (e.g., Case, Knowledge Article, Contact, custom objects)
4. Configure and activate the stream

**Examples:**
- `Case_Stream` for Case attachments
- `KnowledgeArticleVersion_Stream` for Knowledge attachments
- `Contact_Stream` for Contact attachments
- `CustomObject_Stream` for custom object attachments

**Note:** Even without these streams, ContentVersion DMO will still ingest attachments from ALL objects after the three content streams are created. These parent object streams are needed for relationship context and search indexing.

### Step 6: Test File Attachment Ingestion

1. **In Salesforce CRM:**
   - Navigate to a record (Case, Account, Contact, etc.)
   - Click **Upload Files** or **Attach File**
   - Upload a supported file format (PDF, DOCX, XLSX, etc.)

2. **Verify in Data Cloud:**
   ```bash
   # Query ContentDocument DLO
   sf data cloud query run --query "SELECT Id, Title, FileType, ContentSize FROM ContentDocument__dll ORDER BY CreatedDate DESC LIMIT 10" -o lendingapp
   
   # Query ContentVersion DLO
   sf data cloud query run --query "SELECT Id, Title, VersionNumber, ContentDocumentId FROM ContentVersion__dll ORDER BY CreatedDate DESC LIMIT 10" -o lendingapp
   
   # Query ContentDocumentLink DLO
   sf data cloud query run --query "SELECT Id, ContentDocumentId, LinkedEntityId FROM ContentDocumentLink__dll LIMIT 10" -o lendingapp
   ```

3. **Wait 10-15 minutes** for initial ingestion processing

### Step 7: Create Vector Search Index (Optional but Recommended)

To enable AI-powered search on file attachments:

1. **Data Cloud → Search Indexes → New**
2. Configure vector search index including:
   - ContentVersion DMO
   - Parent object DMOs (Case, Account, etc.)
3. Enable file content vectorization
4. Save and deploy

## Supported File Formats

Data Cloud supports these file attachment formats:
- PDF (.pdf)
- Microsoft Word (.doc, .docx)
- Microsoft Excel (.xls, .xlsx)
- Microsoft PowerPoint (.ppt, .pptx)
- Text files (.txt)
- Images (.jpg, .jpeg, .png, .gif)
- And more - see official documentation

## Important Notes

### For Existing Files
- Files attached BEFORE creating ContentVersion stream will NOT be ingested automatically
- You must **re-attach** existing files to trigger ingestion
- Alternatively, update the file (upload new version) to trigger ingestion

### Full Refresh Limitation
- ContentVersion data stream does NOT support full refresh or total replacement
- Use incremental ingestion only

### ContentVersion Behavior
- Ingests files from ALL Salesforce objects automatically
- No need to configure per-object, but parent object streams help with relationships

## Verification Checklist

- [ ] Data Cloud Architect permission set assigned
- [ ] "Query Non Vetoed Files" permission enabled
- [ ] "Enable Files to be ingested into Data Cloud" setting enabled
- [ ] SSOT package version 1.68 or higher installed
- [ ] Content Bundle installed or three content streams created
- [ ] ContentDocument, ContentVersion, ContentDocumentLink DLOs exist
- [ ] Redundant ContentDocument relationship disabled
- [ ] ContentDocumentLink relationships created for each parent object
- [ ] Parent object data streams created
- [ ] Test file uploaded and verified in Data Cloud
- [ ] Vector search index created (if needed)

## Troubleshooting

**Files not appearing in Data Cloud:**
- Check permission sets are assigned
- Verify "Enable Files to be ingested into Data Cloud" is ON
- For existing files, re-attach them to trigger ingestion
- Wait 10-15 minutes for processing
- Check data stream status for errors

**Relationship errors:**
- Verify ContentDocumentLink relationships exist for each parent object
- Ensure parent object data streams are active
- Check field names match exactly (LinkedEntityId vs RelatedEntityId)

**Search not working:**
- Create vector search index configuration
- Include ContentVersion and parent object DMOs
- Allow time for index building (30+ minutes for large datasets)

## CLI Commands Summary

```bash
# Verify org authentication
sf org display -o lendingapp

# List data streams
sf data cloud data-stream list -o lendingapp

# List DLOs
sf data cloud dlo list -o lendingapp

# Query ContentDocument
sf data cloud query run --query "SELECT * FROM ContentDocument__dll LIMIT 10" -o lendingapp

# Query ContentVersion
sf data cloud query run --query "SELECT * FROM ContentVersion__dll LIMIT 10" -o lendingapp

# Query ContentDocumentLink
sf data cloud query run --query "SELECT * FROM ContentDocumentLink__dll LIMIT 10" -o lendingapp

# Check data stream status
sf data cloud data-stream get --name "ContentVersion_Stream" -o lendingapp
```

## Next Steps After Setup

1. **Create calculated insights** on file metadata
2. **Build segments** based on file attachments
3. **Configure vector search** for AI-powered content discovery
4. **Set up activations** to downstream systems
5. **Create data actions** for automation workflows

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    Salesforce CRM Org                           │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Case    │  │ Account  │  │ Contact  │  │ Custom   │      │
│  │          │  │          │  │          │  │ Objects  │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │             │              │             │            │
│       └─────────────┴──────────────┴─────────────┘            │
│                          │                                     │
│                    File Attachments                            │
│                          │                                     │
│       ┌──────────────────┴──────────────────┐                 │
│       │                                     │                 │
│  ┌────▼──────────┐  ┌──────────────────┐  │                 │
│  │ ContentDocument│  │ContentDocumentLink│  │                 │
│  └────┬───────────┘  └──────┬───────────┘  │                 │
│       │                     │               │                 │
│  ┌────▼──────────┐          │               │                 │
│  │ContentVersion │          │               │                 │
│  └────┬──────────┘          │               │                 │
│       │                     │               │                 │
└───────┼─────────────────────┼───────────────┼─────────────────┘
        │                     │               │
        │   Data Cloud Connector              │
        │                     │               │
        ▼                     ▼               ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Data Cloud                               │
│                                                                 │
│  ┌────────────────┐  ┌───────────────────┐  ┌────────────────┐│
│  │ContentDocument │  │ContentDocumentLink│  │ContentVersion  ││
│  │     DLO        │  │       DLO         │  │     DLO        ││
│  └────────┬───────┘  └────────┬──────────┘  └────────┬───────┘│
│           │                   │                      │        │
│           ▼                   ▼                      ▼        │
│  ┌────────────────┐  ┌───────────────────┐  ┌────────────────┐│
│  │ContentDocument │  │ContentDocumentLink│  │ContentVersion  ││
│  │     DMO        │  │       DMO         │  │     DMO        ││
│  └────────────────┘  └───────────────────┘  └────────────────┘│
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            Vector Search Index                           │ │
│  │         (AI-Powered Content Discovery)                   │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

**Project:** Lendingapp
**Org Alias:** lendingapp
**Username:** ravikishantelu510@agentforce.com
**Instance:** https://orgfarm-63f8d138bb-dev-ed.develop.my.salesforce.com

**Created:** May 23, 2026
**Last Updated:** May 23, 2026