# Course Q\&A Generator — Moodle Plugin

Turn **course PDFs** into **questions & answers** via **Snowflake (stage + storage table) and Snowflake Cortex**, then store the generated Q\&A back in Moodle and link them to the correct course. Includes an editor and selective re-generation.

---

## Table of Contents

* [Overview](#overview)
* [Architecture & Data Flow](#architecture--data-flow)
* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Configuration](#configuration)
* [Using the Plugin](#using-the-plugin)

  * [Upload & Generation Workflow](#upload--generation-workflow)
  * [Editing, Approving, Regenerating](#editing-approving-regenerating)
* [Backend Endpoint Contract](#backend-endpoint-contract)

  * [Request](#request)
  * [Response](#response)


---

## Overview

This plugin lets you:

* Upload a **PDF course file** and select/enter a **course**.
* Send the file to a **backend** (`http://localhost:5000/upload`) that:

  * Parses the PDF in Python.
  * Stores raw/parsed content to a **Snowflake stage** and a **Snowflake storage table**, linked by **courseId**.
  * Sends parsed content to **Snowflake Cortex** to generate **questions & answers (Q\&A)**.
  * Returns the generated Q\&A to Moodle.
* Store the Q\&A in Moodle and attach them to the selected course.
* **Edit** Q\&A, **approve** final items, and **regenerate** only the **unapproved** ones.

---

## Architecture & Data Flow

```
[Moodle Plugin UI]
      |
      | 1) POST PDF + course info
      v
[Backend @ http://localhost:5000/upload]
      | 2) Parse PDF (Python)
      | 3) Stage raw/parsed to Snowflake stage
      | 4) Insert parsed content -> Snowflake storage table (courseId)
      | 5) Send parsed content -> Snowflake Cortex (generate Q&A)
      v
[Backend returns Q&A JSON]
      |
      | 6) Store Q&A in Moodle + attach to course
      v
[Moodle Plugin UI: edit / approve / regenerate (unapproved only)]
```

---

## Prerequisites

* Moodle admin access to install and configure plugins.
* A locally running backend service at **`http://localhost:5000/upload`** that accepts `multipart/form-data` and implements the flow above.
* Snowflake:

  * A **stage** for file/content staging.
  * A **storage table** for parsed content (with `courseId` linkage).
  * Credentials for the backend to write to the stage & table.
  * **Snowflake Cortex** enabled and accessible from the backend.

---

## Installation

1. In Moodle, go to **Site administration → Plugins → Install plugins**.
2. Upload the plugin ZIP and complete the installation.
3. After installation, proceed to **Configuration**.

---

## Configuration

1. Go to **Site administration → Plugins → Course Q\&A Generator (this plugin)**.

2. Set **Backend Upload URL** to:

   ```
   http://localhost:5000/upload
   ```

3. Save changes.

> The backend must be reachable from the Moodle server and accept `multipart/form-data` with the PDF and course metadata.

---

## Using the Plugin

### Upload & Generation Workflow

1. Open **Site administration → Plugins → Course Q\&A Generator** (plugin main page).
2. Use the **Upload form**:

   * **Course**: select an existing course (or enter the course name if the UI supports it).
   * **PDF file**: choose the course PDF.
3. Click **Upload**.
4. The plugin sends the file + course info to the backend:

   * Backend **parses** the PDF.
   * Stores data to Snowflake **stage** and **storage table** (linked to **courseId**).
   * Calls **Snowflake Cortex** to generate **Q\&A**.
   * Returns the **Q\&A JSON** to Moodle.
5. The plugin **stores Q\&A** in Moodle and **attaches** them to the selected course.

### Editing, Approving, Regenerating

On the plugin page (or per-course Q\&A view), you can:

* **Edit** question text, answers, explanations; **Save** changes.
* **Approve** finalized items. Approved items are **not regenerated**.
* **Regenerate** Q\&A — this only affects **unapproved** items.

**Regeneration rule:**

> Only **unapproved** questions/answers are regenerated. Approved items remain as-is (but can still be manually edited).

---

## Backend Endpoint Contract

> The plugin expects a single endpoint that handles upload, parsing, Snowflake ops, Cortex generation, and returns Q\&A.

### Request

* **Method:** `POST`
* **URL:** `http://localhost:5000/upload`
* **Content-Type:** `multipart/form-data`
* **Fields:**

  * `file` — PDF file
  * `courseId` — Moodle course ID (string/integer)


### Response

* **200 OK** with JSON payload similar to:

```json
{
  "status": "ready",
  "courseId": "12345",
  "questions": [
    {
      "id": "q-001",
      "question": "What is ...?",
      "answers": [
        {"text": "Option A", "correct": false},
        {"text": "Option B", "correct": true},
        {"text": "Option C", "correct": false}
      ],
      "explanation": "Because ...",
      "approved": false
    }
  ]
}
```

* On error, respond with non-200 and a JSON error body:

```json
{
  "status": "error",
  "message": "Parsing failed: <details>"
}
```

---
