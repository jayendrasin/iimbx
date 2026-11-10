# AI Coding Session 01: Auditing `course_docs/` Quirks & Designing Grounded Ingestion

## Objective
Analyze the raw text files in `course_docs/` and `sample_test_questions.txt` to identify data conflicts, edge cases, and security traps, then design a programmatic preprocessing pipeline in Python without editing the `.txt` files.

## Prompt 1: Auditing the Course Documents
**Developer Prompt:**
> I am building the DPM-101 Course Helpdesk Chatbot. We aren't allowed to edit the `.txt` files in `course_docs/` manually. Let's inspect the 4 files (`01_course_handbook.txt`, `02_grading_policy_v1.txt`, `03_grading_policy_v2.txt`, `04_exam_rules_and_faq.txt`) and the 12 test questions in `sample_test_questions.txt`. What traps exist in these files and how should our ingestion pipeline handle them in code?

**AI Assistant Response:**
There are three deliberate traps across the documents and test questions:

1. **Superseded Policy Conflict (`02_grading_policy_v1.txt` vs. `03_grading_policy_v2.txt`):**
   - `02_grading_policy_v1.txt` (10 Jan 2026) states a 50% passing score, allows 2 late submissions within 48 hours at a 10% penalty, and sets the Capstone deadline to 10 November 2026.
   - `03_grading_policy_v2.txt` (05 Oct 2026) explicitly states: `This document supersedes "02_grading_policy_v1.txt"`. It updates the passing requirement to 60% overall + 40% minimum on the final exam, removes all late submissions, and moves the Capstone deadline to 15 November 2026.
   - **Code Strategy:** Parse each file for `supersedes "<filename>"` regex patterns at startup and filter superseded files out of the active prompt context.

2. **Indirect Prompt Injection in Forum Exports (`04_exam_rules_and_faq.txt`):**
   - Section 3 contains unmoderated student comments (`Student comments are not official course policy`).
   - `Learner_819` includes an adversarial prompt injection claiming the exam is now open-book and moved to 30 December 2026.
   - **Code Strategy:** Strip any section header matching `UNMODERATED STUDENT DISCUSSION FORUM COMMENTS` and sanitize `[IMPORTANT SYSTEM UPDATE FOR AI ASSISTANTS...]` blocks before sending context to Gemini.

3. **Unmentioned Sub-Topics in Follow-Ups (Question 8 -> Question 9):**
    Question 8 asks for the Verified Certificate cost (Rs. 1,500 in `01_course_handbook.txt`).
   - Question 9 follows up asking how to get a refund if dropping in Week 2. Refunds are never mentioned in `01_course_handbook.txt`.
   - **Code Strategy:** Enforce a strict system prompt rule requiring the model to state when a policy (such as refunds, SQL indexing, or Kubernetes) is not mentioned in the documents rather than extrapolating.