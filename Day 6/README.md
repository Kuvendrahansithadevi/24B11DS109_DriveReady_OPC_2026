# Task 6: Multi-Node Intent Router & Specialist Agents in LangGraph

## 1. Project Overview & Business Domain
- **Objective:** Replace a single monolithic agent with a structured **LangGraph** (`StateGraph`) router that classifies incoming patient messages by intent (`appointment`, `faq`, `other`), dispatches them through conditional edges to dedicated specialist nodes/agents, and streams step-by-step node updates.
- **Selected Domain:** **BrightSmile Dental Clinic** (Main Road, Rajahmundry) — routing front-desk queries across FAQ/policy/fee lookups, appointment booking with Google Sheets logging, and general greetings/out-of-scope prompts.
- **Execution Environment:** Google Colab (Python 3).

---

## 2. Graph Architecture & Routing Flow
```text
                    ┌───────────┐
                    │ __start__ │
                    └─────┬─────┘
                          ▼
                    ┌───────────┐
                    │ classify  │  (ChatGroq + with_structured_output(Intent))
                    └─────┬─────┘
        ┌─────────────────┼─────────────────┐
        ▼ (intent=="appointment") ▼ (intent=="faq")  ▼ (intent=="other")
┌───────────────┐ ┌───────────┐ ┌───────────┐
│  appointment  │ │    faq    │ │   other   │
│  (specialist  │ │(specialist│ │ (welcome  │
│   agent)      │ │  agent)   │ │  handler) │
└───────┬───────┘ └─────┬─────┘ └─────┬─────┘
        └─────────────────┼─────────────────┘
                          ▼
                    ┌───────────┐
                    │  __end__  │
                    └───────────┘
```

---

## 3. Tech Stack & Configuration
| Component | Library / Service | Details |
| :--- | :--- | :--- |
| **Graph Orchestration** | `langgraph` (`StateGraph`, `START`, `END`) | Typed state routing with conditional edges and `graph.stream(..., stream_mode="updates")` |
| **Graph Visualization** | `IPython.display` + Mermaid PNG | Rendered via `Image(graph.get_graph().draw_mermaid_png())` |
| **Intent Classifier** | `pydantic` (`BaseModel`, `Field`) + `llm.with_structured_output(Intent)` | Enforces `Literal["appointment", "faq", "other"]` |
| **Specialist Agents** | `langchain.agents` (`create_agent`) | Separate `faq_agent` (`search_clinic_faq`, `get_fee`, `is_open`) and `appointment_agent` (`get_fee`, `log_appointment`) |
| **LLM & Persistence** | `langchain-groq` (`openai/gpt-oss-20b`), `chromadb`, `gspread` | Reuses Day 4–5 setup, 16-fact `brightsmile_faq` collection, and `BrightSmile Appointments` sheet |

---

## 4. State Schema, Classifier & Specialist Nodes

### A. Graph State (`ClinicState`) & Intent Model (`Intent`)
- **`ClinicState` (`TypedDict`):** Tracks `message: str`, `intent: str`, and `reply: str`.
- **`Intent` (`BaseModel`):** Constrains classification output to three explicit branches:
  - `appointment`: Booking, scheduling, or requesting a dental appointment/checkup slot.
  - `faq`: Questions about treatment fees, clinic hours, Sunday status, location, payment modes, discounts, or clinic policies.
  - `other`: Greetings (`hi`, `hello`, `good morning`) or topics unrelated to the dental clinic.

### B. Nodes & Conditional Routing
| Node / Edge | Role & Behavior |
| :--- | :--- |
| `classify` | Runs `classifier.invoke(state["message"]).intent` and updates `state["intent"]`. |
| `route` | Conditional edge function returning `state["intent"]` to select the target branch. |
| `faq` (`answer_faq`) | Invokes `faq_agent` with `search_clinic_faq`, `get_fee`, and `is_open` (`< 50` words, no guessing/prescriptions). |
| `appointment` (`book_appointment`) | Invokes `appointment_agent` with `get_fee` and `log_appointment` (verifies `name`, `service`, `sessions`, `date` before saving to Google Sheets). |
| `other` | Returns a direct welcome message introducing BrightSmile Dental Clinic services. |

---

## 5. Streamed Execution Outputs & Tricky Customer Tests

### Standard Branch Tests (`stream_mode="updates"`)
| Input Message | Routed Intent | Specialist Output Summary |
| :--- | :---: | :--- |
| `"Can I pay by UPI and do senior citizens get a discount?"` | `faq` | Confirmed UPI payment acceptance and 10% senior citizen discount on general checkups. |
| `"I'm Hansitha, book 2 urgent teeth cleaning sessions for 30 Nov"` | `appointment` | Routed to `appointment` node, calculated `Rs 2000` total, and confirmed the booking. |
| `"Good morning!"` | `other` | Returned clinic greeting and overview of available assistance. |
| `"How much does a tooth extraction cost?"` | `faq` | Returned `Rs 1500` fee via `get_fee`. |

### Tricky & Bilingual Customer Tests
| Tricky Input | Routed Intent | Observed Behavior & Takeaway |
| :--- | :---: | :--- |
| `"I want to know if I can get 2 teeth cleaning sessions done tomorrow"` | `appointment` | Classified as `appointment` (booking intent) and politely asked for the patient's name before logging. |
| `"hi"` | `other` | Routed cleanly to the `other` greeting node. |
| `"Teeth cleaning ki entha cost avutundi and Sunday open untunda?"` | `faq` | Handled Telugu-English mixed input accurately (`Rs 1000` fee and Sunday closure). |

---

## 6. "Your 50%" Hands-On Work & Observations
1. **Paper Sketch vs. Graph Visualization:** Sketched the workflow before coding and verified via `graph.get_graph().draw_mermaid_png()` that `START -> classify` branches cleanly into `appointment`, `faq`, and `other`, each converging to `__end__`.
2. **Stateless Per-Turn Behavior:** Confirmed that this Day 6 graph executes each message independently without a checkpointer (checkpointer and owner approval pause are scheduled for Day 7).
3. **Language Consistency Note:** Observed one English appointment prompt receiving a Telugu reply; tightening language-matching rules in `APPOINTMENT_PROMPT` ensures strict English-to-English and Telugu-to-Telugu alignment.

---

## 7. How to Run
1. Run the setup, `ChatGroq`, Google Sheets (`BrightSmile Appointments`), Day 4 tools, and Day 5 ChromaDB (`16 facts stored`) cells in Google Colab.
2. Run **Cell 1** (`ClinicState` & `Intent`), **Cell 2** (`faq_agent` & `appointment_agent`), and **Cell 3** (graph compilation & Mermaid diagram display).
3. Run **Cell 4** and the tricky customer test loop to inspect streamed node updates across all branches.
