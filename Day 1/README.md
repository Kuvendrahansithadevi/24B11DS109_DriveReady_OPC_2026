# Task 1: Constrained LLM Assistant & Prompt Guard

## Overview
This task covers two parts using the Groq API:
1. **Constrained Domain Assistant:** Built with `langchain_groq` (`ChatGroq`, model `openai/gpt-oss-120b`, `temperature=0`) acting as the front-desk assistant for **BrightSmile Dental Clinic** under strict facts, rules, and a 60-word limit.
2. **Prompt Injection Detection:** Uses the `groq` SDK with `meta-llama/llama-prompt-guard-2-22m` to test whether an input attempts to override system instructions.

## Setup & API Key
- Install packages: `pip install langchain_groq groq`
- Add `GROQ_API_KEY` in Google Colab **Secrets** (enable **Notebook access**). The script fetches it via `google.colab.userdata` and sets `os.environ["GROQ_API_KEY"]`.

## What the Code Does
- **System Prompt (`BrightSmile Dental Clinic`):** Sets clinic hours (Mon–Sat, 9 AM–7 PM), appointment and emergency slot rules, blocks diagnoses/prescriptions/unknown pricing, and enforces responses under 60 words.
- **`ask(question)` & Test Queries:** Sends system and human messages to `llm.invoke(...)` and tests walk-in pain/medication, Sunday availability, and bleeding gums/pricing queries.
- **Prompt Guard Check:** Sends a prompt-injection test string to `meta-llama/llama-prompt-guard-2-22m` and prints the result.

