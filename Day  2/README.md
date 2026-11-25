# Task 2: Stateful Multi-Turn Assistant Using OpenAI-Compatible Groq API

## 1. What the Task Is
The objective of this task is to build a stateful, multi-turn conversational assistant using the standard `openai` Python SDK connected to Groq's OpenAI-compatible endpoint (`https://api.groq.com/openai/v1`). Instead of single-turn prompts, the assistant must retain conversation context, use structured service/fee data, log helper function calls, handle API errors gracefully, and run interactively in a console loop.

## 2. Systems & Setup Chosen
- **Use-Case / Domain System Selected:** Front-desk virtual assistant for **BrightSmile Dental Clinic**.
- **Execution Environment:** Google Colab (Python 3).
- **API & Model Setup:** Groq API accessed via the `openai` Python client (`base_url="https://api.groq.com/openai/v1"`) using `model="openai/gpt-oss-120b"` at `temperature=0.3`.
- **Authentication:** `GROQ_API_KEY` stored securely in Google Colab Secrets and loaded via `google.colab.userdata`.
- **Libraries Used:** `openai` (`OpenAI`, `RateLimitError`, `APIConnectionError`), `json`, and `functools`.

## 3. How It Was Implemented
1. **Connected OpenAI Client to Groq:** Initialized the `OpenAI` client with `userdata.get("GROQ_API_KEY")` and `base_url="https://api.groq.com/openai/v1"` so standard OpenAI chat completion calls execute on Groq.
2. **Structured Service Data & JSON Handling:** Defined the clinic's `SERVICES` fee dictionary (`general checkup`: Rs 300, `teeth cleaning`: Rs 1000, `tooth extraction`: Rs 1500), practiced safe lookups using `.get()`, and serialized/deserialized data with `json.dumps()` and `json.loads()`.
3. **Designed the Constrained System Prompt:** Injected the JSON-formatted `SERVICES` fees into `SYSTEM` along with clinic timings (Mon–Sat, 9 AM–7 PM), appointment rules, strict blocks on medical diagnoses/prescriptions, a fallback response (`"I'll check with the doctor."`), patient memory instructions, and bilingual matching (English/Telugu under 50 words).
4. **Added a Logged Helper Function:** Built `get_fee(service, sessions)` wrapped with a custom `@log_call` decorator (`functools.wraps`) to print function arguments and return values during execution.
5. **Implemented Sliding-Window Memory & Error Handling:** Stored messages in `history` and restricted API payloads to `[history[0]] + history[1:][-MAX_TURNS:]` (`MAX_TURNS = 8`) to preserve the system prompt and recent context, while catching `RateLimitError` and `APIConnectionError` with fallback replies.
6. **Built the Interactive Chat Loop:** Created a `while True` loop reading user queries via `input("You: ")`, generating replies through `chat(text)`, and terminating cleanly when the user types `bye`, `exit`, or `quit`.

## 4. How to Run
1. Add `GROQ_API_KEY` in Google Colab **Secrets** and enable **Notebook access**.
2. Open the Task 2 notebook in Google Colab and run all cells sequentially.
3. Enter questions at the `You:` prompt and type `bye` to end the chat.
