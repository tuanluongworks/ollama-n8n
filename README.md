# AI Multi-Document Summarizer with n8n & Ollama

A privacy-first, cost-free workflow automation that reads multiple documents (.pdf, .docx, .md, .txt) from your local filesystem, merges their content, summarizes them using a locally-hosted AI model, and delivers results to Slack.

## Overview

This project demonstrates end-to-end multi-document summarization automation using:

- **n8n** - Open-source workflow automation platform
- **Ollama** - Local LLM inference engine (no API costs, full privacy)
- **Slack** - Result delivery via webhooks
- **Docker Compose** - One-command deployment

**Why This Stack?**

- **Privacy-first**: All AI processing happens on your infrastructure—no data leaves your network
- **Zero API costs**: After initial setup, unlimited summarizations at no per-request charge
- **Flexible**: Swap models, customize prompts, or change data sources in minutes
- **Multi-format support**: Automatically processes .txt, .pdf, .docx, .md files
- **Batch processing**: Handles multiple documents simultaneously and merges content
- **Production-ready foundation**: Add error handling, scheduling, or database storage

## Quick Start

### Prerequisites

- Docker & Docker Compose
- Slack workspace with an Incoming Webhook configured

### Step 1: Start Services

```bash
# Launch n8n and Ollama containers
docker-compose up -d

# Confirm both containers are running
docker ps
```

### Step 2: Download AI Model

```bash
# Pull the default lightweight model (3B parameters)
docker exec -it ollama ollama pull qwen2.5:3b-instruct

# Alternatives (larger, more capable):
# docker exec -it ollama ollama pull llama3
# docker exec -it ollama ollama pull mistral
```

### Step 3: Import Workflow

1. Open n8n at **http://localhost:5678**
2. Navigate to **Workflows → Import from File**
3. Select `workflows/ai-summarize-to-slack.json`
4. In the workflow editor, update the **Send to Slack** node with your webhook URL

### Step 4: Add Your Documents

```bash
# Place your documents in the n8n-local-files directory
cd n8n-local-files

# Add any combination of supported file types:
cp /path/to/document.pdf .
cp /path/to/report.docx .
cp /path/to/notes.md .
cp /path/to/data.txt .
```

### Step 5: Run Summarization

1. Click **Execute Workflow** in the n8n UI
2. The workflow will automatically:
   - Find all .pdf, .docx, .md, and .txt files
   - Extract text from each document
   - Merge all content together with file name headers
   - Send to Ollama for AI summarization
   - Post the combined summary to Slack

## Architecture

The workflow consists of 8 sequential nodes:

```
┌──────────────────┐
│ Manual Trigger   │  ← On-demand execution
└────────┬─────────┘
         ↓
┌──────────────────┐
│ List All Files   │  ← Execute: find /files -type f \( -name '*.pdf' -o ... \)
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Split File Paths │  ← JavaScript: parse find output into array of file paths
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Read Binary File │  ← Loops: reads each file as binary data
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Extract from File│  ← Auto-detects format and extracts text (PDF/DOCX/MD/TXT)
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Merge All Text   │  ← JavaScript: combines all extracted text with headers
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Prepare Request  │  ← JavaScript: builds Ollama API payload with merged content
└────────┬─────────┘
         ↓
┌──────────────────┐
│ Summarize with AI│  ← HTTP POST → http://ollama:11434/api/generate
└────────┬─────────┘     (20-minute timeout for large documents)
         ↓
┌──────────────────┐
│ Send to Slack    │  ← HTTP POST → Slack Webhook URL with file count
└──────────────────┘
```

**Data Flow:**
1. User triggers workflow manually
2. n8n finds all supported files in mounted volume using `find` command
3. File paths split into separate workflow items for parallel processing
4. Each file read as binary data
5. Text extracted based on file type (PDF/DOCX handled automatically)
6. All text content merged with file name separators
7. Ollama payload constructed with merged text and file count
8. HTTP request to Ollama on internal Docker network
9. AI response formatted with file count and posted to Slack

## Supported File Formats

| Format | Extension | Extraction Method | Notes |
|--------|-----------|-------------------|-------|
| Plain Text | `.txt` | Direct UTF-8 decode | Fastest processing |
| Markdown | `.md` | Direct UTF-8 decode | Preserves formatting |
| PDF | `.pdf` | Extract from File node | Supports text-based PDFs |
| Word Document | `.docx` | Extract from File node | Modern Office format |

**Note:** The workflow automatically detects file type and applies appropriate extraction. No manual configuration needed.

## Project Structure

```
.
├── workflows/
│   └── ai-summarize-to-slack.json    # 8-node n8n workflow (multi-file support)
├── n8n-local-files/
│   ├── funny-story.md                # Example: Markdown document
│   ├── meeting-notes.txt             # Example: Plain text document
│   ├── autocorrect-disaster.pdf      # Example: PDF document
│   └── tech-support-comedy.docx      # Example: Word document
├── docker-compose.yml                # n8n + Ollama orchestration
├── .gitignore                        # Excludes sensitive files
└── README.md                         # This file
```

## Configuration Guide

### Change the AI Model

Edit the **Prepare Ollama Request** node's JavaScript code:

```javascript
const jsonPayload = JSON.stringify({
  model: "llama3",  // Change this line
  prompt: `Please provide a concise summary of the following ${fileCount} document(s):\n\n${text}`,
  stream: false
});
```

**Available Models** (pull before using):
- `qwen2.5:3b-instruct` - Fast, lightweight (default)
- `llama3` - Better reasoning, slower
- `mistral` - Balanced performance
- `codellama` - Optimized for code summarization

### Customize the Summarization Prompt

In the **Prepare Ollama Request** node, modify the prompt string:

```javascript
prompt: `Summarize these ${fileCount} documents in 3 bullet points:\n\n${text}`
// Or:
prompt: `Extract key action items from these ${fileCount} documents:\n\n${text}`
// Or:
prompt: `Compare and contrast the main themes in these ${fileCount} documents:\n\n${text}`
```

### Change Input Directory

The workflow searches in `/files/` by default. To change this:

1. Update the **List All Files** node's command parameter:
   ```bash
   find /files -type f \( -name '*.pdf' -o -name '*.docx' -o -name '*.md' -o -name '*.txt' \) -print
   ```
2. Change `/files` to your preferred path (must be mounted in docker-compose.yml)

### Add More File Types

To support additional formats (e.g., `.html`, `.rtf`):

1. Update the **List All Files** node command:
   ```bash
   find /files -type f \( -name '*.pdf' -o -name '*.docx' -o -name '*.md' -o -name '*.txt' -o -name '*.html' \) -print
   ```
2. The **Extract from File** node will attempt automatic extraction

### Configure Slack Webhook

1. Create a Slack app at https://api.slack.com/apps
2. Enable **Incoming Webhooks** and select a channel
3. Copy the webhook URL (format: `https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXX`)
4. Paste into the **Send to Slack** node's `url` field

## Development Workflow

### Access Services

| Service | URL | Purpose |
|---------|-----|---------|
| n8n UI | http://localhost:5678 | Workflow editor and execution monitor |
| Ollama API | http://localhost:11434 | Direct API access for testing |

### View Container Logs

```bash
# Real-time n8n logs
docker logs -f n8n

# Real-time Ollama logs (shows model loading and inference)
docker logs -f ollama
```

### Test Ollama Independently

```bash
# Interactive chat with the model
docker exec -it ollama ollama run qwen2.5:3b-instruct
# Type your prompts, Ctrl+D to exit

# Direct API test (curl)
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:3b-instruct",
  "prompt": "Explain Docker Compose in one sentence.",
  "stream": false
}'
```

### Verify Volume Mount

```bash
# Check files in local directory
ls -la n8n-local-files/

# Check files visible inside n8n container
docker exec n8n ls -la /files/

# Test the find command used by workflow
docker exec n8n find /files -type f \( -name '*.pdf' -o -name '*.docx' -o -name '*.md' -o -name '*.txt' \) -print
```

### Export Modified Workflows

After editing workflows in the n8n UI:
1. Click **⋮** (three dots) → **Download**
2. Save to `workflows/ai-summarize-to-slack.json`
3. Commit changes with descriptive message

## Troubleshooting

### Problem: Workflow Execution Times Out

**Symptoms:** n8n shows "Execution timed out" error after 20 minutes.

**Root Cause:** Large documents or slow models exceed the HTTP request timeout.

**Solution:**
1. Open the **Summarize with AI** node settings
2. Increase **Timeout** from 1200000ms (20 min) to 3600000ms (60 min)
3. Save and re-execute

**Alternative:** Use a faster model like `qwen2.5:3b-instruct` instead of larger models.

### Problem: "Model Not Found" Error

**Symptoms:** Ollama returns 404 or "model 'xyz' not found".

**Root Cause:** Model not pulled to local Ollama instance.

**Solution:**
```bash
# Pull the exact model name specified in your workflow
docker exec -it ollama ollama pull qwen2.5:3b-instruct

# Verify available models
docker exec -it ollama ollama list
```

### Problem: No Files Found

**Symptoms:** Workflow completes but no files are processed, or "Split File Paths" returns empty array.

**Root Cause:** No matching files in `/files/` directory or volume mount issue.

**Solution:**
1. Verify files exist: `ls -la n8n-local-files/`
2. Check files in container: `docker exec n8n ls -la /files/`
3. Test find command: `docker exec n8n find /files -type f \( -name '*.pdf' -o -name '*.docx' -o -name '*.md' -o -name '*.txt' \) -print`
4. If mount is stale, restart: `docker-compose restart n8n`

### Problem: "Command Failed: find" Error

**Symptoms:** Error message shows "find: unrecognized: -printf" or similar.

**Root Cause:** n8n container uses BusyBox which has limited `find` command support.

**Solution:** The workflow has been updated to use `-print` instead of `-printf`. If you see this error, re-import the latest workflow JSON.

### Problem: PDF/DOCX Extraction Fails

**Symptoms:** Text files work but PDF/DOCX files return empty content.

**Root Cause:** Extract from File node may not support certain PDF/DOCX formats.

**Solution:**
1. Verify the PDF is text-based (not scanned images)
2. Check DOCX is modern format (.docx, not .doc)
3. For scanned PDFs, you'll need OCR (not included in this workflow)
4. Check n8n logs for specific extraction errors: `docker logs n8n`

### Problem: Slack Message Not Delivered

**Symptoms:** Workflow completes successfully but no Slack message appears.

**Debugging Steps:**
1. Check the **Send to Slack** node's execution output for HTTP errors (look for 4xx/5xx status codes)
2. Verify webhook URL is correct (test with `curl` from command line)
3. Confirm the Slack app has permission to post to the target channel
4. Check Slack app wasn't disabled or webhook revoked

**Common Fix:** Regenerate the webhook URL in Slack and update the n8n node.

### Problem: Ollama Container Can't Be Reached

**Symptoms:** n8n shows "ECONNREFUSED" or "getaddrinfo ENOTFOUND ollama".

**Root Cause:** Containers not on the same Docker network.

**Solution:**
1. Verify both containers use `n8n-network`: `docker network inspect n8n-network`
2. Confirm URL in workflow uses internal hostname: `http://ollama:11434` (not `localhost`)
3. Restart containers: `docker-compose restart`

## Extending the Workflow

### Add Error Handling

**Goal:** Retry failed summarizations and notify on persistent errors.

**Implementation:**
1. Add an **Error Trigger** node to catch failures
2. Configure it to trigger on errors from **Summarize with AI**
3. Add conditional logic: retry 3 times with exponential backoff
4. On final failure, send error details to Slack or email

### Filter Files by Date or Size

**Goal:** Only process files modified in the last 24 hours or larger than 1KB.

**Implementation:**
1. Update **List All Files** node command:
   ```bash
   find /files -type f \( -name '*.pdf' -o -name '*.docx' -o -name '*.md' -o -name '*.txt' \) -mtime -1 -size +1k -print
   ```
2. `-mtime -1` = modified in last 24 hours
3. `-size +1k` = larger than 1 kilobyte

### Store Summaries in a Database

**Goal:** Persist summaries for historical analysis and search.

**Implementation:**
1. Add a **Postgres** or **MySQL** node after **Summarize with AI**
2. Create schema: `id`, `file_names`, `file_count`, `merged_text`, `summary`, `created_at`
3. Insert each result into the database
4. Optionally build a search API on top

**Use Case:** Track summarization history, run analytics on common themes, build a knowledge base.

### Schedule Automatic Execution

**Goal:** Run summarization daily at a specific time.

**Implementation:**
1. Replace **Manual Trigger** with a **Cron** node
2. Configure schedule (e.g., `0 9 * * *` for daily at 9 AM)
3. Workflow will automatically process all files in directory

**Use Case:** Daily digest of documents dropped into a folder, automated report generation.

### Add File-by-File Summaries

**Goal:** Get individual summaries for each file plus a combined summary.

**Implementation:**
1. After **Extract from File**, add a branch that sends each file to Ollama
2. Store individual summaries in an array
3. After processing all files, also generate a "summary of summaries"
4. Send both individual and combined summaries to Slack

**Complexity:** Medium (requires workflow branching and aggregation)

### Support OCR for Scanned PDFs

**Goal:** Extract text from image-based PDFs.

**Implementation:**
1. Add a custom Docker image with Tesseract OCR
2. Detect if PDF extraction returns empty content
3. Run OCR on the PDF and extract text
4. Pass OCR text to Ollama

**Complexity:** High (requires custom Docker image and OCR configuration)

## Technology Stack

| Component | Version | Purpose | License |
|-----------|---------|---------|---------|
| n8n | latest (Docker) | Workflow orchestration | Sustainable Use License |
| Ollama | latest (Docker) | Local LLM inference | MIT |
| qwen2.5:3b-instruct | 3B params | Default AI model (Qwen family) | Apache 2.0 |
| Docker Compose | 3.8+ | Container orchestration | Apache 2.0 |
| Slack Webhook API | - | Notification delivery | Proprietary |

**System Requirements:**
- Docker: 20.10+
- RAM: 4GB minimum (8GB+ recommended for larger models)
- Disk: 5GB for Ollama models + n8n data
- CPU: Modern x86_64 or ARM64 processor

## Why Ollama Over Cloud LLMs?

**Comparison with OpenAI/Anthropic/Cohere:**

| Factor | Ollama (Local) | Cloud APIs |
|--------|----------------|------------|
| **Privacy** | Data never leaves your network | Data sent to third-party servers |
| **Cost** | Free after setup (electricity only) | $0.0015 - $0.06 per 1K tokens |
| **Latency** | <1s for small models (local) | 2-10s (network + processing) |
| **Offline** | Works without internet | Requires internet connection |
| **Customization** | Full control over models/prompts | API limits and policies |
| **Setup Complexity** | Medium (Docker + model pull) | Low (just API key) |
| **File Processing** | No file upload limits | File size limits apply |

**When to Use Cloud APIs:**
- You need state-of-the-art models (GPT-4, Claude 3)
- You process <10K documents/month (cost-effective at small scale)
- You prioritize setup speed over privacy

**When to Use Ollama:**
- Privacy is critical (healthcare, legal, internal docs)
- High volume usage (>100K documents/month)
- Offline operation required
- Experimenting with open-source models
- Processing sensitive or confidential documents

## Example Use Cases

### 1. Daily Meeting Notes Digest
- Team members drop meeting notes (.md, .txt, .docx) into shared folder
- Cron trigger runs daily at 6 PM
- Workflow summarizes all day's meetings
- Combined summary posted to #daily-digest Slack channel

### 2. Research Paper Analysis
- Upload multiple PDF research papers
- Workflow extracts text from all papers
- AI generates comparative analysis
- Results stored in database for searchability

### 3. Legal Document Review
- Privacy-critical: all processing stays on-premises
- Upload contracts, agreements, legal memos
- AI extracts key terms, obligations, dates
- Summary sent to legal team's private Slack channel

### 4. Customer Support Ticket Summarization
- Export support tickets as .txt files daily
- Workflow merges and summarizes common issues
- Management receives daily insight report
- Trends tracked over time in database

## Documentation

- **[local-n8n-apikey.md](local-n8n-apikey.md)** - JWT API key for programmatic workflow execution
- **[slack-url.md](slack-url.md)** - Slack webhook URL reference
- **[n8n Official Docs](https://docs.n8n.io)** - Comprehensive n8n documentation
- **[Ollama Documentation](https://ollama.com/docs)** - Model library and API reference
- **[Ollama Model Library](https://ollama.com/library)** - Browse available AI models
