# Setup Guide

Complete step-by-step guide to set up the AI Content Generation Pipeline.

---

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Google Sheets Setup](#google-sheets-setup)
3. [n8n Configuration](#n8n-configuration)
4. [API Credentials](#api-credentials)
5. [Workflow Import](#workflow-import)
6. [Testing](#testing)
7. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Services
- **n8n** (v2.1.4 or higher)
  - Self-hosted: [Installation Guide](https://docs.n8n.io/hosting/)
  - Cloud: [n8n Cloud](https://n8n.io/cloud/)
- **Google Account** with Sheets API access
- **OpenAI API Key** ([Get one here](https://platform.openai.com/api-keys))

### Technical Requirements
- Basic understanding of n8n workflows
- Access to Google Sheets
- API key with sufficient credits

---

## Google Sheets Setup

### Step 1: Create New Spreadsheet

Create a Google Sheet named `content_input_v1` with **3 tabs**:

#### Tab 1: `input`
Headers (Row 1):

| A | B | C | D | E | F | G | H | I | J | K | L | M | N |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| id | topic | audience | language | length | validation_status | validation_notes | n8n_validation_status | n8n_validation_notes | qa_status | qa_message | content_generated_at | n8n_processed_at | processing_attempts |

**Column Descriptions:**
- `A-E`: Input data (manual entry)
- `F-G`: Excel formulas for manual validation
- `H-I`: Automated validation status from n8n
- `J-K`: QA results from n8n
- `L-M`: Timestamps
- `N`: Attempt counter (future use)

#### Tab 2: `output`
Headers (Row 1):

| A | B | C | D | E | F | G | H | I |
|---|---|---|---|---|---|---|---|---|
| id | language | length | topic | audience | title | body | qa_status | generated_at |

#### Tab 3: `logs`
Headers (Row 1):

| A | B | C | D | E | F |
|---|---|---|---|---|---|
| timestamp | id | event_type | step | status | message |

### Step 2: Add Validation Formulas (Optional)

In the `input` tab, you can add Excel formulas for manual validation:

**Cell F2 (validation_status):**
```excel
=IF(
  OR(
    A2="",
    B2="",
    C2="",
    NOT(OR(D2="EN", D2="ES")),
    NOT(OR(LOWER(E2)="short", LOWER(E2)="medium"))
  ),
  "error",
  "ok"
)
```

**Cell G2 (validation_notes):**
```excel
=IF(
  F2="ok",
  "",
  TEXTJOIN(
    " | ",
    TRUE,
    IF(A2="", "id missing", ""),
    IF(B2="", "topic missing", ""),
    IF(C2="", "audience missing", ""),
    IF(NOT(OR(D2="EN", D2="ES")), "invalid language (" & D2 & ")", ""),
    IF(NOT(OR(LOWER(E2)="short", LOWER(E2)="medium")), "invalid length (" & E2 & ")", "")
  )
)
```

Then drag these formulas down for all rows.

### Step 3: Share Sheet
- Get the Sheet ID from the URL
- Share with your n8n Google Service Account email
- Grant "Editor" permissions

---

## n8n Configuration

### Step 1: Install n8n (if not already)

**Option A: Docker**
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

**Option B: npm**
```bash
npm install n8n -g
n8n start
```

Access at: `http://localhost:5678`

### Step 2: Configure Credentials

#### Google Sheets Credential
1. In n8n, go to **Credentials** → **Add Credential**
2. Select **Google Sheets OAuth2 API**
3. Follow Google OAuth flow
4. Test connection

#### OpenAI API Credential
1. Go to **Credentials** → **Add Credential**
2. Select **OpenAI**
3. Enter your API key
4. Name it (e.g., "OpenAI GPT-4")

---

## Workflow Import

### Step 1: Download Workflow
- Download `workflows/content-generation-workflow.json` from this repo

### Step 2: Import to n8n
1. In n8n, click **Workflows** → **Import from File**
2. Select the downloaded JSON file
3. Click **Import**

### Step 3: Configure Nodes

**Update these nodes with your credentials:**

1. **Get row(s) in sheet**
   - Credential: Your Google Sheets credential
   - Document: Your spreadsheet ID or URL
   - Sheet: `input`

2. **All Google Sheets nodes**
   - Update Document reference to your spreadsheet

3. **Message a model**
   - Credential: Your OpenAI credential
   - Model: `gpt-4` or `gpt-4-turbo`

### Step 4: Update Sheet References
Find/Replace in the workflow:
- Find: `content_input_v1` (old sheet name)
- Replace: Your actual spreadsheet ID or name

---

## Testing

### Test 1: Validation Errors

Add this row to `input` sheet:

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| TEST01 | Sample topic | Test audience | FR | extra_long |

**Expected Result:**
- Validation fails (invalid language + invalid length)
- Entry in `logs` sheet with `event_type: validation_error`
- `n8n_validation_status: error` in `input` sheet

### Test 2: Successful Generation

Add this row:

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| TEST02 | Write a blog post about AI benefits | Tech enthusiasts | EN | medium |

**Expected Result:**
- Content generated
- QA passes
- Entry appears in `output` sheet
- Entry in `logs` with `event_type: qa_pass`
- `qa_status: qa_ok` in `input` sheet

### Test 3: QA Failure

Add this row:

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| TEST03 | Hi | Test | EN | medium |

**Expected Result:**
- Content generated (but likely too short)
- QA fails on word count
- Entry in `logs` with `event_type: qa_fail`
- `qa_status: qa_fail_length` in `input` sheet
- NO entry in `output` sheet

---

## Troubleshooting

### Issue 1: `#ERROR!` in Google Sheets

**Symptom:** Cells show `#ERROR!` or `#NAME?`

**Cause:** Google Sheets interpreting values as formulas

**Solution:** 
- In n8n Google Sheets nodes, go to **Options**
- Add option: **Cell Format**
- Select: **Let n8n format**

### Issue 2: No Output Data from Update Nodes

**Symptom:** "Update Row" nodes show "No output data returned"

**Solution:**
- In the node's **Settings** tab
- Enable: **Always Output Data**

### Issue 3: Workflow Stops at Loop

**Symptom:** Workflow executes only first item in loop

**Cause:** `n8n_processed_at` being set too early

**Solution:**
- Check Validation node code
- Ensure `n8n_processed_at` is NOT set there
- Only set in final Update nodes

### Issue 4: Context Loss Between Nodes

**Symptom:** Variables show `undefined` in later nodes

**Solution:**
- Use node references instead of `$json`:
```javascript
// Instead of:
{{ $json.id }}

// Use:
{{ $('QA node').item.json.id }}
```

### Issue 5: API Rate Limits

**Symptom:** Errors from OpenAI API about rate limits

**Solution:**
- Reduce Loop Batch Size to 1
- Add delay between iterations
- Upgrade API tier

---

## Environment Variables (Optional)

Create `.env` file for self-hosted n8n:

```env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=your_username
N8N_BASIC_AUTH_PASSWORD=your_password
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678/
```

---

## Next Steps

After successful setup:

1. **Populate Input Sheet** with real marketing briefs
2. **Execute Workflow** (manually or on schedule)
3. **Monitor Logs** for issues
4. **Review Output** for quality
5. **Iterate on Prompts** based on QA failures

---

## Support

If you encounter issues:

1. Check n8n execution logs
2. Review Google Sheets formulas
3. Verify API credentials are active
4. Check [Troubleshooting](#troubleshooting) section above

For additional help:
- Create an issue in this repo
- Contact: maumontenegrom@gmail.com

---

*Last updated: February 2026*
