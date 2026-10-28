# Enterprise Structured Ingestion Firewall

A small, beginner-friendly AI project. You type a customer-support message in a chat page, and the app:

1. **Week 1 – Understands it.** An LLM sorts the message into a fixed category (payment issue, order tracking, greeting…) and returns structured JSON. Pydantic checks that JSON before anything else uses it.
2. **Week 2 – Answers it.** A second LLM call uses that checked JSON to write a helpful reply in Markdown, which the chat page displays.

The "firewall" idea: the AI's output is not trusted until it passes a strict schema. If the model returns a category that doesn't exist or a missing field, the app rejects it.

## What you will learn

- Building an API with **FastAPI**
- Calling a cloud LLM (**Groq**) from Python
- Getting **structured output** from an LLM with **Instructor**
- **Validating** data with **Pydantic**
- Connecting a plain **HTML/CSS/JavaScript** frontend to a backend

## Key words (quick glossary)

| Word | Meaning in this project |
|---|---|
| **LLM** | Large Language Model, the AI that reads and writes text. We use one hosted by Groq. |
| **API** | The backend URL the frontend sends messages to (`/api/analyze`). |
| **FastAPI** | The Python library that runs our API. |
| **Pydantic** | Checks that data has the right fields and allowed values. |
| **Instructor** | Makes the LLM reply in a shape that matches our Pydantic model. |
| **Schema** | The rules for what valid data looks like. |
| **Intent** | What the user wants, e.g. `payment_issue`. |
| **`.env` file** | A private file holding your API key. Never share or commit it. |
| **venv** | A private Python environment just for this project. |

## How a message flows through the app

```text
 You type a message in the browser (frontend, port 5500)
        │
        ▼
 POST /api/analyze  ──►  FastAPI backend (port 8000)
                              │
                              ├─ 1. Check the input        (schemas/ingestion.py)
                              ├─ 2. WEEK 1: classify it    (services/llm_service.py)   ── LLM call #1
                              │       └─ Pydantic validates the JSON
                              ├─ 3. Count tokens (estimate) (utils/token_counter.py)
                              └─ 4. WEEK 2: write a reply  (services/response_service.py) ── LLM call #2
                              │
        ◄──────── JSON: intent details + response_markdown
        │
 Chat page shows the reply, with "Detected Intent" details underneath
```

## What you need

- **Python 3.10 or newer** (check with `python --version`)
- A free **Groq API key**: https://console.groq.com/keys
- Internet connection

## Run it in 5 steps (Windows PowerShell)

You need **two terminals**: one for the backend, one for the frontend.

### Step 1: Set up the backend (one time only)

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

You'll know the venv is active when your prompt starts with `(.venv)`.

### Step 2: Add your API key (one time only)

```powershell
copy .env.example .env
notepad .env
```

Make the file look like this, then save:

```env
GROQ_API_KEY=gsk_your_real_key_here
GROQ_MODEL=openai/gpt-oss-120b
```

> Only replace the API key. The model is already set. If you ever see `404 model_not_found`, the model was retired; see [backend/README.md](backend/README.md#troubleshooting) to pick another.

### Step 3: Start the backend (Terminal 1)

Still inside `backend`, with `(.venv)` showing:

```powershell
uvicorn app.main:app --reload --port 8000
```

Check it at http://127.0.0.1:8000/health. You should see `{"status":"ok",...}`. Leave this terminal running.

### Step 4: Start the frontend (Terminal 2)

Open a **new** terminal in the **project root** (the folder that contains `backend` and `frontend`):

```powershell
python -m http.server 5500 --bind 127.0.0.1 --directory frontend
```

### Step 5: Open the app

Go to http://127.0.0.1:5500. The sidebar should say **Backend Connected**. Click a suggested prompt or type your own message.

To stop the servers, press `Ctrl + C` in each terminal.

On macOS/Linux, use `source .venv/bin/activate` instead of `Activate.ps1`, and `cp` instead of `copy`.

## Common problems

| What you see | Why | Fix |
|---|---|---|
| Page has no styling and says "Checking backend..." | Frontend server started from the wrong folder, or two servers on port 5500 | Stop all servers (`Ctrl + C`), start **one** from the project root, then press `Ctrl + F5` |
| `404 File not found` on port 5500 | You ran `http.server` inside `backend` | Run it from the project root, or use `--directory ..\frontend` |
| `{"detail":"Not Found"}` at http://127.0.0.1:8000/ | Normal: there is no `/` route | Use `/docs` or `/health` |
| "Backend Disconnected" in the sidebar | Backend isn't running | Do Step 3 |
| `502 LLM request failed` | Wrong key, or the model name doesn't exist | See [backend/README.md](backend/README.md#troubleshooting) |
| `Activate.ps1 cannot be loaded` | PowerShell blocks scripts | Run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once |

## Project map

```text
enterprise-ingestion-firewall/
├── backend/                     ← Python API (see backend/README.md)
│   ├── app/
│   │   ├── main.py              ← routes: /health and /api/analyze
│   │   ├── config.py            ← reads your .env file
│   │   ├── schemas/
│   │   │   ├── ingestion.py     ← Week 1 data rules (input + intent result)
│   │   │   └── assistant.py     ← Week 2 data rules (adds response_markdown)
│   │   ├── services/
│   │   │   ├── llm_service.py   ← Week 1: classify the message
│   │   │   └── response_service.py ← Week 2: write the answer
│   │   └── utils/token_counter.py ← estimates token count
│   ├── tests/test_ingestion.py
│   ├── .env.example             ← copy this to .env
│   └── requirements.txt
├── frontend/                    ← chat page (see frontend/README.md)
│   ├── index.html
│   ├── style.css
│   └── script.js
└── README.md                    ← you are here
```

## Good to know

- Each message makes **two** LLM calls (classify, then answer).
- The app does **not** really look up orders or reset passwords; it only classifies and gives advice.
- Token counts are **estimates**, not what Groq bills.
- Schema validation checks the **shape** of data, not whether the AI's answer is true.

## Next reading

- [backend/README.md](backend/README.md): how each Python file works, the API, tests, troubleshooting
- [frontend/README.md](frontend/README.md): how the chat page works
