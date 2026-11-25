# Task 3: LangChain Assistant, Evaluation Suite & Agent Preview

## 1. What the Task Is & System Chosen
- **Goal:** Build a stateful conversational assistant in **LangChain** (`ChatPromptTemplate`, `MessagesPlaceholder`, `StrOutputParser`) using Groq (`openai/gpt-oss-20b`), test it against a custom CSV benchmark, run prompt and temperature experiments, and preview a LangChain agent (`create_agent`).
- **Selected Business System:** **BrightSmile Dental Clinic** (replacing the default bakery template with dental clinic services, fees, timings, and patient-care rules).
- **Environment:** Google Colab (Python 3) with `GROQ_API_KEY` loaded from Colab Secrets.

## 2. Core Implementation & Outputs (Cells 1–4)
1. **Setup (`Cell 1`):** Installed `langchain` and `langchain-groq`, configured `GROQ_API_KEY`, and initialized `ChatGroq(model="openai/gpt-oss-20b", temperature=0.3)`.
2. **Conversational Chain with Memory (`Cell 2`):** Built the LCEL pipeline `prompt | llm | StrOutputParser()` using `MessagesPlaceholder("history")` and `ask(question)`.
   - **Output Observed:** When asked *"Hi, I'm Hansitha. Do you offer teeth cleaning?"* followed by *"What's my name?"*, the assistant remembered the name **Hansitha** across turns while obeying clinic rules (Mon–Sat 9 AM–7 PM, Rs 1000 fee, no diagnoses/prescriptions, English/Telugu matching under 50 words).
3. **Automated Test Runner (`Cell 3`):** Implemented `run_tests("tests.csv")` to check whether each model reply contains the expected `must_include` keyword.
4. **LangChain Agent Sneak Peek (`Cell 4`):** Initialized `create_agent(model=llm, tools=[], system_prompt=SYSTEM)` and verified that with an empty tool list it responds identically to the standard chain.

## 3. "Your 30%" Hands-On Work & Results
1. **Custom Business Adaptation:** Rewrote the `SYSTEM` prompt for **BrightSmile Dental Clinic** with exact service fees (`general checkup`: Rs 300, `teeth cleaning`: Rs 1000, `tooth extraction`: Rs 1500), timings, appointment policies, and fallback behavior (`"I'll check with the doctor."`).
2. **10-Question Test Suite (`tests.csv`):** Created 10 patient test cases covering service costs, clinic hours, Sunday availability, walk-in rules, multi-session calculation (`2 × 1000 = 2000`), unlisted services (root canal), medication refusal, and Telugu queries.
   - **Initial Output (`temperature=0.3`):** **7/10 passed (70%)**.
3. **Prompt Refinement & Re-Test:** Updated the `SYSTEM` prompt with an explicit instruction to state exact numbers and timings directly from `FACTS` and re-ran `run_tests()`.
   - **Improved Output (`temperature=0.3`):** Pass rate increased from **70% (7/10) to 80% (8/10)**.
4. **Temperature Sensitivity Experiment (`temperature=1.0`):** Switched `temperature` to `1.0` and executed `run_tests()` twice consecutively.
   - **Run 1 Output:** **7/10 passed (70%)**
   - **Run 2 Output:** **6/10 passed (60%)**
   - **Key Finding:** Higher temperature introduces non-determinism across identical runs—for example, formatting `1000` vs `1,000`, using non-breaking spaces in `9 AM`, and paraphrasing `"diagnose"` as `"can't recommend medications"`, causing exact keyword checks to vary.
5. **Architecture Classification (Plain Code vs. One LLM Call vs. Agent):**
   - **Calculating total fee (`sessions * fee`):** **Plain code** (deterministic arithmetic, no LLM needed).
   - **Answering a single FAQ on timings or fees:** **One LLM call** (single prompt-response pair).
   - **Checking doctor availability, booking an appointment slot, and sending SMS confirmation:** **Agent** (requires multi-step reasoning and tool execution).

## 4. How to Run
1. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
2. Open the Task 3 notebook in Google Colab and run all cells in order (`%%writefile tests.csv` must execute before `run_tests()`).
