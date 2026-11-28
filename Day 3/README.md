# Task 3: LangChain Assistant, Evaluation Suite & Agent Preview

## 1. Project Overview & Business Domain
- **Objective:** Build a memory-enabled conversational assistant in **LangChain** (`ChatPromptTemplate`, `MessagesPlaceholder`, `StrOutputParser`) backed by Groq (`openai/gpt-oss-20b`), evaluate it systematically against a custom CSV benchmark (`tests.csv`), measure how prompt refinement and temperature tuning (`0.3` vs `1.0`) affect pass rates, classify clinic workflows by architecture, and preview `create_agent`.
- **Selected Business System:** **BrightSmile Dental Clinic** — front-desk virtual assistant adapted from the bakery template to handle clinic fees, Mon–Sat working hours, appointment policies, bilingual English/Telugu replies (`< 50` words), and medical safety boundaries.
- **Execution Environment:** Google Colab (Python 3).

---

## 2. Architecture & Execution Flow
```text
Patient Question + Conversation History (MessagesPlaceholder)
       │
       ▼
LCEL Chain: ChatPromptTemplate ──► ChatGroq (openai/gpt-oss-20b) ──► StrOutputParser()
       ├──► Updates session history list across turns
       ├──► Evaluated via run_tests("tests.csv") against must_include checks
       └──► Compared against create_agent(model=llm, tools=[], system_prompt=SYSTEM)
```

---

## 3. Tech Stack & Configuration
| Component | Library / Service | Details |
| :--- | :--- | :--- |
| **Orchestration** | `langchain` (LCEL) | `ChatPromptTemplate`, `MessagesPlaceholder("history")`, `StrOutputParser()` |
| **LLM Provider** | `langchain-groq` (`ChatGroq`) | Model: `openai/gpt-oss-20b`, tested at `temperature=0.3` and `temperature=1.0` |
| **Evaluation Suite** | Custom CSV (`tests.csv`) + `run_tests()` | 10-question benchmark checking required substrings (`must_include`) |
| **Agent Preview** | `langchain.agents` (`create_agent`) | Initialized with empty tool list (`tools=[]`) to compare baseline behavior |
| **Secrets Management** | `google.colab.userdata` | `GROQ_API_KEY` loaded into `os.environ["GROQ_API_KEY"]` |

---

## 4. Core Implementation & Observed Outputs (Cells 1–4)

### Cell 1: Environment & Model Setup
- Installed `langchain` and `langchain-groq`, configured `GROQ_API_KEY` from Colab Secrets, and initialized `ChatGroq(model="openai/gpt-oss-20b", temperature=0.3)`.

### Cell 2: Conversational Chain with Memory (`ask(question)`)
- Built the LCEL pipeline `prompt | llm | StrOutputParser()` using `MessagesPlaceholder("history")` and a Python `history` list.
- **Observed Output:** When asked *"Hi, I'm Hansitha. Do you offer teeth cleaning?"* followed by *"What's my name?"*, the assistant remembered **Hansitha** across turns while respecting clinic timings (`Mon–Sat 9 AM–7 PM`), fee rules (`Rs 1000`), medical safety guardrails, and `< 50` word bilingual responses.

### Cell 3: Automated CSV Test Runner (`run_tests("tests.csv")`)
- Implemented `run_tests()` to iterate through test cases in `tests.csv` and verify whether each model response includes the expected `must_include` string.

### Cell 4: LangChain Agent Sneak Peek (`create_agent`)
- Initialized `create_agent(model=llm, tools=[], system_prompt=SYSTEM)` and confirmed that with no tools attached (`tools=[]`), its responses match the standard conversational chain.

---

## 5. "Your 30%" Hands-On Work, Experiments & Architecture Decisions

1. **Custom Business Adaptation:** Replaced the bakery prompt with **BrightSmile Dental Clinic** rules, exact fees (`general checkup`: Rs 300, `teeth cleaning`: Rs 1000, `tooth extraction`: Rs 1500), operating hours, appointment rules, and `"I'll check with the doctor."` fallback.
2. **10-Question Benchmark (`tests.csv`):** Authored 10 patient test prompts covering service pricing, clinic timings, Sunday availability, walk-in rules, multi-session math (`2 × 1000 = 2000`), unlisted treatments (root canal), medication refusal, and Telugu queries.
3. **Prompt Refinement & Temperature Experiment:**

| Experiment Stage | Temperature | Pass Rate | Key Observations |
| :--- | :-: | :-: | :--- |
| **Baseline `tests.csv` run** | `0.3` | **70% (7/10)** | Initial benchmark before explicit fact-citing refinement. |
| **After prompt refinement** | `0.3` | **80% (8/10)** | Explicit rule to state exact numbers and timings from `FACTS` improved consistency. |
| **Temperature test — Run 1** | `1.0` | **70% (7/10)** | Higher sampling randomness caused phrasing and number-format shifts. |
| **Temperature test — Run 2** | `1.0` | **60% (6/10)** | Fluctuations in `1000` vs `1,000`, non-breaking spaces (`9 AM`), and paraphrasing `"diagnose"` as `"can't recommend medications"` affected substring checks. |

4. **Architecture Classification (Plain Code vs. One LLM Call vs. Agent):**

| Clinic Task | Chosen Approach | Reasoning |
| :--- | :--- | :--- |
| **Calculating total fee (`sessions * fee`)** | **Plain code** | Deterministic arithmetic; fast, exact, and avoids LLM formatting variations. |
| **Answering a single FAQ on timings or fees** | **One LLM call** | Single prompt-response pass using existing clinic facts and rules; no multi-step loop needed. |
| **Checking doctor availability, booking an appointment slot, and sending SMS confirmation** | **Agent** | Multi-step workflow requiring tool calls, state checks across steps, and dynamic decisions. |

---

## 6. How to Run
1. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
2. Open the Task 3 notebook in Google Colab and run all cells sequentially (`%%writefile tests.csv` must execute before `run_tests()`).
