# Task 4: Tool-Using LangChain Agent, Pydantic Validation & Google Sheets Integration

## 1. Project Overview & Business Domain
- **Objective:** Upgrade the conversational assistant into a tool-calling **LangChain Agent** (`create_agent`) equipped with custom `@tool` functions, **Pydantic** input validation (`BaseModel` and `with_structured_output`), live **Google Sheets** persistence via `gspread`, execution trace inspection (`pretty_print()`), and tool-routing sensitivity experiments.
- **Selected Domain:** **BrightSmile Dental Clinic** — an AI front-desk virtual assistant capable of quoting treatment fees, checking working hours, validating patient appointment details, and logging confirmed bookings directly to Google Sheets.
- **Execution Environment:** Google Colab (Python 3).

---

## 2. Architecture & Request Flow
```text
Patient Message + Conversation History
       │
       ▼
LangChain Agent (create_agent + ChatGroq: openai/gpt-oss-20b, temp=0.2)
       ├──► get_fee(service, sessions)         ──► Exact treatment fee calculation
       ├──► is_open(day)                       ──► Daily clinic schedule lookup
       └──► log_appointment(...)               ──► Pydantic Appointment validation
                                                    └──► Google Sheets append_row
       │
       ▼
Grounded Front-Desk Response (< 60 words, English or Telugu matching the patient)
```

---

## 3. Tech Stack & Configuration
| Component | Library / Service | Details |
| :--- | :--- | :--- |
| **LLM Provider** | `langchain-groq` (`ChatGroq`) | Model: `openai/gpt-oss-20b`, `temperature=0.2`, `max_retries=3` |
| **Agent Framework** | `langchain` (`create_agent`, `@tool`) | Tool-calling loop inspected via `m.pretty_print()` |
| **Data Validation** | `pydantic` (`BaseModel`, `Field`) | `Appointment` schema + `llm.with_structured_output(Appointment)` |
| **Database / Logging** | `gspread`, `google.colab.auth` | Live Google Sheet: `BrightSmile Appointments` |
| **Secrets Management** | `google.colab.userdata` | `GROQ_API_KEY` loaded securely from Colab Secrets |

---

## 4. Tools & Data Schema Reference

### A. Pydantic Validation Schema (`Appointment`)
| Field | Type / Constraint | Description |
| :--- | :--- | :--- |
| `name` | `str` | Patient's first name |
| `service` | `Literal["general checkup", "teeth cleaning", "tooth extraction"]` | Restricts booking to valid clinic services |
| `sessions` | `int` (`ge=1`) | Number of sessions (rejects `0` or negative counts) |
| `date` | `str` | Appointment date as stated by the patient |
| `urgent` | `bool` | Flags emergency/urgent slot requests |

### B. Custom Agent Tools (`@tool`)
| Tool Name | Parameters | Responsibility |
| :--- | :--- | :--- |
| `get_fee` | `service: str, sessions: int = 1` | Computes exact rupee totals (`Rs 300`, `Rs 1000`, `Rs 1500`) or reports unlisted treatments safely. |
| `is_open` | `day: str` | Returns clinic hours (`9 am to 7 pm` Mon–Sat, `Closed on Sunday`). |
| `log_appointment` | `name, service, sessions, date, urgent` | Validates fields through `Appointment`, computes total cost, and appends a confirmed row to Google Sheets. |

---

## 5. Core Implementation & Observed Outputs (Cells 1–4)

### Cell 1: Setup & Google Sheet Initialization
- Installed `langchain`, `langchain-groq`, and `gspread`, initialized `ChatGroq`, authenticated the Google account via `google.colab.auth`, and created/opened the **`BrightSmile Appointments`** spreadsheet with header columns `["name", "service", "sessions", "date", "urgent", "total"]`.

### Cell 2: Pydantic Model & Custom `@tool` Definitions
- Defined the `Appointment` validation model and implemented `get_fee`, `is_open`, and `log_appointment` with clear type hints and docstrings so the LLM knows when and how to invoke each tool.

### Cell 3: Stateful Agent Execution & Live Sheet Logging
- Built `agent = create_agent(model=llm, tools=[get_fee, is_open, log_appointment], system_prompt=SYSTEM)` with message history tracking.
- **Observed Output:** Accurately checked Sunday closure and multi-session fees via tool calls, and appended confirmed bookings (e.g., *Hansitha, 2 urgent teeth cleaning sessions, 28 Nov, Rs 2000*) directly as a new row in Google Sheets.

### Cell 4: Execution Trace Inspection & Edge-Case Testing
- Inspected the full message chain (`HumanMessage` -> `AIMessage` with `tool_calls` -> `ToolMessage` -> final `AIMessage`) via `m.pretty_print()`.
- Verified safe refusal/clarification when users ask for unlisted services (root canal), omit an appointment date, or request `0` sessions without writing invalid rows to Google Sheets.

---

## 6. "Your 30%" Hands-On Work & Experiments

1. **Custom `@tool` Functions:** Converted clinic fee and schedule lookups into typed, docstring-documented LangChain `@tool` functions (`get_fee`, `is_open`).
2. **Structured Extraction (`with_structured_output`):** Tested `llm.with_structured_output(Appointment)` across 5 messy, unformatted patient booking messages and verified clean extraction into validated `Appointment` objects.
3. **Google Sheets Persistence (`log_appointment`):** Connected the `BrightSmile Appointments` Google Sheet and verified row insertion only after all required patient fields were collected and validated.
4. **Tool Docstring Sensitivity ("Break It" Experiment):**

| Experiment Step | Tool Configuration | Observed Behavior |
| :--- | :--- | :--- |
| **Broken state** | Renamed `is_open` to `tool1` with docstring `"does stuff"` | Removing semantic tool naming and descriptive docstrings degraded reliable tool routing on schedule queries. |
| **Restored state** | Restored `@tool def is_open(day: str)` with clear docstring | Agent reliably invoked `is_open` for every weekday/Sunday timing question. |

5. **Execution Trace Count (`pretty_print()`):** Ran a combined query (`"Are you open on Sunday and how much is 1 teeth cleaning?"`) on a fresh trace and counted the exact `AIMessage` (model calls) and `ToolMessage` (tool calls) steps executed by the agent loop.
6. **RAG Knowledge Base Preparation (Before Day 5):** Created `clinic_facts.txt` containing 16 structured policies, timings, pricing rules, and clinic FAQs for Day 5 ChromaDB ingestion.

---

## 7. How to Run
1. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
2. Run **Cell 1** and complete the Google OAuth prompt to authorize Google Sheets access (use `gc.open("BrightSmile Appointments")` on subsequent runs).
3. Run **Cells 2–4** and the experiment cells sequentially, then verify newly logged appointment rows in the generated Google Sheet URL.
