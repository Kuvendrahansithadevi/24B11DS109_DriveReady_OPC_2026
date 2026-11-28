# Task 2: Stateful Multi-Turn Assistant Using OpenAI-Compatible Groq API

## 1. Project Overview & Business Domain
- **Objective:** Build a stateful, multi-turn conversational assistant using the standard `openai` Python SDK connected to Groq's OpenAI-compatible endpoint (`https://api.groq.com/openai/v1`). Instead of isolated single-turn prompts, the assistant retains sliding-window conversation memory, uses structured JSON service/fee data, logs helper function execution via decorators, handles API exceptions gracefully, and runs in an interactive console loop.
- **Selected Domain:** **BrightSmile Dental Clinic** — an AI front-desk virtual assistant handling treatment fee inquiries, working hours, bilingual English/Telugu responses (`< 50` words), and patient context tracking across turns.
- **Execution Environment:** Google Colab (Python 3).

---

## 2. Architecture & Request Flow
```text
Interactive Console Loop (input("You: "))
       │
       ▼
Sliding-Window History ([history[0]] + history[1:][-MAX_TURNS:])
       │
       ▼
OpenAI-Compatible Groq Client (base_url="[https://api.groq.com/openai/v1](https://api.groq.com/openai/v1)")
       ├──► Model: openai/gpt-oss-120b (temperature=0.3)
       ├──► Context: System Prompt + JSON-serialized SERVICES fee table
       ├──► Exception Guard: Catches RateLimitError & APIConnectionError
       └──► Helper Function: get_fee(service, sessions) wrapped with @log_call
       │
       ▼
Assistant Reply (English or Telugu matching patient, < 50 words)
```

---

## 3. Tech Stack & Configuration
| Component | Library / Service | Details |
| :--- | :--- | :--- |
| **Client SDK** | `openai` (`OpenAI`) | Configured with `base_url="https://api.groq.com/openai/v1"` |
| **LLM Model** | Groq Cloud | `model="openai/gpt-oss-120b"`, `temperature=0.3` |
| **Memory Strategy** | Python `list` slicing | Sliding window (`MAX_TURNS = 8`) preserving `history[0]` (system prompt) |
| **Serialization & Logging** | `json`, `functools.wraps` | `json.dumps()` / `json.loads()` for fee tables; `@log_call` decorator for tracing |
| **Error Handling** | `openai.RateLimitError`, `openai.APIConnectionError` | `try/except` blocks returning safe user-facing fallback messages |
| **Secrets Management** | `google.colab.userdata` | `GROQ_API_KEY` loaded securely into the OpenAI client |

---

## 4. Implementation Details & Core Components

### A. Structured Service Catalog & JSON Handling
- Defined the `SERVICES` dictionary (`"general checkup": 300`, `"teeth cleaning": 1000`, `"tooth extraction": 1500`).
- Implemented safe dictionary lookups via `.get()` to avoid `KeyError` on unlisted treatments, and practiced serialization/deserialization using `json.dumps()` and `json.loads()` to inject structured pricing cleanly into the `SYSTEM` prompt.

### B. Constrained System Prompt (`BrightSmile Dental Clinic`)
- **Clinic Facts & Rules:** Enforces clinic timings (**Monday–Saturday, 9:00 AM – 7:00 PM**), appointment-only consultations, strict refusal of medical diagnoses or prescriptions, and `"I'll check with the doctor."` fallback for unknown queries.
- **Bilingual & Length Policy:** Matches the patient's language (**English or Telugu**) while keeping every response strictly **under 50 words**.

### C. Logged Helper Function (`@log_call` + `get_fee`)
- Created a reusable `@log_call` decorator using `functools.wraps` to log function names, input arguments, and return values to the console.
- Applied `@log_call` to `get_fee(service, sessions)` to compute exact multi-session fee totals (`fee * sessions`) or flag unlisted services.

### D. Sliding-Window Memory & Error Resilience (`chat(text)`)
- Appends each user and assistant message to `history` while sending `[history[0]] + history[1:][-MAX_TURNS:]` (`MAX_TURNS = 8`) to the API so context stays bounded without losing system rules.
- Wraps `client.chat.completions.create(...)` in `try/except` handlers for `RateLimitError` and `APIConnectionError` to prevent crashes during transient network or rate-limit events.

### E. Interactive Console Loop
- Implemented a `while True` loop reading user prompts via `input("You: ")`, printing assistant replies via `chat(text)`, and exiting cleanly when the user enters `bye`, `exit`, or `quit`.

---

## 5. Test Scenarios & Observed Outputs

| # | Test Scenario | Expected Behavior | Observed Result |
| :- | :--- | :--- | :- |
| 1 | **Multi-turn patient context recall** | Remember patient name and previously mentioned dental service across follow-up turns within `MAX_TURNS = 8`. | **PASS** — Retained patient identity and service context accurately across consecutive turns. |
| 2 | **JSON service fee lookup & `@log_call` trace** | Quote exact fees (`Rs 300`, `Rs 1000`, `Rs 1500`) and print decorator trace logs for `get_fee()`. | **PASS** — Printed `[LOG]` arguments/output and returned accurate totals. |
| 3 | **Bilingual matching (Telugu & English)** | Reply in Telugu when asked in Telugu and English when asked in English, under 50 words. | **PASS** — Matched language naturally while staying concise. |
| 4 | **Out-of-scope / medical advice query** | Refuse prescriptions/diagnoses and trigger `"I'll check with the doctor."` fallback. | **PASS** — Blocked medical advice and stayed within clinic guardrails. |
| 5 | **Console loop exit commands (`bye` / `exit` / `quit`)** | Terminate the interactive loop cleanly without errors. | **PASS** — Exited loop smoothly on `bye`. |

---

## 6. How to Run
1. Install the OpenAI Python package in Google Colab:
   ```bash
   pip install -q openai
   ```
2. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
3. Open the Task 2 notebook in Google Colab and run all cells sequentially.
4. Type patient queries at the `You:` prompt and enter `bye`, `exit`, or `quit` to end the session.
