# AI-Powered PO Document Processing Workflow

Automate the extraction of purchase order data from PDF documents using OCR and AI, then seamlessly sync structured data into your Supabase database. This n8n workflow eliminates manual data entry and ensures consistency across vendor orders.

**Table of Contents**
- [Features](#features)
- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [How It Works](#how-it-works)
- [Database Schema](#database-schema)
- [Workflow Nodes](#workflow-nodes)
- [Customization](#customization)
- [Troubleshooting](#troubleshooting)
- [Performance & Costs](#performance--costs)
- [Contributing](#contributing)

---

## Features

- **End-to-End Automation:** Process PDFs from any cloud storage (Dropbox, Google Drive, S3) without manual intervention
- **OCR + AI Extraction:** Mistral AI extracts document content; OpenAI structures data with schema enforcement
- **Smart Duplicate Detection:** Automatically creates or updates records based on PO number to prevent duplicates
- **Line Item Processing:** Splits multi-item orders and stores them in a separate table for detailed analysis
- **Data Quality:** Strict JSON schema validation ensures only clean data reaches your database
- **Error Handling:** Built-in error detection with detailed failure reasons
- **Ready for Integration:** Structured output feeds directly into invoicing, inventory, or ERP systems

---

## Quick Start

### 1-Minute Overview

```
PDF Input → Mistral OCR → OpenAI Extraction → Supabase Upsert
                ↓
          Structured JSON Schema
                ↓
          New PO or Update Existing
                ↓
          Automatic Line Item Split
```

Result: Your purchase orders go from PDF chaos to organized database records.

---

## Prerequisites

Before you begin, ensure you have:

### API Credentials
- **Mistral AI API Key** (for OCR)
  - Sign up: https://console.mistral.ai/
  - Cost: Varies by usage (typically $0.01–0.05 per page)

- **OpenAI API Key** (for structured extraction)
  - Sign up: https://platform.openai.com
  - Model: GPT-4o recommended (or GPT-4o-mini for lower cost)
  - Cost: ~$0.01–0.03 per PO

- **Supabase Project** (for database)
  - Sign up: https://supabase.com
  - Free tier includes 500MB storage

### Software & Access
- **n8n instance** (Cloud or self-hosted)
  - Access: https://n8n.cloud or your self-hosted URL
- **Git** (to clone this repository)
- Basic understanding of REST APIs and JSON

---

## Installation

### Step 1: Clone the Repository

```bash
git clone https://github.com/your-org/po-extraction-workflow.git
cd po-extraction-workflow
```

### Step 2: Import Workflow into n8n

1. Open your n8n instance
2. Navigate to **Workflows** → **Import**
3. Upload the `workflow.json` file from this repository
4. Click **Import**

Alternatively, use the n8n CLI:

```bash
n8n import:workflow --input=workflow.json
```

### Step 3: Create API Credentials in n8n

#### Mistral Cloud API Credential
1. In n8n, go to **Credentials** → **Create New**
2. Search for "Mistral Cloud API"
3. Enter your Mistral API key
4. Click **Save**

#### OpenAI API Credential
1. Go to **Credentials** → **Create New**
2. Search for "OpenAI API"
3. Enter your OpenAI API key
4. Click **Save**

#### Supabase API Credential
1. Go to **Credentials** → **Create New**
2. Search for "Supabase"
3. Enter your Supabase project URL and API key
4. Click **Save**

### Step 4: Set Up Supabase Tables

Run the following SQL in your Supabase dashboard (SQL Editor → New Query):

```sql
-- Create orders table
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  purchase_order_number TEXT UNIQUE NOT NULL,
  vendor_name TEXT,
  vendor_number TEXT,
  vendor_street TEXT,
  vendor_postal_code TEXT,
  vendor_city TEXT,
  vendor_vat_registration TEXT,
  contact_person TEXT,
  contact_phone TEXT,
  purchase_order_date DATE,
  delivery_date DATE,
  delivery_name TEXT,
  delivery_street TEXT,
  delivery_postal_code TEXT,
  delivery_city TEXT,
  terms_of_delivery TEXT,
  payment_method TEXT,
  payment_due_days INTEGER,
  payment_reference TEXT,
  from_invoice_date BOOLEAN DEFAULT false,
  currency TEXT DEFAULT 'EUR',
  subtotal_excl_tax NUMERIC,
  total_tax NUMERIC,
  total_amount NUMERIC,
  pdf_source_url TEXT,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);

-- Create order_items table
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  purchase_order_number TEXT NOT NULL REFERENCES orders(purchase_order_number),
  item_number TEXT,
  material_number TEXT,
  description TEXT,
  order_quantity NUMERIC,
  unit TEXT,
  ean_number TEXT,
  total_net_value NUMERIC,
  created_at TIMESTAMP DEFAULT now()
);

-- Create indexes for faster queries
CREATE INDEX idx_orders_po_number ON orders(purchase_order_number);
CREATE INDEX idx_order_items_po_number ON order_items(purchase_order_number);
```

### Step 5: Activate the Workflow

1. Open the imported workflow in n8n
2. Click the **Activate** toggle (top-right)
3. The form trigger will generate a webhook URL
4. Copy the webhook URL for testing

---

## Configuration

### Input Form Setup

The workflow triggers via an n8n form that accepts:

| Field | Type | Required | Example |
|-------|------|----------|---------|
| Document_Url | URL | Yes | `https://dropbox.com/s/abc123/PO-2024.pdf` |

### Updating AI Extraction Schema

Edit the **AI Agent** node to customize extracted fields:

1. Click the **AI Agent** node
2. Find the section marked `### OUTPUT SCHEMA`
3. Modify the JSON schema to add/remove fields

Example: Add project code and department

```json
"project_code": { "title": "Project_Code", "type": "string or null" },
"department": { "title": "Department", "type": "string or null" }
```

Then update the corresponding database columns and the upsert nodes accordingly.

### Adjusting AI Behavior

In the **AI Agent** prompt, you can tune:

- **Strictness:** Increase `additionalProperties: false` to reject unknown fields
- **Precision:** Modify extraction guidelines for specific vendor formats
- **Validation:** Add business rules (e.g., "reject orders > $100k without approval code")

---

## How It Works

### Workflow Execution Flow

```
1. USER SUBMITS FORM
   └─> Document_Url (e.g., Dropbox PDF link)

2. SET DATA URL
   └─> Store URL in variable for next step

3. MISTRAL OCR PROCESSING
   └─> Extract raw text + metadata from PDF
   └─> Output: Markdown-formatted document content

4. AI AGENT + OPENAI
   └─> Parse markdown using GPT-4o
   └─> Apply JSON schema extraction rules
   └─> Output: Structured JSON object

5. EDIT FIELDS
   └─> Restructure nested JSON for database insertion
   └─> Prepare vendor, order, and item data

6. DUPLICATE CHECK (GET A ROW)
   └─> Query Supabase for existing PO number
   └─> Result: Record found or not found

7. CONDITIONAL LOGIC (IF)
   └─> If NOT found → CREATE new record
   └─> If FOUND → UPDATE existing record

8. SPLIT LINE ITEMS
   └─> Extract order.items array
   └─> Convert each item into separate row

9. INSERT LINE ITEMS
   └─> Write each item to order_items table
   └─> Link to parent PO number

RESULT: Complete PO + line items in Supabase
```

### Data Flow Diagram

```
PDF (Cloud Storage)
       ↓
   Mistral OCR
       ↓
   Raw Text
       ↓
   OpenAI GPT-4o
       ↓
   Structured JSON
       ↓
   Supabase Upsert
    ↙        ↘
orders    order_items
```

---

## Database Schema

### Orders Table

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | UUID | No | Primary key |
| purchase_order_number | TEXT | No | Unique identifier for deduplication |
| vendor_name | TEXT | Yes | Extracted from "Vendor" section |
| vendor_number | TEXT | Yes | Internal vendor code |
| vendor_street | TEXT | Yes | Vendor address street |
| vendor_postal_code | TEXT | Yes | Vendor postal code |
| vendor_city | TEXT | Yes | Vendor city |
| vendor_vat_registration | TEXT | Yes | VAT/Tax ID |
| contact_person | TEXT | Yes | Vendor contact name |
| contact_phone | TEXT | Yes | Vendor phone number |
| purchase_order_date | DATE | Yes | PO creation date |
| delivery_date | DATE | Yes | Expected delivery date |
| delivery_name | TEXT | Yes | Recipient name |
| delivery_street | TEXT | Yes | Delivery address street |
| delivery_postal_code | TEXT | Yes | Delivery postal code |
| delivery_city | TEXT | Yes | Delivery city |
| terms_of_delivery | TEXT | Yes | Delivery terms (e.g., "FOB", "CIF") |
| payment_method | TEXT | Yes | Payment type (e.g., "Net 30", "CC") |
| payment_due_days | INTEGER | Yes | Days until payment due |
| payment_reference | TEXT | Yes | Reference number (e.g., invoice prefix) |
| from_invoice_date | BOOLEAN | Yes | Payment terms start from invoice date |
| currency | TEXT | Yes | Currency code (default: EUR) |
| subtotal_excl_tax | NUMERIC | Yes | Line items total before tax |
| total_tax | NUMERIC | Yes | Total tax amount |
| total_amount | NUMERIC | Yes | Grand total (subtotal + tax) |
| pdf_source_url | TEXT | Yes | Original PDF URL |
| created_at | TIMESTAMP | No | Record creation time |
| updated_at | TIMESTAMP | No | Last modified time |

### Order Items Table

| Column | Type | Nullable | Notes |
|--------|------|----------|-------|
| id | UUID | No | Primary key |
| purchase_order_number | TEXT | No | Foreign key to orders |
| item_number | TEXT | Yes | Line item number from PO |
| material_number | TEXT | Yes | SKU/Product code |
| description | TEXT | Yes | Item description |
| order_quantity | NUMERIC | Yes | Quantity ordered |
| unit | TEXT | Yes | Unit of measure (pcs, kg, etc.) |
| ean_number | TEXT | Yes | EAN/Barcode |
| total_net_value | NUMERIC | Yes | Line total (qty × unit price) |
| created_at | TIMESTAMP | No | Record creation time |

---

## Workflow Nodes

### Core Nodes Breakdown

| # | Node Name | Type | Purpose | Input | Output |
|---|-----------|------|---------|-------|--------|
| 1 | On form submission | Trigger | Webhook to receive PDF URL | HTML Form | `Document_Url` |
| 2 | Set Data URL | Variable | Store URL for reuse | `Document_Url` | Stored variable |
| 3 | Mistral DOC OCR2 | HTTP | Call Mistral OCR API | PDF URL | Markdown text |
| 4 | AI Agent | LLM Chain | Extract structured data | Markdown | JSON object |
| 5 | OpenAI Chat Model | LLM | Provide reasoning engine | User input | Structured output |
| 6 | Structured Output Parser | Parser | Validate JSON schema | JSON | Validated JSON |
| 7 | Edit Fields | Transform | Prepare for DB | Nested JSON | Flat structure |
| 8 | Get a row | Database | Query existing PO | PO number | Record or empty |
| 9 | If | Logic | Route create vs update | Query result | Boolean |
| 10 | Create a row | Database | Insert new PO | Prepared data | New record |
| 11 | Update a row | Database | Modify existing PO | Prepared data | Updated record |
| 12 | Split Out | Array | Expand line items | Items array | Individual items |
| 13 | Add item into table | Database | Insert line items | Split items | Item records |

---

## Customization

### Adding New Fields

**Example: Add Department & Cost Center**

1. Update Supabase schema:
   ```sql
   ALTER TABLE orders ADD COLUMN department TEXT;
   ALTER TABLE orders ADD COLUMN cost_center TEXT;
   ```

2. Edit AI Agent extraction prompt to include:
   ```json
   "department": { "type": "string or null", "description": "Department responsible for order" },
   "cost_center": { "type": "string or null", "description": "Cost center code" }
   ```

3. Update the "Create a row" node's field mapping:
   ```
   fieldId: "department"
   fieldValue: "={{ $json.output.department }}"
   ```

### Changing OCR Provider

Currently using Mistral OCR. To switch to another provider (e.g., AWS Textract, Google Vision):

1. Replace the **Mistral DOC OCR2 Process** node with your provider's HTTP request
2. Transform the output to match the markdown format expected by the AI Agent
3. Test extraction quality on 5+ sample PDFs

### Integrating with ERP/Accounting Software

Add a new node after line item processing:

```
Split Out (line items)
    ↓
Add item into table (Supabase)
    ↓
Send to ERP API (new node)
    ↓
Sync to invoice matching system
```

Example: Send to SAP, NetSuite, or QuickBooks via REST API.

### Error Notifications

Add a Slack or email notification on workflow failure:

1. Click "+" to add a node after AI extraction
2. Add **Slack** or **Email** node
3. Configure message with error details:
   ```
   Error: {{ $json.error.reason }}
   Document: {{ $json.pdf_source_url }}
   Time: {{ $now.toISO() }}
   ```

---

## Troubleshooting

### Common Issues & Solutions

#### Issue: "OCR extraction failed"
**Cause:** PDF is scanned image, corrupted, or password-protected
**Solution:**
- Verify PDF is readable in Adobe Reader or browser
- Try a different sample PDF to isolate the issue
- Check Mistral API rate limits (upgrade plan if needed)
- Ensure PDF file size < 25MB

**Test:** Use the provided sample: `https://pdfobject.com/pdf/sample.pdf`

---

#### Issue: "Invalid JSON output from AI Agent"
**Cause:** AI model unable to parse messy OCR text or unexpected format
**Solution:**
- Review the OCR markdown output in the execution trace
- Increase OpenAI `temperature` parameter (0.3 → 0.5) for flexibility
- Refine extraction prompt with more specific guidelines
- Add field examples to the schema

**Debug:** Check the exact markdown output in node execution logs

---

#### Issue: "Duplicate key error: purchase_order_number"
**Cause:** PO already exists but the If logic didn't trigger update
**Solution:**
- Verify the `Get a row` query is filtering by correct column
- Check that PO number format matches exactly (spacing, case)
- Ensure the If condition logic is correct: `isEmpty() == true`
- Manually update the record in Supabase to unblock

---

#### Issue: "Supabase table not found"
**Cause:** Table name mismatch or table doesn't exist
**Solution:**
- Verify table names: `orders`, `order_items` (case-sensitive)
- Run the SQL setup scripts from Configuration section
- Check Supabase credential permissions (should have INSERT, UPDATE)
- Review node field IDs against actual column names

---

#### Issue: "No line items extracted"
**Cause:** PDF doesn't contain itemized list or format not recognized
**Solution:**
- Check if PDF has a "line items" or "items" section
- Manually inspect OCR output (Mistral node logs)
- Update AI extraction schema with vendor-specific item parsing
- Test with known good PO that has clear line item structure

---

### Debugging Steps

1. **Enable detailed logging:**
   - In n8n, click the **Logs** icon during execution
   - View full request/response for each API call

2. **Test OCR independently:**
   - Use Mistral API directly via curl or Postman
   - Verify document extraction works before adding AI

3. **Validate JSON schema:**
   - Copy OCR output → paste into JSON validator
   - Ensure structure matches expected schema

4. **Check API quotas:**
   - Mistral: https://console.mistral.ai/usage
   - OpenAI: https://platform.openai.com/account/usage
   - Supabase: Dashboard → Logs & Monitoring

5. **Test with sample data:**
   - Keep a known-good test PDF always available
   - Use consistent test data for reproducibility

---

## Performance & Costs

### Execution Time

| Component | Time | Notes |
|-----------|------|-------|
| PDF Upload | 1–5 sec | Depends on file size & network |
| Mistral OCR | 10–30 sec | Per-page; scales with document length |
| OpenAI Extraction | 3–8 sec | Usually GPT-4o is faster than GPT-4 |
| Supabase Queries | 0.5–2 sec | Including duplicate check & upsert |
| **Total Per PO** | 15–45 sec | Typical case |

**For 100 orders/day:** ~1.5–3 hours of compute time per day (can run concurrently)

### API Costs (Approximate)

| Service | Cost Per PO | Monthly (500 POs) |
|---------|-------------|-------------------|
| Mistral OCR | $0.01–0.05 | $5–25 |
| OpenAI GPT-4o | $0.01–0.03 | $5–15 |
| Supabase (free tier) | Free | Free |
| n8n Cloud | $25–960 | Per plan |
| **Total** | ~$0.02–0.08 | ~$35–$50/mo |

**ROI Example:**
- Manual data entry: 5 min/PO × 50 POs/day = 250 min (~4 hours)
- Automation cost: $0.05/PO × 50 = $2.50/day
- Labor savings: ~4 hours/day × $25/hr = $100/day
- **Payback period: < 1 day** ✓

---

## Contributing

### How to Contribute

1. **Report issues:** Open a GitHub issue with execution logs and error details
2. **Suggest features:** Use GitHub Discussions for feature requests
3. **Submit improvements:**
   - Fork the repository
   - Create a feature branch: `git checkout -b feature/my-improvement`
   - Commit changes: `git commit -m "Add feature X"`
   - Push to branch: `git push origin feature/my-improvement`
   - Open a Pull Request

### Development Guidelines

- Test changes with 5+ sample PDFs before submitting PR
- Update this README if adding new fields or workflow steps
- Document any new API integrations or customizations
- Ensure backward compatibility with existing Supabase schema

---

## Support & Resources

### Documentation Links
- **n8n Docs:** https://docs.n8n.io
- **Mistral OCR API:** https://docs.mistral.ai/api/ocr/
- **OpenAI API:** https://platform.openai.com/docs
- **Supabase Docs:** https://supabase.com/docs
- **JSON Schema Guide:** https://json-schema.org/

### Community & Help
- **n8n Slack:** https://n8n.io/slack
- **GitHub Issues:** Open an issue in this repository
- **Email Support:** your-email@company.com

### Related Workflows
- Invoice Processing Pipeline (GitHub link)
- Vendor Master Data Sync (GitHub link)
- Purchase Requisition Router (GitHub link)

---

## License

This workflow is provided as-is for commercial and non-commercial use. Attribution appreciated but not required.

---

## Changelog

### Version 1.0.0 (Initial Release)
- OCR extraction via Mistral AI
- Structured data via OpenAI GPT-4o
- Supabase integration with upsert logic
- Line item splitting and storage
- Comprehensive error handling

### Planned Features
- Batch PDF processing with queue triggers
- Webhook integrations (Slack, Teams notifications)
- Dashboard for monitoring extraction quality
- Multi-language support for international vendors
- Custom field mapping UI

---

**Last Updated:** 2025-01-20  
**Maintained by:** [Your Team/Name]  
**Status:** Active & Tested ✓