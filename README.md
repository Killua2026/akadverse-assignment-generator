# AkadVerse Assignment Generator
### Tier 5 Learning AI Tool | Microservice Port: `8008`

> A faculty-facing AI tool that generates a complete, professionally
> formatted assignment package from course context in a single API call.
> Produces three documents simultaneously: an assignment brief, a marking
> rubric, and a marking scheme. All three are saved locally as `.docx` files
> and optionally synced to Google Drive as native editable Google Docs.

---

## Table of Contents

1. [What This Microservice Does](#what-this-microservice-does)
2. [The Three Output Documents](#the-three-output-documents)
3. [Architecture Overview](#architecture-overview)
4. [Prerequisites](#prerequisites)
5. [Getting Your API Key](#getting-your-api-key)
6. [Installation](#installation)
7. [Running the Server](#running-the-server)
8. [API Endpoints](#api-endpoints)
   - [POST /generate-assignment](#1-post-generate-assignment)
   - [GET /assignments](#2-get-assignments)
   - [GET /health](#3-get-health)
9. [Testing with Swagger UI](#testing-with-swagger-ui)
10. [Example Test Inputs](#example-test-inputs)
11. [Understanding the Responses](#understanding-the-responses)
12. [Enabling Google Docs Sync](#enabling-google-docs-sync)
13. [Generated Files](#generated-files)
14. [Common Errors and Fixes](#common-errors-and-fixes)
15. [Project Structure](#project-structure)

---

## What This Microservice Does

This service is a **Tier 5 component** of the AkadVerse AI-first e-learning
platform. It lives inside the *My Teaching* module and serves as an
intelligent assistant for lecturers and faculty.

A lecturer provides their course title, topic, academic level, total marks,
and learning outcomes. In a single API call, Gemini generates three complete,
consistent academic documents that can be handed directly to students or
used for grading -- no manual editing required.

All three documents are:
- Saved locally as formatted `.docx` files (simulating Google Cloud Storage)
- Logged to a SQLite database with metadata (simulating PostgreSQL)
- Optionally uploaded to Google Drive as native editable Google Docs, placed
  inside `/AkadVerse/2026/Assignments/` in the connected Google account

---

## The Three Output Documents

### 1. Assignment Brief (student-facing)
The document given to students. Contains the assignment title, course
details, marks and weighting, background context, learning outcomes being
assessed, numbered task list, format requirements, submission instructions,
and an academic integrity notice.

### 2. Marking Rubric
A criteria grid with four grade band columns: Distinction (70-100%),
Merit (60-69%), Pass (40-59%), and Fail (0-39%). Each row is one
assessment criterion with descriptors for each band and the marks
available for that criterion. Marks across all criteria sum to the
total specified by the lecturer.

### 3. Marking Scheme
Model answers and mark allocations for every task in the brief. Includes
specific guidance for markers: what earns full marks, what earns partial
marks, common student errors to watch out for, and notes on borderline cases.

> All three documents are internally consistent -- the total marks, task
> descriptions, and criteria match across all three. This is enforced by
> generating them together in a single structured API call rather than
> three separate requests.

---

## Architecture Overview

```
Lecturer input (form fields)
        │
        ▼
Gemini structured generation
(single API call → AssignmentPackage schema)
        │
        ▼
┌────────────────────┬──────────────────┬──────────────────┐
│  Assignment Brief  │  Marking Rubric  │  Marking Scheme  │
└────────────────────┴──────────────────┴──────────────────┘
        │                    │                   │
        ▼                    ▼                   ▼
    .docx files saved to generated_assignments/
        │
        ├── SQLite metadata logged to akadverse_assignments.db
        │
        ├── [Optional] Uploaded to Google Drive as native Google Docs
        │   → /AkadVerse/2026/Assignments/ folder
        │
        └── Kafka mock: assignment.created event published
```

**Key design decisions:**

- **Single API call for three documents** guarantees internal consistency.
  Marks, tasks, and criteria are coherent across all three because Gemini
  generates them as one structured output object.
- **Actual `.docx` files are uploaded to Drive** rather than being
  recreated as plain text. Drive's MIME type conversion transforms each
  `.docx` into a fully editable native Google Doc on arrival.
- **Google Docs sync is optional and non-fatal.** If `token.json` is
  missing or expired, the service completes successfully with local files
  and logs a clear warning. Nothing breaks.
- **Dynamic model discovery** -- `ListModels` is called at runtime to
  find the best available Gemini model rather than hardcoding a name.

---

## Prerequisites

- **Python 3.10 or higher**
- **pip** (Python package manager)
- A **Google Gemini API key** (free tier sufficient)
- For Google Docs sync only: a `token.json` OAuth credential file
  (see [Enabling Google Docs Sync](#enabling-google-docs-sync))

> **Windows users:** All commands below work in VS Code's integrated
> terminal or Windows PowerShell. Use `python` instead of `python3`
> if needed.

---

## Getting Your API Key

1. Go to [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey)
2. Sign in with a Google account.
3. Click **Create API Key**.
4. Copy the key -- you will paste it as a form field in Swagger UI.

> The free tier includes access to Gemini 2.5 Flash with quota sufficient
> for assignment generation.

---

## Installation

### Step 1 — Set up your project folder

Place `assignment_generator.py` and `requirements.txt` in a dedicated folder:

```
akadverse-assignment-generator/
├── assignment_generator.py
└── requirements.txt
```

### Step 2 — Create a virtual environment

```bash
# Create the environment
python -m venv venv

# Activate — Windows
venv\Scripts\activate

# Activate — macOS/Linux
source venv/bin/activate
```

### Step 3 — Install dependencies

```bash
pip install -r requirements.txt
```

Full dependency reference:

| Package | Purpose |
|---|---|
| `fastapi` | Web framework for the API |
| `uvicorn` | ASGI server to run FastAPI |
| `google-genai>=1.67.0` | Gemini SDK for structured generation |
| `langchain-google-genai` | LangChain wrapper for structured output |
| `langchain-core` | LangChain prompt templates |
| `pydantic` | Data validation and response schemas |
| `python-docx` | Generates formatted `.docx` files locally |
| `google-api-python-client` | Google Drive API for Docs sync |
| `google-auth` | OAuth2 credential handling |
| `google-auth-oauthlib` | OAuth2 flow support |

---

## Running the Server

From inside your project folder with the virtual environment activated:

```bash
uvicorn assignment_generator:app --host 127.0.0.1 --port 8008 --reload
```

**Expected startup output:**

```
[Startup] AkadVerse Assignment Generator initialising...
[Startup] Output directory: 'generated_assignments'
[DB] Assignments database initialised successfully.
[Startup] Ready.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8008 (Press CTRL+C to quit)
```

Two things are created automatically on first startup if they do not exist:
- `generated_assignments/` -- output folder for `.docx` files
- `akadverse_assignments.db` -- SQLite metadata database

---

## API Endpoints

### 1. `POST /generate-assignment`

**What it does:** Accepts course context, generates a complete assignment
package in one Gemini call, saves three `.docx` files locally, logs metadata
to SQLite, and optionally syncs to Google Drive.

**Form fields:**

| Field | Required | Constraints | Description |
|---|---|---|---|
| `course_title` | Yes | — | e.g. `Data Structures and Algorithms` |
| `course_code` | Yes | — | e.g. `CSC 301` |
| `topic` | Yes | — | e.g. `Binary Search Trees` |
| `academic_level` | Yes | — | e.g. `300 Level`, `Year 2`, `Postgraduate` |
| `total_marks` | Yes | 10–100 | Total marks this assignment is worth |
| `weighting` | Yes | 5–60 | Percentage of final grade |
| `deadline` | Yes | — | e.g. `Two weeks from date of issue` |
| `assignment_type` | No | default: `Individual Assignment` | e.g. `Individual Assignment`, `Group Project`, `Lab Report` |
| `learning_outcomes` | Yes | max 2000 chars | Outcomes assessed -- one per line |
| `additional_instructions` | No | default: `None.` | Extra instructions to Gemini, e.g. `Include a Python programming task` |
| `sync_to_docs` | No | default: `false` | Upload to Google Drive if `token.json` is present |
| `google_api_key` | Yes | — | Your Gemini API key |

**Success response (200 OK):**

```json
{
  "status": "success",
  "assignment_id": "fef0966687c9",
  "message": "Assignment package for 'Binary Search Trees' generated successfully. Three documents saved to 'generated_assignments'.",
  "course_title": "Data Structures and Algorithms",
  "topic": "Binary Search Trees",
  "files_saved": [
    "generated_assignments\\fef0966687c9_Binary_Search_Trees_..._brief.docx",
    "generated_assignments\\fef0966687c9_Binary_Search_Trees_..._rubric.docx",
    "generated_assignments\\fef0966687c9_Binary_Search_Trees_..._scheme.docx"
  ],
  "docs_synced": true,
  "total_marks": 30
}
```

**Validation error (400):**

```json
{
  "detail": "Learning outcomes are too short. Please provide meaningful outcomes."
}
```

**Terminal output when sync is enabled and succeeds:**

```
[Model] Using: gemini-2.5-flash
[Generate] Generating assignment: 'Binary Search Trees' for CSC 332 (300 Level)...
[Generate] Package generated successfully. Saving documents...
[DocX] Brief saved: generated_assignments\abc_Binary_Search_Trees_..._brief.docx
[DocX] Rubric saved: generated_assignments\abc_Binary_Search_Trees_..._rubric.docx
[DocX] Scheme saved: generated_assignments\abc_Binary_Search_Trees_..._scheme.docx
[Generate] Attempting Google Docs sync...
[Docs Sync INFO] Drive folder found. | context={...}
[Docs Sync INFO] Drive folder found. | context={...}
[Docs Sync INFO] Drive folder created. | context={...}
[Docs Sync INFO] Document uploaded and converted to Google Doc. | context={...}
[Docs Sync INFO] Document uploaded and converted to Google Doc. | context={...}
[Docs Sync INFO] Document uploaded and converted to Google Doc. | context={...}
[Docs Sync INFO] All three documents uploaded to /AkadVerse/2026/Assignments/ successfully.
[DB] Assignment 'abc' logged to database.
[KAFKA MOCK] Published event 'assignment.created' — ID: abc, Course: CSC 332, Topic: Binary Search Trees
```

> **Note on generation time:** This endpoint takes 15 to 30 seconds.
> Gemini is generating three detailed academic documents simultaneously.

---

### 2. `GET /assignments`

**What it does:** Returns a paginated list of previously generated
assignments from the SQLite database. Optionally filters by course title.

**Query parameters:**

| Parameter | Default | Description |
|---|---|---|
| `course_title` | — | Optional partial match filter on course title |
| `limit` | `10` | Max records returned per page |
| `offset` | `0` | Records to skip for pagination |

**Success response (200 OK):**

```json
{
  "assignments": [
    {
      "id": "fef0966687c9",
      "course_title": "Data Structures and Algorithms",
      "topic": "Binary Search Trees",
      "academic_level": "300 Level",
      "total_marks": 30,
      "docs_synced": true,
      "created_at": "2026-03-20T19:23:50.101373"
    }
  ],
  "total_returned": 1
}
```

---

### 3. `GET /health`

**What it does:** Reports service status, output directory, number of
assignments generated, and whether Google Docs sync is available.

**Success response (200 OK):**

```json
{
  "status": "ok",
  "version": "1.0",
  "output_directory": "generated_assignments",
  "output_dir_exists": true,
  "assignments_generated": 3,
  "docs_sync_available": true,
  "endpoints": {
    "POST /generate-assignment": "Generate a full assignment package (brief + rubric + scheme).",
    "GET  /assignments": "List previously generated assignments with optional filtering.",
    "GET  /health": "This endpoint."
  }
}
```

| Field | Meaning |
|---|---|
| `docs_sync_available: true` | `token.json` is present -- Drive sync is ready |
| `docs_sync_available: false` | No `token.json` -- sync will be skipped gracefully |
| `assignments_generated` | Total count in the SQLite database |

---

## Testing with Swagger UI

With the server running, open:

```
http://127.0.0.1:8008/docs
```

To test any endpoint: click its name, click **"Try it out"**, fill in
the fields, and click **"Execute"**. Keep your terminal visible alongside
the browser to watch server logs in real time.

---

## Example Test Inputs

Run these tests in order for a complete end-to-end verification.

---

### Test 1 — Health check

`GET /health` -- confirm `status: ok` and `output_dir_exists: true`. Note
the value of `docs_sync_available` before attempting any sync tests.

---

### Test 2 — Generate (local files only)

`POST /generate-assignment` with `sync_to_docs: false`:

| Field | Value |
|---|---|
| `course_title` | `Data Structures and Algorithms` |
| `course_code` | `CSC 301` |
| `topic` | `Binary Search Trees` |
| `academic_level` | `300 Level` |
| `total_marks` | `30` |
| `weighting` | `15` |
| `deadline` | `Two weeks from date of issue` |
| `assignment_type` | `Individual Assignment` |
| `learning_outcomes` | `Understand BST insertion and deletion operations`<br>`Analyse time complexity of BST operations`<br>`Implement a BST in Python or Java` |
| `additional_instructions` | `Include at least one programming task` |
| `sync_to_docs` | `false` |
| `google_api_key` | Your key |

**Expected:** `200 OK`, three file paths in `files_saved`, `docs_synced: false`.
Open `generated_assignments/` and verify three `.docx` files are present.
Open each one to confirm the content and formatting quality.

---

### Test 3 — List assignments

`GET /assignments` -- confirm one record appears with correct metadata
and `docs_synced: false`.

---

### Test 4 — Partial match filter

`GET /assignments?course_title=Data` -- should return the same record.
`GET /assignments?course_title=Physics` -- should return an empty list.

---

### Test 5 — Generate with Google Docs sync

Requires `token.json` -- see [Enabling Google Docs Sync](#enabling-google-docs-sync).

`POST /generate-assignment` with same inputs as Test 2 but `sync_to_docs: true`.

**Expected terminal output:** Three `[Docs Sync INFO] Document uploaded`
lines, each with a Google Docs link. `docs_synced: true` in the response.

**Expected in Google Drive:** Navigate to `My Drive → AkadVerse → 2026 →
Assignments`. Three documents should appear, named `[Brief]`, `[Rubric]`,
`[Scheme]`. Open one to confirm it is a fully editable Google Doc.

---

### Test 6 — Validation guard

`POST /generate-assignment` with `learning_outcomes` set to just `ok`.

**Expected:** `400 Bad Request` -- `"Learning outcomes are too short."` No
files should be created.

---

## Understanding the Responses

### Why `docs_synced: false` even when sync was requested

| Terminal message | Cause and fix |
|---|---|
| `No token.json found` | Copy `token.json` from `akadverse-workspace-service` |
| `missing fields refresh_token` | Token is incomplete -- re-authenticate with `prompt='consent'` |
| `invalid_grant: Bad Request` | Refresh token expired -- re-authenticate to get a fresh token |
| `Token is invalid and cannot be refreshed` | Same cause -- re-authenticate |

In every case the assignment is fully saved locally. `docs_synced: false`
is a warning, not a failure.

### The assignment ID

Every generation run produces a unique 12-character hex ID, e.g. `fef0966687c9`.
This ID prefixes all three `.docx` filenames, appears in Google Doc titles,
is the primary key in the SQLite database, and is included in the Kafka mock
event. It is the stable identifier that ties everything together.

### The `[KAFKA MOCK]` line

```
[KAFKA MOCK] Published event 'assignment.created' — ID: fef0966687c9, Course: CSC 332, Topic: Binary Search Trees
```

This simulates publishing to the Apache Kafka event bus. In production this
event triggers student notifications, course dashboard updates, and Insight
Engine data ingestion. During development it is log-only.

---

## Enabling Google Docs Sync

Google Docs sync requires a `token.json` OAuth credential file in the
same folder as `assignment_generator.py`. This file is generated by the
**AkadVerse Google Workspace Integration** service (Tier 4, port 8002).
The workspace service does **not** need to keep running after the token
is generated -- the assignment generator loads and uses it independently.

### Step 1 — Add `prompt='consent'` to the workspace service login

Open `akadverse-workspace-service/main.py` and update the authorization
URL call inside the `/login` endpoint:

```python
authorization_url, state = flow.authorization_url(
    access_type='offline',
    include_granted_scopes='true',
    prompt='consent'    # Forces Google to always issue a refresh_token
)
```

> Without `prompt='consent'`, repeat logins skip the `refresh_token`,
> producing an incomplete token that causes
> `"missing fields refresh_token"` errors.

### Step 2 — Revoke existing app access (recommended)

Go to [https://myaccount.google.com/permissions](https://myaccount.google.com/permissions),
find your AkadVerse app, and click **Remove Access**. This ensures
the next login is treated as a first authorisation and Google always
issues a complete token.

### Step 3 — Re-authenticate

```bash
cd akadverse-workspace-service
venv\Scripts\activate
uvicorn main:app --host 127.0.0.1 --port 8002 --reload
```

Open `http://127.0.0.1:8002/login` in your browser. Complete the Google
sign-in. You should see:

```json
{"status": "success", "message": "AkadVerse is connected! Check your folder for token.json."}
```

### Step 4 — Verify the token has a `refresh_token`

Open the new `token.json` from the workspace service folder and confirm
this field is present:

```json
{
  "token": "ya29.a0...",
  "refresh_token": "1//0g...",   ← must be present
  "token_uri": "https://oauth2.googleapis.com/token",
  ...
}
```

If it is missing, repeat Steps 2 and 3.

### Step 5 — Copy the token

```bash
# Windows -- run from inside akadverse-workspace-service
copy token.json ..\akadverse-assignment-generator\token.json
```

Or drag the file across in VS Code's Explorer panel.

### Step 6 — Confirm in the health check

`GET /health` on port 8008 should now show `docs_sync_available: true`.

---

## Generated Files

The following are created at runtime. **Do not commit them to version
control** -- list them in `.gitignore`.

| File / Folder | What it is |
|---|---|
| `generated_assignments/` | Output folder for all `.docx` files |
| `generated_assignments/*.docx` | Brief, rubric, and scheme files per generation run |
| `akadverse_assignments.db` | SQLite metadata database |
| `token.json` | Google OAuth credentials -- **never commit this** |

**Suggested `.gitignore`:**

```
generated_assignments/
akadverse_assignments.db
token.json
client_secret.json
__pycache__/
*.pyc
.env
.vscode/
```

To reset to a clean state:

```bash
# Windows
del akadverse_assignments.db
rmdir /s /q generated_assignments

# macOS/Linux
rm akadverse_assignments.db
rm -rf generated_assignments
```

---

## Common Errors and Fixes

**`ModuleNotFoundError: No module named 'docx'`**
```bash
pip install python-docx
```

**`ModuleNotFoundError: No module named 'googleapiclient'`**
```bash
pip install google-api-python-client
```

**`Docs Sync WARNING: missing fields refresh_token`**

Your `token.json` was generated without `prompt='consent'` and is missing
the refresh token. Add `prompt='consent'` to the workspace service `/login`
endpoint, revoke the app at
[https://myaccount.google.com/permissions](https://myaccount.google.com/permissions),
then re-authenticate.

**`Docs Sync WARNING: invalid_grant: Bad Request`**

Your refresh token has expired. This happens when the Google Cloud project
is in Testing mode, which limits refresh token lifetime to 7 days. Fix:
re-authenticate via the workspace service to get a fresh token, or publish
the app to Production in the
[Google Cloud Console](https://console.cloud.google.com) OAuth consent
screen settings.

**Generation returns mark totals that do not match**

Gemini occasionally allocates marks that do not sum precisely to the target.
The prompt explicitly instructs it to match totals but this is best-effort.
Re-run the generation if the mismatch matters for your use case.

**`400 Bad Request`: learning outcomes too short**

The `learning_outcomes` field requires at least 20 characters. Write out
the actual outcomes in full, one per line.

**`Address already in use` on startup**

Port 8008 is occupied. Use a different port:

```bash
uvicorn assignment_generator:app --host 127.0.0.1 --port 8009 --reload
```

---

## Project Structure

```
akadverse-assignment-generator/
│
├── assignment_generator.py        # Main microservice — all logic here
├── requirements.txt               # Python dependencies
├── README.md                      # This file
├── .gitignore                     # Excludes DB, output files, and credentials
│
├── token.json                     # Google OAuth credentials — DO NOT COMMIT
│
├── generated_assignments/         # Created on first run — DO NOT COMMIT
│   ├── abc123_Topic_..._brief.docx
│   ├── abc123_Topic_..._rubric.docx
│   └── abc123_Topic_..._scheme.docx
│
└── akadverse_assignments.db       # SQLite metadata — DO NOT COMMIT
```

---

## Part of the AkadVerse Platform

This microservice is **Tier 5** in the AkadVerse AI architecture, operating
within the *My Teaching* module alongside:

- Concept Explainer (Port 8006)
- External Resources Puller (Port 8007)
- Notes Creator (Port 8008)
- Quiz Generator (Port 8009)

The `assignment.created` Kafka event published by this service is consumed
by the Insight Engine and notification services in the full platform.
During local development it is simulated as a terminal log line.

---

*AkadVerse AI Architecture v1.0*