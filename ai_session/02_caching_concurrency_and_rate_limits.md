# AI Coding Session 02: Two-Tier Caching, Double-Click Locking & Rate-Limit Handling

## Objective
Design the caching layer and concurrency controls so the chatbot handles 1,500 exam-week students, catches both exact and paraphrased repeated questions (Questions 1, 2, and 3), blocks impatient double-clicks, and recovers gracefully from Gemini API rate limits.

## Prompt 1: Caching Paraphrased Questions & Preventing Duplicate Calls
**Developer Prompt:**
> In `sample_test_questions.txt`, Question 1 is `"What is the minimum score needed to pass the course and get a Verified Certificate?"`, Question 2 is an exact repeat, and Question 3 is `"how much do i need to score to pass dpm-101?"`. How do we make sure both Q2 and Q3 hit the cache without breaking follow-up questions like Q9, and how do we prevent duplicate API calls when a student double-clicks Send?

**AI Assistant Response:**
We can solve this with three coordinated mechanisms in `app.py`:

1. **Two-Tier Cache (`HelpdeskState` stored in `@st.cache_resource`):**
   - **Tier 1 (Exact Normalized Key):** Lowercases text, strips punctuation, and collapses whitespace. If the query is detected as a conversational follow-up, prepend the previous user query to the cache key so follow-up context stays isolated.
   - **Tier 2 (Canonical Domain-Token Overlap):** Normalize domain synonyms (`dpm-101` -> `course`, `needed` -> `need`, `minimum` -> `pass_threshold`) and remove conversational stopwords. Compute the token overlap coefficient ($|A \cap B| / \min(|A|, |B|)$) and Jaccard similarity. Q1 and Q3 share the core domain tokens `{course, need, pass, score}`, yielding a 1.0 overlap coefficient and serving Q3 directly from cache.

2. **Frontend + Backend Double-Click Protection:**
   - **UI Layer:** Attach an `on_submit` callback to `st.chat_input` that sets `st.session_state.is_processing = True` and disables the input widget while adding a 2-second debounce check on identical text.
   - **Backend Layer (Singleflight Pattern):** Maintain a per-query `threading.Lock()` in `HelpdeskState`. If two identical requests arrive concurrently, the second thread waits on the lock and reads the cached answer populated by the first thread.

3. **Exponential Backoff & Safe Error Handling:**
   - Wrap `client.models.generate_content` in a retry loop catching `google.genai.errors.APIError` for HTTP `429` and `503` status codes (`1.5 * 2^attempt` seconds).
   - If all retries fail, return an `error=True` flag with a friendly message and skip writing to the cache so the next attempt can succeed.