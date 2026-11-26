# Task 4: Tool-Using LangChain Agent, Pydantic Validation & Google Sheets Integration

## 1. What the Task Is & System Chosen
- **Goal:** Upgrade the conversational assistant into a full **LangChain Agent** (`create_agent`) equipped with custom `@tool` functions, **Pydantic** input validation (`BaseModel` & `with_structured_output`), live **Google Sheets** persistence via `gspread`, execution trace inspection (`pretty_print()`), and tool-routing sensitivity experiments.
- **Selected Business System:** **BrightSmile Dental Clinic** front-desk appointment and fee-lookup assistant.
- **Environment & Setup:** Google Colab (Python 3) using `langchain`, `langchain-groq` (`ChatGroq` with `model="openai/gpt-oss-20b"`, `temperature=0.2`), `pydantic`, `gspread`, `GROQ_API_KEY` from Colab Secrets, and Google Colab OAuth (`google.colab.auth`).

## 2. Core Implementation & Outputs (Cells 1–4)
1. **Setup & Google Sheet Initialization (`Cell 1`):** Installed dependencies, initialized `ChatGroq`, authenticated the Google account, and created the **`BrightSmile Appointments`** spreadsheet with columns `["name", "service", "sessions", "date", "urgent", "total"]`.
2. **Pydantic Model & Custom Tools (`Cell 2`):**
   - Defined the `Appointment` Pydantic schema enforcing valid service names (`"general checkup"`, `"teeth cleaning"`, `"tooth extraction"`), positive session counts (`ge=1`), date string, and boolean urgency flag.
   - Built three `@tool` functions with clear docstrings and type hints:
     - `get_fee(service, sessions)`: Calculates total rupee fees (`300`, `1000`, `1500`) or reports unlisted services safely.
     - `is_open(day)`: Returns clinic hours (`9 am to 7 pm` Mon–Sat, `Closed` on Sunday).
     - `log_appointment(name, service, sessions, date, urgent)`: Validates inputs via `Appointment`, computes total cost, and appends confirmed bookings directly to Google Sheets.
3. **Stateful Agent Execution (`Cell 3`):** Created `agent = create_agent(model=llm, tools=[get_fee, is_open, log_appointment], system_prompt=SYSTEM)` with full message history retention.
   - **Output Observed:** Checked Sunday hours and multi-session fees accurately via tool calls, and logged confirmed appointments (e.g., *Hansitha, 2 urgent teeth cleaning sessions, 28 Nov, Rs 2000*) directly into the Google Sheet.
4. **Trace Inspection & Edge-Case Testing (`Cell 4`):** Used `m.pretty_print()` to inspect `HumanMessage` -> `AIMessage` (`tool_calls`) -> `ToolMessage` -> final `AIMessage` steps, and verified safe handling when users request unlisted treatments (root canal), omit dates, or pass `0` sessions.

## 3. "Your 30%" Hands-On Work & Observations
1. **Custom `@tool` Functions:** Converted clinic fee and schedule lookups into typed, docstring-documented LangChain `@tool` functions (`get_fee`, `is_open`).
2. **Structured Extraction (`with_structured_output`):** Tested `llm.with_structured_output(Appointment)` across 5 messy, unformatted patient booking messages and verified clean extraction into validated `Appointment` objects.
3. **Google Sheets Persistence (`log_appointment`):** Connected the `BrightSmile Appointments` Google Sheet and verified automatic row insertion only after all required patient fields were collected and validated.
4. **Tool Docstring Sensitivity ("Break It" Experiment):** Temporarily renamed `is_open` to `tool1` with a vague docstring (`"does stuff"`), observed how removing descriptive tool metadata degrades reliable tool selection, and restored `is_open`.
5. **Execution Trace Count (`pretty_print()`):** Inspected a multi-tool query (`"Are you open on Sunday and how much is 1 teeth cleaning?"`) on a fresh trace and counted the exact `AIMessage` (model calls) and `ToolMessage` (tool calls) steps executed by the agent loop.
6. **RAG Knowledge Base Preparation (Before Day 5):** Created `clinic_facts.txt` containing 16 structured policies, timings, pricing rules, and clinic FAQs for retrieval-augmented generation.

## 4. How to Run
1. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
2. Run **Cell 1** and complete the Google OAuth prompt to authorize Google Sheets access (use `gc.open("BrightSmile Appointments")` on subsequent runs).
3. Run **Cells 2–4** and the experiment cells sequentially, then verify newly logged rows in the generated Google Sheet URL.
