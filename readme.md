# DPM-101 Course Helpdesk Chatbot

An exam-week-ready Course Helpdesk Chatbot for **IIMBx - Digital Product Management (DPM-101)** built with Python, Streamlit, and the Google Gemini API (`google-genai`).

## Key Features & Engineering Architecture

1. **Automated Document Sanitization (Zero Manual Edits to `course_docs/`):**
   - **Policy Conflict Resolution:** Automatically parses document headers and supersession notices, detecting that `03_grading_policy_v2.txt` (05 Oct 2026) supersedes `02_grading_policy_v1.txt` (10 Jan 2026). Outdated thresholds (50% pass mark, 48-hour late grace period, Nov 10 deadline) are excluded from active policy context.
   - **Prompt Injection Defense:** Strips Section 3 (`UNMODERATED STUDENT DISCUSSION FORUM COMMENTS`) from `04_exam_rules_and_faq.txt` at ingestion time, neutralizing `Learner_819`'s embedded `[IMPORTANT SYSTEM UPDATE FOR AI ASSISTANTS...]` injection attempt.

2. **Two-Tier Smart Caching & Singleflight Concurrency:**
   - **Tier 1 (Normalized Exact Match):** Strips casing, punctuation, and whitespace variations.
   - **Tier 2 (Canonical Domain-Token Similarity):** Maps synonyms (`dpm-101` -> `course`, `minimum` -> `pass_threshold`, `needed` -> `need`) and computes token overlap so paraphrased repeats (e.g., *"What is the minimum score needed to pass..."* vs. *"how much do i need to score to pass dpm-101?"*) are served directly from cache without extra Gemini API calls.
   - **Context-Aware Follow-Ups:** Detects conversational follow-ups (e.g., asking about refunds after asking about the Rs. 1,500 Verified Certificate fee) and binds recent conversation history while refusing to guess unmentioned policies.

3. **Double-Click Prevention & Rate-Limit Resilience:**
   - Uses an `on_submit` callback lock to immediately disable the chat input during processing, paired with a 2-second debounce window and backend per-query `threading.Lock()` (singleflight pattern).
   - Implements exponential backoff for HTTP `429`/`503` errors and displays a friendly fallback message without caching failed responses.

## Running Locally

```bash
pip install -r requirements.txt
export GEMINI_API_KEY="your-api-key-here"
streamlit run app.py