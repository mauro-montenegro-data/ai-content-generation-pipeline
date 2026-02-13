# AI Content Generation Pipeline with Quality Assurance

> **Automated content generation workflow** with multi-level validation, quality assurance, and comprehensive logging built with n8n, OpenAI API, and Google Sheets.

[![n8n](https://img.shields.io/badge/n8n-2.1.4-orange)](https://n8n.io/)
[![OpenAI API](https://img.shields.io/badge/OpenAI-ChatGPT--4-blue)](https://openai.com/)
[![Google Sheets](https://img.shields.io/badge/Google-Sheets%20API-green)](https://developers.google.com/sheets/api)

---

## 📋 Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [System Flow](#system-flow)
- [Technical Highlights](#technical-highlights)
- [Results & Metrics](#results--metrics)
- [Setup & Installation](#setup--installation)
- [Usage](#usage)
- [Lessons Learned](#lessons-learned)
- [Future Improvements](#future-improvements)
- [Contact](#contact)

---

## 🎯 Overview

This project demonstrates a **production-ready content generation pipeline** that processes marketing briefs, generates content using OpenAI GPT-4, validates quality, and maintains comprehensive audit logs.

**Problem Solved:** Marketing teams need to generate high-volume, quality-controlled content at scale while maintaining brand standards and tracking performance.

**Solution:** An automated pipeline that:
- Validates inputs before processing
- Generates content using OpenAI's GPT-4
- Performs automated quality checks
- Logs all events for observability
- Only outputs approved content

---

## ✨ Key Features

### 🔍 **Multi-Level Validation**
- **Input Validation**: Checks data integrity before API calls (saves costs)
- **QA Validation**: Verifies generated content meets specifications
- **Quality Gates**: Only approved content reaches the output

### 📊 **Comprehensive Logging**
- Event-driven logging system
- Tracks validation errors, QA passes/fails
- Full audit trail for debugging and metrics

### 🎛️ **Three-Sheet Architecture**
- **Input Sheet**: Content briefs and processing status
- **Output Sheet**: Only quality-approved content
- **Logs Sheet**: Complete event history

### 🔄 **Error Handling & Recovery**
- Graceful failure handling at each stage
- Detailed error messages for debugging
- Automatic marking of failed items for retry

### 🚀 **Scalable Design**
- Batch processing with configurable size
- Parallel execution support
- Rate limit management

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      INPUT (Google Sheets)                   │
│  Marketing briefs with: id, topic, audience, language, etc.  │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                   VALIDATION NODE (Code)                     │
│  • Validates required fields                                 │
│  • Checks language & length constraints                      │
│  • Normalizes data                                           │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
  ✅ VALID                 ❌ INVALID
        │                       │
        │                       ▼
        │           ┌───────────────────────┐
        │           │  LOG + UPDATE INPUT   │
        │           │  (validation_error)   │
        │           └───────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│                    LOOP OVER ITEMS                           │
│  Process each valid brief one by one                         │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│              CONTENT GENERATION (OpenAI GPT-4)                 │
│  • Generates title + body based on brief                     │
│  • Follows strict JSON schema                                │
│  • Includes length/quality instructions                      │
└───────────────────┬─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│                      QA NODE (Code)                          │
│  • Validates JSON schema                                     │
│  • Checks word count ranges                                  │
│  • Verifies paragraph structure                              │
│  • Confirms metadata matches                                 │
└───────────────────┬─────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
   ✅ QA PASS             ❌ QA FAIL
        │                       │
        ▼                       ▼
┌──────────────┐      ┌──────────────────┐
│ SAVE TO      │      │  LOG + UPDATE    │
│ OUTPUT       │      │  (qa_fail)       │
│              │      └──────────────────┘
│ LOG + UPDATE │
│ (qa_pass)    │
└──────────────┘
```

---

## 🔄 System Flow

### Stage 1: Input Validation
**Goal:** Catch errors before expensive API calls

**Validation Rules:**
- Required fields: `id`, `topic`, `audience`, `language`, `length`
- Allowed languages: `EN`, `ES`
- Allowed lengths: `short` (90-140 words), `medium` (180-260 words)

**Outcomes:**
- ✅ Valid → Proceed to generation
- ❌ Invalid → Log error + mark in input sheet

---

### Stage 2: Content Generation
**Goal:** Generate high-quality marketing content

**Process:**
1. Add metadata field to payload
2. Call OpenAI GPT-4 with structured prompt
3. Receive JSON response with `id`, `language`, `length`, `title`, `body`

**Prompt Engineering:**
- Explicit word count requirements
- Strict JSON schema enforcement
- Quality guidelines (paragraph structure, tone)

---

### Stage 3: Quality Assurance
**Goal:** Ensure generated content meets specifications

**QA Checks:**
- ✅ Schema validation (correct JSON structure)
- ✅ Metadata match (id, language, length match input)
- ✅ Title length (5-12 words)
- ✅ Body word count (within specified range)
- ✅ Paragraph structure (separated by blank lines)

**Outcomes:**
- ✅ QA Pass → Save to output + log success
- ❌ QA Fail → Log failure + mark for review

---

## 💡 Technical Highlights

### 1️⃣ **Evolution from Visual to Code**

**Initial Approach:** Used n8n's visual IF nodes for each validation rule
- ✅ Easy to understand
- ❌ Quickly became unmaintainable (10+ IF nodes)
- ❌ Hard to debug

**Final Approach:** Consolidated into a single JavaScript validation node
- ✅ Maintainable (one place to edit rules)
- ✅ Easy to test and debug
- ✅ Scales to any number of validation rules

**Learning:** Sometimes code is clearer than visual workflows.

---

### 2️⃣ **Google Sheets Formula Conflict Resolution**

**Challenge:** When writing data to Google Sheets via API, values like `"602"` or `"invalid length: extra_long"` were being interpreted as formulas (`=602`, `=invalid length: extra_long`), causing `#ERROR!`.

**Solutions Tried:**
1. ❌ Adding quotes: `"'" + value` → Created `='602` (still a formula)
2. ❌ Adding spaces: `" " + value` → Inconsistent formatting
3. ❌ `String()` wrapper → Still interpreted as formula

**Final Solution:** 
- Use n8n's **"Cell Format: Let n8n format"** option
- This tells n8n to send data with proper type information
- Google Sheets receives it as text, not formula

**Learning:** API integrations require understanding how both systems handle data types.

---

### 3️⃣ **Context Loss Between Nodes**

**Challenge:** After the "Append to output" node, the next node couldn't access original data (`$json.id`, `$json.qa_status` were `undefined`).

**Root Cause:** "Append to output" node using "Auto-map" mode only passes the data it wrote to the sheet, not the original input.

**Solution:** Reference earlier nodes directly:
```javascript
// Instead of: {{ $json.id }}
// Use: {{ $('QA node').item.json.id }}
```

**Learning:** n8n nodes pass their output, not the original input. Use node references when needed.

---

### 4️⃣ **Batch Processing Strategy**

**Design Decision:** Use `Batch Size = 1` in Loop Over Items

**Reasoning:**
- ✅ Easier to debug (one item at a time)
- ✅ Avoids API rate limits
- ✅ Clear error isolation
- ✅ Better for long-running workflows

**Trade-off:** Slower than parallel processing, but more reliable and debuggable.

---

## 📈 Results & Metrics

### Performance (Sample Run with 11 briefs):

| Metric | Count | Percentage |
|--------|-------|------------|
| **Total Briefs Processed** | 11 | 100% |
| **Validation Errors** | 3 | 27% |
| **Content Generated** | 8 | 73% |
| **QA Pass** | 7 | 88% of generated |
| **QA Fail** | 1 | 12% of generated |
| **Final Output** | 7 | 64% end-to-end success |

### Error Breakdown:
- `invalid length: extra_long` (1 brief)
- `invalid language: FR` (1 brief)  
- `missing topic` (1 brief)
- `body words out of range` (1 brief)

### Key Insights:
- **27% of briefs had input errors** → Validation saved 3 unnecessary API calls
- **88% of generated content passed QA** → High model reliability
- **12% QA failure rate** → Room for prompt optimization

---
🔐 Security & Cost Considerations

### Security Best Practices
- **API Keys**: Stored securely in n8n credentials (encrypted), never hardcoded in workflows
- **Data Privacy**: Google Sheets may contain PII - ensure proper access controls
- **Credential Rotation**: Regularly rotate API keys and service account credentials
- **Access Control**: Limit Google Sheets sharing to authorized users only

### Cost Optimization
- **Pre-validation**: Input validation prevents ~27% of unnecessary API calls
- **Efficient Prompting**: Structured prompts reduce token usage per request
- **Batch Processing**: Sequential processing avoids rate limit penalties
- **Quality Gates**: QA system prevents regeneration of poor content

### Estimated Costs (based on test run)
- **Per Brief**: ~$0.005 - $0.01 (GPT-4)
- **100 Briefs/day**: ~$0.50 - $1.00
- **Validation Savings**: ~$0.15/day (prevented API calls)

**Note:** Actual costs vary based on GPT-4 pricing tier and content length.

---

## 🚀 Setup & Installation

### Prerequisites
- n8n instance (self-hosted or cloud)
- Google Sheets API credentials
- OpenAI API (GPT-4) 

### Installation Steps

1. **Clone this repository**
```bash
git clone https://github.com/mauro-montenegro-data/ai-content-qa-pipeline.git
cd ai-content-qa-pipeline
```

2. **Import workflow to n8n**
- Open n8n
- Go to Workflows → Import from File
- Select `workflows/content-generation-workflow.json`

3. **Configure Google Sheets**
- Create a Google Sheet with 3 tabs: `input`, `output`, `logs`
- Set up headers (see `docs/SETUP.md` for details)
- Connect Google Sheets credential in n8n

4. **Configure OpenAI API**
- Add OpenAI credential in n8n
- Select model: `gpt-4` or `gpt-4-turbo`

5. **Test the workflow**
- Add sample data to `input` sheet
- Execute workflow
- Verify results in `output` and `logs` sheets

For detailed setup instructions, see [`docs/SETUP.md`](docs/SETUP.md).

---

## 📖 Usage

### Input Sheet Format

| id | topic | audience | language | length |
|----|-------|----------|----------|--------|
| 601 | UGC creator brief for TikTok... | Prospects in LATAM | ES | medium |
| 602 | TikTok script: creativity + performance | Prospects in LATAM | ES | short |

### Running the Workflow

1. Add new briefs to the `input` sheet
2. Execute the n8n workflow (manual or scheduled)
3. Check `output` sheet for approved content
4. Review `logs` sheet for full execution history
5. Check `input` sheet for processing status

### Monitoring

- **Input Sheet**: Check `qa_status`, `qa_message`, `n8n_processed_at` columns
- **Logs Sheet**: Filter by `event_type` to see validation_error, qa_pass, qa_fail
- **Output Sheet**: Only contains quality-approved content ready for use

---

## 🎓 Lessons Learned

### 1. **Start Simple, Refactor Smart**
Began with visual IF nodes (easy to understand), migrated to code when complexity grew. Don't over-engineer from the start.

### 2. **Validation Early, Save Money**
Catching errors before API calls saved ~27% of potential API costs in this test run.

### 3. **Logging is Non-Negotiable**
Without comprehensive logs, debugging failures would have been nearly impossible. Event-driven logging from day one.

### 4. **Test Edge Cases**
- Empty fields
- Invalid enums (wrong language, wrong length)
- Boundary conditions (exactly 90 words, exactly 260 words)
- API errors and timeouts

### 5. **Documentation as You Build**
Writing this README forced me to understand my own architecture better. Document while context is fresh.

---

## 🔮 Future Improvements

### Short-term
- [ ] Add retry logic for QA failures (regenerate with refined prompt)
- [ ] Implement rate limiting checks before API calls
- [ ] Add email notifications for batch completion
- [ ] Create dashboard visualization of metrics

### Medium-term
- [ ] Support for multiple AI providers (Antropic, Gemini, etc.)
- [ ] A/B testing framework for different prompts
- [ ] Automatic prompt optimization based on QA failure patterns
- [ ] Cost tracking per brief

### Long-term
- [ ] Web UI for non-technical users
- [ ] Real-time collaboration features
- [ ] Multi-language support expansion
- [ ] Integration with CMS platforms

---

## 📞 Contact

**Mauro Montenegro**
- 💼 LinkedIn: [linkedin.com/in/mauro-montenegro-data](https://www.linkedin.com/in/mauro-montenegro-data/)
- 📧 Email: maumontenegrom@gmail.com
- 🐙 GitHub: https://github.com/mauro-montenegro-data

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

## 🙏 Acknowledgments

- n8n community for excellent documentation
- OpenAI for their GPT-4 API
- Everyone who helped debug the Google Sheets formula issue 😅

---

**⭐ If this project helped you, please consider giving it a star!**

---

*Built with ☕ and late nights by Mauro Montenegro*
