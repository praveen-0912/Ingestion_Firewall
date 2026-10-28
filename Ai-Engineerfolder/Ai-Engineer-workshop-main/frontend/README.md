# Frontend

A chat-style web page for the AI Assistant. It's plain **HTML, CSS and JavaScript**, so there's nothing to install or build. It sends your message to the backend and shows the AI's answer.

## Run it

1. Make sure the **backend is running** on port 8000 (see [../backend/README.md](../backend/README.md)).
2. Open a terminal in the **project root** (the folder that contains `frontend`) and run:

   ```powershell
   python -m http.server 5500 --bind 127.0.0.1 --directory frontend
   ```

3. Open http://127.0.0.1:5500

If you are inside the `backend` folder instead, use `--directory ..\frontend`.

Stop it with `Ctrl + C`. Run only **one** frontend server at a time.

## Using the page

- **Type a message** and press **➤**, or click a **Suggested Prompt** in the sidebar.
- The answer appears as formatted text (headings, bullet points).
- Click **Detected Intent (Week 1)** under an answer to see the classification JSON (intent, risk, priority…).
- **＋ New Chat** clears the conversation.
- The sidebar shows **Backend Connected** or **Backend Disconnected**.

## How it works

```text
Page loads  →  checkBackend() calls GET /health  →  sidebar shows Connected / Disconnected

You send a message
   →  addUserMessage()      shows your bubble
   →  "AI is thinking..."
   →  fetch POST /api/analyze  { "message": "..." }
   →  addAssistantMessage() renders response_markdown
                            + adds the "Detected Intent" box
```

### The three files

| File | What it does |
|---|---|
| `index.html` | The layout: sidebar (prompts, backend status) and chat panel (messages, input box). Also loads two libraries from a CDN. |
| `style.css` | All the styling. Hides the sidebar on small screens (under 760px). |
| `script.js` | All the behaviour: sending messages, showing replies, errors, New Chat, and the backend health check. |

### The two libraries

| Library | Why |
|---|---|
| **marked** | Turns the AI's Markdown (`## Heading`, `- bullet`) into HTML. |
| **DOMPurify** | Cleans that HTML so nothing harmful (like a `<script>`) can run. Always sanitize AI output before putting it on a page. |

If the CDN can't load (no internet), the answer is shown as plain text instead.

## Changing the backend address

At the top of `script.js`:

```js
const API_URL = "http://127.0.0.1:8000/api/analyze";
const HEALTH_URL = "http://127.0.0.1:8000/health";
```

Change both if your backend runs on another port.

## Common problems

| What you see | Why | Fix |
|---|---|---|
| No styling, plain text page, stuck on "Checking backend..." | `style.css` and `script.js` didn't load. Usually two servers are on port 5500, or one was started in the wrong folder | Stop all servers, start one from the project root, press `Ctrl + F5` |
| `404 File not found` | Server is serving the wrong folder | Check the `--directory` path |
| "Backend Disconnected" | Backend isn't running | Start `uvicorn` in `backend` |
| Red error: "LLM request failed" | Backend can't reach the AI (key or model problem) | See backend troubleshooting |
| Red error: "missing response_markdown" | Backend is the old Week 1 version | Make sure `main.py` returns `AssistantResult` |

**Tip:** press `F12` → **Console** to see errors, and **Network** to see each request and its response.
