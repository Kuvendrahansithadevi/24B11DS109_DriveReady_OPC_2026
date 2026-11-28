# Task 5: Complete Single-Agent Assistant with ChromaDB RAG & Thread-Isolated Memory

## 1. Project Overview & Business Domain
- **Objective:** Build an end-to-end, stateful single-agent conversational assistant combining deterministic tool calling, **Retrieval-Augmented Generation (RAG)** via **ChromaDB**, **thread-isolated conversation memory** (`InMemorySaver` + `thread_id`), Pydantic-validated **Google Sheets** persistence, out-of-scope hallucination guardrails, and an automated 12-question evaluation benchmark (`>= 80%` sign-off target).
- **Selected Domain:** **BrightSmile Dental Clinic** (Main Road, Rajahmundry) — an AI front-desk virtual assistant capable of quoting treatment fees, checking working hours, answering clinic policies/FAQs in English or Telugu, and logging confirmed patient appointments directly to Google Sheets.
- **Execution Environment:** Google Colab (Python 3).

---

## 2. Architecture & Request Flow
```text
Patient Message (with thread_id)
       │
       ▼
LangGraph Checkpointer (InMemorySaver) ──► Loads/saves per-patient thread history
       │
       ▼
LangChain Agent (create_agent + ChatGroq: openai/gpt-oss-20b, temp=0.2)
       ├──► get_fee(service, sessions)         ──► Exact treatment fee calculation
       ├──► is_open(day)                       ──► Daily clinic schedule lookup
       ├──► search_clinic_faq(question)        ──► ChromaDB semantic search (16 clinic facts)
       └──► log_appointment(...)               ──► Pydantic validation + Google Sheets append_row
       │
       ▼
Grounded Response (< 60 words, English or Telugu matching the patient)
```

---

## 3. Tech Stack & Configuration
| Component | Library / Service | Details |
| :--- | :--- | :--- |
| **LLM Provider** | `langchain-groq` (`ChatGroq`) | Model: `openai/gpt-oss-20b`, `temperature=0.2`, `max_retries=3` |
| **Agent & Memory** | `langchain`, `langgraph` | `create_agent` with `InMemorySaver()` checkpointer (`thread_id` isolation) |
| **Vector Store (RAG)** | `chromadb` | In-memory client, collection `brightsmile_faq` (`all-MiniLM-L6-v2` embeddings) |
| **Data Validation** | `pydantic` (`BaseModel`, `Field`) | `Appointment` schema validating service literals, `sessions >= 1`, date, and urgency |
| **Database / Logging** | `gspread`, `google.colab.auth` | Live Google Sheet: `BrightSmile Appointments` |
| **Secrets Management** | `google.colab.userdata` | `GROQ_API_KEY` loaded securely from Colab Secrets |

---

## 4. Tools & Data Schema Reference

### A. Pydantic Validation Schema (`Appointment`)
| Field | Type / Constraint | Description |
| :--- | :--- | :--- |
| `name` | `str` | Patient's first name |
| `service` | `Literal["general checkup", "teeth cleaning", "tooth extraction"]` | Enforces valid clinic services |
| `sessions` | `int` (`ge=1`) | Number of sessions (rejects `0` or negative inputs) |
| `date` | `str` | Appointment date as stated by the patient |
| `urgent` | `bool` | Flags emergency/urgent appointment requests |

### B. Agent Tools (`@tool`)
| Tool Name | Parameters | Responsibility |
| :--- | :--- | :--- |
| `get_fee` | `service: str, sessions: int = 1` | Computes exact rupee totals (`Rs 300`, `Rs 1000`, `Rs 1500`) or safely flags unlisted treatments. |
| `is_open` | `day: str` | Returns clinic hours (`9 am to 7 pm` Mon–Sat, `Closed on Sunday`). |
| `search_clinic_faq` | `question: str` | Queries ChromaDB (`n_results=3`) for clinic rules/policies without guessing. |
| `log_appointment` | `name, service, sessions, date, urgent` | Validates via `Appointment`, calculates total fee, and appends a row to Google Sheets. |

---

## 5. Step-by-Step Implementation & Observed Outputs

### Cell 1: ChromaDB Knowledge Ingestion (16 Clinic Facts)
- **Implementation:** Created the `brightsmile_faq` ChromaDB collection and upserted **16 single-fact chunks** covering clinic location, Mon–Sat working hours, Sunday emergency policy, treatment fees, prior-booking requirement, 10-minute early arrival, 4-hour cancellation notice, digital X-rays, pediatric age eligibility (`4+`), UPI/card/cash payment methods, `10%` senior citizen discount, autoclave sterilization protocols, and medical diagnosis/prescription restrictions.
- **Observed Output:**
  - Confirmed `16 facts stored`.
  - Semantic test query `"Can I pay using PhonePe or GPay?"` accurately retrieved `['Payments are accepted via UPI, debit/credit cards, and cash.', 'General dental checkup fee is Rs 300 per session.']`.

### Cell 2: `search_clinic_faq` RAG Tool
- **Implementation:** Defined `@tool def search_clinic_faq(question: str)` with a scoped docstring instructing the LLM to invoke it exclusively for clinic rules, policies, payments, discounts, and FAQs while leaving fees and opening hours to `get_fee` and `is_open`.

### Cell 3: Complete Stateful Agent (`InMemorySaver` + `thread_id`)
- **Implementation:** Wired all 4 tools into `create_agent(..., checkpointer=InMemorySaver())` and passed `config={"configurable": {"thread_id": thread_id}}` on every `agent.invoke()` call.
- **Observed Output (Multi-Thread Isolation Test):**
  - Thread `"hansitha"`: Stored patient name (`Hansitha`), retrieved UPI payment acceptance, 10% senior discount, and 10-minute arrival policy across consecutive turns.
  - Thread `"kiran"`: Answered pediatric age eligibility (`4 years and above`) independently and confirmed zero memory leakage from `"hansitha"`'s session.

### Cell 4: Automated 12-Question Evaluation (`tests.csv` & `run_agent_tests()`)
- **Implementation:** Created a 12-question evaluation suite covering treatment costs, multi-session math, clinic hours, Sunday status, UPI payments, cancellation notice, senior citizen discount, location, pediatric age limit, and out-of-scope queries, executing each row on an isolated thread (`eval_thread_{idx}`).

---

## 6. "Your 30%" Hands-On Work, Guardrail Tuning & Final Results

1. **Knowledge Base Creation:** Authored and ingested 16 domain-specific facts for BrightSmile Dental Clinic into ChromaDB (one fact per chunk).
2. **Custom RAG Tool Design:** Built `search_clinic_faq` with clear boundary instructions separating policy retrieval from deterministic fee/schedule tools.
3. **Thread-Isolated Memory Verification:** Tested multi-turn recall within one `thread_id` and verified strict isolation across separate patient threads.
4. **Out-of-Scope Guardrail & Prompt Refinement:**
   - **Initial Evaluation (`67%` — 8/12 passed):** Responses were factually accurate, but substring checks failed due to comma formatting (`₹1,000` instead of `1000`), non-breaking spaces (`10 %` instead of `10%`), and asking which weekday the user wanted for general clinic hours.
   - **System Prompt Updates Applied:**
     - Instructed the agent to write numbers and percentages plainly without commas or extra spaces (`Rs 1000`, `10%`).
     - Added default Mon–Sat hours lookup when general clinic hours are requested without a specific weekday.
     - Enforced strict fallback (`"I'll check with the doctor."`) for topics outside the 16 ChromaDB facts.
5. **Final Benchmark Sign-Off (`92%` — 11/12 passed):**

| # | Test Question | Expected Keyword | Model Output Preview | Result |
| :- | :--- | :--- | :--- | :- |
| 1 | How much does a general checkup cost? | `300` | A general checkup costs Rs 300. | **PASS** |
| 2 | What is the fee for teeth cleaning? | `1000` | The fee for a teeth cleaning session is Rs 1000. | **PASS** |
| 3 | How much for a tooth extraction? | `1500` | A single tooth extraction costs Rs 1500... | **PASS** |
| 4 | What are your clinic hours? | `9` | Hi! We’re open Monday through Saturday from 9 am to 7 pm... | **PASS** |
| 5 | Are you open on Sunday? | `Sunday` | We’re closed on Sundays. We’re open Monday through Saturday... | **PASS** |
| 6 | What is the total cost for 2 teeth cleaning sessions? | `2000` | The total cost for two teeth cleaning sessions is Rs 2000. | **PASS** |
| 7 | Can I pay by UPI? | `UPI` | Yes, you can pay by UPI. We also accept debit/credit cards and cash... | **PASS** |
| 8 | How many hours before should I cancel an appointment? | `4` | You should cancel at least 4 hours before your appointment. | **PASS** |
| 9 | Do senior citizens get any discount? | `10%` | Yes, senior citizens receive a 10% discount on general check-ups. | **PASS** |
| 10 | Where is BrightSmile Dental Clinic located? | `Rajahmundry` | BrightSmile Dental Clinic is located at Main Road, Rajahmundry. | **PASS** |
| 11 | What is the minimum age for children? | `4` | Pediatric dental checkups are available for children aged 4 and above. | **PASS** |
| 12 | Do you sell hair transplant packages? | `check with the doctor` | I’m sorry, but BrightSmile Dental Clinic does not offer hair transplant packages. | **FAIL** *(accurate refusal, phrased differently)* |

---

## 7. Troubleshooting Notes
- **Colab Package Compatibility:** Upgraded `langchain`, `langchain-core`, and `langchain-groq` together (`pip install -q -U langchain langchain-core langchain-groq gspread chromadb`) to resolve `ImportError: cannot import name 'ModelRefusalError'`.
- **Standalone Execution:** Consolidated the base LLM setup, Google Sheets authorization (`gc.open("BrightSmile Appointments")`), Pydantic `Appointment` model, and Day 4 tools into the same runtime before initializing ChromaDB and `InMemorySaver`.

---

## 8. How to Run
1. Save your Groq API key in Google Colab **Secrets** and enable **Notebook access**.
2. Run the setup & Google Sheets authentication cell and complete the OAuth prompt.
3. Run the Pydantic model & tools cell (`get_fee`, `is_open`, `log_appointment`), followed by the ChromaDB ingestion cell (`16 facts stored`) and `search_clinic_faq`.
4. Run the `InMemorySaver` agent cell to test multi-thread memory, then execute `run_agent_tests()` to reproduce the **92% (11/12)** evaluation benchmark.
