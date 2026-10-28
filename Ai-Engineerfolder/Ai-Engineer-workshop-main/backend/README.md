# Backend

The Python part of the project. It receives a message, asks an LLM to **classify** it (Week 1), validates the result with Pydantic, then asks the LLM to **write an answer** (Week 2).

All commands below are run from the `backend` folder with the virtual environment active (your prompt shows `(.venv)`).

## Setup (one time)

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
notepad .env
```

Your `.env` should contain:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

- No quotes and no spaces around `=`.
- Save it as `.env`, not `.env.txt`.
- Never share it or push it to GitHub (it's already in `.gitignore`).

If `Activate.ps1` is blocked, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.

## Run

```powershell
uvicorn app.main:app --reload --port 8000
```

`--reload` restarts the server automatically when you save a Python file.

| Open this | What you get |
|---|---|
| http://127.0.0.1:8000/docs | Interactive page to try the API with no frontend needed |
| http://127.0.0.1:8000/health | `{"status":"ok",...}` if the server is up |
| http://127.0.0.1:8000/ | `{"detail":"Not Found"}`. This is normal; there is no `/` route. |

Stop the server with `Ctrl + C`.

## How the code works, file by file

Read them in this order; each builds on the one before.

### 1. `app/config.py`: settings

Loads `GROQ_API_KEY` and `GROQ_MODEL` from your `.env` file so the rest of the code can import them.

### 2. `app/schemas/ingestion.py`: Week 1 data rules

Defines what valid data looks like, using Pydantic:

- **`IngestionRequest`**: what the frontend sends. One field, `message`, 1–5000 characters. Spaces at the ends are trimmed; a message of only spaces is rejected.
- **`IngestionResult`**: what the LLM must return. Only these values are allowed:

| Field | Allowed values |
|---|---|
| `intent` | `greeting`, `order_tracking`, `payment_issue`, `current_time`, `general_question`, `unknown` |
| `risk_profile` | `low`, `medium`, `high`, `unknown` |
| `priority` | `low`, `medium`, `high` |
| `requires_action` | `true` / `false` |
| `summary` | text, 1–500 characters |

`Literal[...]` is what restricts a field to a fixed list. `extra="forbid"` rejects any field we didn't define.

### 3. `app/services/llm_service.py`: Week 1, classify the message

- A **system prompt** tells the LLM what each intent and risk level means.
- **Instructor** wraps the Groq client so the reply must fit `IngestionResult`.
- `temperature=0` gives consistent answers; `max_retries=2` lets Instructor ask again if the reply fails validation.

### 4. `app/utils/token_counter.py`: estimate tokens

Counts roughly how many tokens the message is, using `tiktoken`. It's an estimate only, not Groq's real billing number.

### 5. `app/schemas/assistant.py`: Week 2 data rules

**`AssistantResult`** = everything in `IngestionResult` **plus** `response_markdown` (the written answer). It inherits from `IngestionResult`, so we don't repeat the fields.

### 6. `app/services/response_service.py`: Week 2, write the answer

- Takes the **validated** Week 1 result (intent, summary, risk, priority).
- Sends it to the LLM with a support-assistant prompt (be clear, never ask for passwords/OTPs, don't pretend to take actions).
- Returns the answer as Markdown. `temperature=0.3` lets the wording vary a little so it sounds natural.

### 7. `app/main.py`: the API

Connects everything:

```text
POST /api/analyze
  → IngestionRequest checks the message
  → analyze_message()      (Week 1)
  → estimate_tokens()
  → generate_response()    (Week 2)
  → returns AssistantResult
```

Any error in the LLM calls becomes HTTP `502`.

## API reference

### `POST /api/analyze`

Send:

```json
{ "message": "My payment was deducted twice. What should I do?" }
```

Get back (wording varies):

```json
{
  "intent": "payment_issue",
  "risk_profile": "high",
  "priority": "high",
  "requires_action": true,
  "summary": "User reports a duplicate payment deduction and asks what to do.",
  "token_metadata": {
    "estimated_input_tokens": 11,
    "note": "Estimated locally; may differ from provider token counts."
  },
  "response_markdown": "## Duplicate charge\n\n- Check your bank statement...\n"
}
```

| Status code | Meaning |
|---|---|
| `200` | Success |
| `422` | Bad input (empty message, too long, or extra fields) |
| `502` | The LLM call failed (see troubleshooting) |

Try it from PowerShell without the frontend:

```powershell
Invoke-RestMethod -Method Post -Uri http://127.0.0.1:8000/api/analyze `
  -ContentType "application/json" `
  -Body '{"message":"Where is my order?"}'
```

### `GET /health`

Returns `{"status": "ok", "message": "Ingestion Firewall API is running"}`. Doesn't call the LLM, so it's a quick way to check the server is up.

## Tests

```powershell
pytest -q
```

Expected: `4 passed`. They check the Week 1 Pydantic rules (trimming, blank messages, invalid intent, valid data). They don't call the LLM, so they work without an API key or internet.

## Troubleshooting

### `502 LLM request failed`

The API hides the real reason, so call the Week 1 service directly to see it:

```powershell
python -c "from app.services.llm_service import analyze_message; print(analyze_message('Hi'))"
```

| Error you see | Meaning | Fix |
|---|---|---|
| `401` / invalid API key | Key is wrong or expired | Fix `GROQ_API_KEY` in `.env` |
| `404 model_not_found` | That model doesn't exist anymore | Set `GROQ_MODEL=openai/gpt-oss-120b` |
| `GROQ_API_KEY is missing` | `.env` not found | Make sure the file is `backend\.env`, not `.env.txt` |
| Connection / timeout | No internet or Groq is down | Check your connection |

List the models your key can use:

```powershell
python -c "from groq import Groq; from app.config import GROQ_API_KEY; [print(m.id) for m in Groq(api_key=GROQ_API_KEY).models.list().data]"
```

Restart `uvicorn` after changing `.env`.

### Port 8000 already in use

An old server is still running. Press `Ctrl + C` in its terminal, or check what's using the port:

```powershell
Get-NetTCPConnection -LocalPort 8000 -State Listen
```

## Known limitations (good discussion topics)

- If Instructor runs out of retries, it raises `InstructorRetryException`, not `ValidationError`, so you get the generic 502 instead of the detailed validation message.
- `token_metadata` is part of the schema sent to the LLM in Week 1, so the model may fill it in; the code overwrites the count afterwards.
- A new Groq client is created on every request.
- CORS allows every website (`*`). Fine for learning, but restrict it before deploying.
- Tests don't cover the `/api/analyze` route or Week 2.
