[EXAMPLES.md](https://github.com/user-attachments/files/25278650/EXAMPLES.md)
# Usage Examples

Real-world examples of using the AI Content Generation Pipeline.

---

## Example 1: Single Content Brief

### Input (Google Sheets - `input` tab)

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| BLOG001 | Benefits of AI in healthcare | Medical professionals | EN | medium |

### Execution
1. Execute workflow manually
2. Wait ~5 seconds for processing

### Expected Output

**Sheet: `output`**
| id | language | length | topic | audience | title | body | qa_status | generated_at |
|----|----------|--------|-------|----------|-------|------|-----------|--------------|
| BLOG001 | EN | medium | Benefits of AI... | Medical professionals | AI Transforms Healthcare: Five Key Benefits | *[~200 word article with 3 paragraphs]* | qa_ok | 2026-02-13... |

**Sheet: `logs`**
| timestamp | id | event_type | step | status | message |
|-----------|----|-----------| -----|--------|---------|
| 2026-02-13... | BLOG001 | qa_pass | qa | qa_ok | Content generated successfully |

**Sheet: `input` (updated)**
- `qa_status`: "qa_ok"
- `qa_message`: "ok"
- `n8n_processed_at`: "2026-02-13T..."

---

## Example 2: Batch Processing

### Input (10 briefs)

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| SOC001 | TikTok UGC script for fintech app | Gen Z | EN | short |
| SOC002 | Instagram caption for product launch | Millennials | ES | short |
| SOC003 | LinkedIn thought leadership post | B2B executives | EN | medium |
| ... | ... | ... | ... | ... |

### Execution
1. Execute workflow
2. Processing time: ~10-15 seconds per brief
3. Total: ~2-3 minutes for 10 briefs

### Results Breakdown

**Validation Phase:**
- 8 passed validation
- 2 failed (invalid language or length)

**Generation Phase:**
- 8 API calls made
- ~$0.04 total cost (assuming $0.005 per call)

**QA Phase:**
- 7 passed QA
- 1 failed (word count too low)

**Final Output:**
- 7 approved content pieces in `output` sheet
- 3 items need manual review (2 validation errors + 1 QA failure)

---

## Example 3: Validation Error Handling

### Input

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| ERR001 | Sample topic | Sample audience | FR | medium |

Note: FR (French) is not supported (only EN/ES allowed)

### Expected Behavior

**API Call:** ❌ NOT made (saves cost)

**Sheet: `logs`**
| timestamp | id | event_type | step | status | message |
|-----------|----|-----------| -----|--------|---------|
| 2026-02-13... | ERR001 | validation_error | validation | error | invalid language: FR |

**Sheet: `input` (updated)**
- `n8n_validation_status`: "error"
- `n8n_validation_notes`: "invalid language: FR"
- `n8n_processed_at`: "2026-02-13T..."
- `qa_status`: (empty - never reached QA)

**Sheet: `output`**
- No entry (validation failed)

---

## Example 4: QA Failure Scenario

### Input

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| QA001 | Hi | Test | EN | medium |

Note: Topic is too short - likely to generate short content

### Execution Flow

**Validation:** ✅ Passes (all required fields present, valid values)

**Generation:** ✅ Succeeds (API returns content)

**QA Check:** ❌ Fails
```
body words out of range (180-260), got 87
```

### Expected Output

**Sheet: `logs`**
| timestamp | id | event_type | step | status | message |
|-----------|----|-----------| -----|--------|---------|
| 2026-02-13... | QA001 | qa_fail | qa | qa_fail_length | body words out of range (180-260), got 87 |

**Sheet: `input` (updated)**
- `qa_status`: "qa_fail_length"
- `qa_message`: "body words out of range (180-260), got 87"
- `n8n_processed_at`: "2026-02-13T..."

**Sheet: `output`**
- No entry (QA failed)

**Action Required:** Manual review and regeneration with better topic

---

## Example 5: Multi-Language Campaign

### Input (Mixed Languages)

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| CAMP001 | Product launch announcement | US market | EN | medium |
| CAMP002 | Anuncio de lanzamiento de producto | Mercado LATAM | ES | medium |
| CAMP003 | Product benefits overview | US market | EN | short |
| CAMP004 | Resumen de beneficios del producto | Mercado LATAM | ES | short |

### Expected Output

All 4 pieces generated and saved to `output` sheet, each in their respective language.

**Use Case:** Launch the same product in multiple markets with localized content.

---

## Example 6: Content Variations

### Input (Same Topic, Different Lengths)

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| VAR001 | AI in customer service | Business owners | EN | short |
| VAR002 | AI in customer service | Business owners | EN | medium |

### Expected Output

**VAR001 (short):**
- Title: ~7 words
- Body: ~110 words (90-140 range)
- Tone: Concise, punchy

**VAR002 (medium):**
- Title: ~9 words
- Body: ~220 words (180-260 range)
- Tone: More detailed, comprehensive

**Use Case:** Create both social media (short) and blog post (medium) versions of same content.

---

## Example 7: Error Recovery Workflow

### Scenario: Retry Failed Items

**Step 1: Identify Failures**
Query `input` sheet:
```
Filter: qa_status = "qa_fail_length"
```

**Step 2: Fix Root Cause**
- If topic too vague → improve topic description
- If prompt issue → adjust system prompt

**Step 3: Clear Processing Flag**
```
Set n8n_processed_at = "" (empty)
```

**Step 4: Re-run Workflow**
- Only unprocessed items will be picked up
- Already successful items are skipped

**Step 5: Verify**
- Check if previously failed items now pass QA
- Review new entries in `logs` and `output`

---

## Example 8: Monitoring & Metrics

### Daily Metrics Query

**Google Sheets Query (logs tab):**
```sql
=QUERY(logs!A:F, 
  "SELECT C, COUNT(C) 
   WHERE A >= date '" & TEXT(TODAY(), "yyyy-mm-dd") & "' 
   GROUP BY C 
   LABEL COUNT(C) 'Count'"
)
```

**Example Output:**
| event_type | Count |
|-----------|-------|
| validation_error | 12 |
| qa_pass | 87 |
| qa_fail | 5 |

**Insights:**
- **Success Rate:** 87/92 = 94.6%
- **Validation Saved:** 12 API calls (~$0.06)
- **QA Failure Rate:** 5/92 = 5.4%

---

## Example 9: Scheduled Automation

### Setup
1. Replace "Manual Trigger" with "Schedule Trigger"
2. Configure: Run every weekday at 9 AM
3. Team adds briefs to `input` sheet throughout the day
4. Workflow processes all pending items next morning

### Benefits
- No manual execution needed
- Consistent processing schedule
- Team can batch-add briefs

---

## Example 10: Integration with CMS

### Workflow Extension
After successful QA:
1. Content saved to `output` sheet ✅
2. Webhook node calls CMS API
3. Content auto-published as draft
4. Editor receives notification for final review

### n8n Nodes to Add
- HTTP Request (Webhook to CMS)
- Email notification to editor
- Slack message to content team

---

## Tips & Best Practices

### Input Preparation
✅ **DO:**
- Be specific in topics
- Use clear, descriptive language
- Follow the exact format for language/length

❌ **DON'T:**
- Use vague topics ("Hi", "Test", single words)
- Mix languages in topic field
- Use unsupported languages/lengths

### Prompt Optimization
- If QA failures are high, adjust model prompt
- Add examples of good output
- Refine word count instructions

### Cost Management
- Validation saves ~20-30% of API costs
- Monitor `logs` for failure patterns
- Batch processing is more cost-effective

### Quality Control
- Review `qa_fail` items weekly
- Look for patterns (e.g., always too short)
- Adjust prompts based on failures

---

*Last updated: February 2026*
