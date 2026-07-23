# TDS Project 1 — Q5: Data-Analyst Telegram Bot (37.5 marks) — Complete Guide

A step-by-step guide to building, deploying, and registering the data-analyst
Telegram bot. This walks through the **approach** — you write your own code and
run your own bot.

> **Working reference implementation:** https://github.com/angadseth/tds-p1-databot

---

## 1. Understand what is being graded

The grader is a **real Telegram user account** (not a bot) that messages your
bot data-analysis questions. Your bot must reply to **every** message with
**exactly one JSON object and nothing else**:

```json
{"answer": <shaped exactly as the question asks>, "log_url": "https://your-host/run.jsonl"}
```

Key facts (from the public grading pipeline —
[Jivraj-18/tds-p1-t2-2026-telegram-bot](https://github.com/Jivraj-18/tds-p1-t2-2026-telegram-bot)):

- **It waits for a reply after each message**, even in multi-turn sequences
  (`collect.py` does `send_message` → `get_response()` in a loop). So reply to
  *every* message, not just the last one.
- **Timeout is ~300 seconds per question.** Your agent must always answer
  within that, even if a dataset download fails.
- **Answers are compared exact-match** against a key computed per student
  (inputs are seeded from `email + question id`). Any prose around the JSON =
  `format_error`. No markdown fences, no "Here is the answer:", nothing.
- Questions spell out the exact JSON shape they want — e.g.
  `{"answer": {"state": "<state name>"}, "log_url": "..."}`. The inner shape
  changes per question; match it exactly (keys, nesting, number vs string).
- `log_url` must be a **public, wget-able URL** to a JSONL log of your agent's
  run (one JSON object per line). A route on your own server is fine.

## 2. Create the Telegram bot (2 minutes)

1. In Telegram, open **@BotFather** → send `/newbot`.
2. Give it any display name, then a username **ending in `bot`**
   (e.g. `yourname_databot`).
3. BotFather returns an HTTP API token like `1234567890:AAE...`. Keep it
   secret — anyone with it controls your bot.

No webhook setup is needed if you use long polling (recommended — works from
any host, no HTTPS certificate hassle).

## 3. Architecture that works

One small Python file is enough. Three pieces run in one process:

```
FastAPI web app ──► GET /health        (keep-alive + sanity check)
                └─► GET /run.jsonl     (the public agent log)

Background thread ──► Telegram getUpdates long-poll loop
                      └─► per-message: agent loop → sendMessage(JSON)

Background thread ──► self-ping /health every 10 min (free hosts idle out)
```

### The agent loop

Use any OpenAI-compatible chat API with **function calling** and give the model
one tool:

- `run_python(code)` — `exec()` the code server-side, capture stdout, return
  it (cap the output, e.g. last 8000 chars). Install pandas, numpy, requests,
  BeautifulSoup, openpyxl so the model can download and analyse public
  datasets (MOSPI publishes XLSX/CSV/HTML tables).

Loop: send conversation → if the model calls the tool, run it, append the
result, repeat (cap at ~10 steps) → when the model answers with plain text,
parse the JSON out of it.

### System prompt essentials

Tell the model to:
1. Answer the **latest** message; earlier messages are context (multi-turn).
2. Use `run_python` to fetch/compute — never guess a number it can compute.
   For published statistics where fetching fails, answer from knowledge.
3. Output **only** the JSON object the question asks for — no prose, no
   fences; put a placeholder in `log_url` (your code substitutes the real URL).
4. Match the requested `answer` shape exactly; never add extra keys.
5. If a mid-conversation message is only setup ("I'll send data next"), still
   reply with a small JSON ack — the grader waits for a reply to every message.

### Defensive layers in code (these save marks)

- **JSON extraction:** strip code fences, find the first *balanced* `{...}`
  in the model output, `json.loads` it. If there's no `"answer"` key, wrap it:
  `{"answer": parsed}`. Always overwrite `log_url` with your real URL.
- **Wall-clock budget:** track a deadline (~210 s). Past it, disable tools and
  force the model to answer immediately. A late perfect answer scores zero.
- **Per-chat history:** keep the last ~20 turns per `chat_id` so multi-turn
  questions have context.
- **Never crash silently:** wrap the handler in try/except and reply
  `{"answer": "internal error", "log_url": ...}` as a last resort — a reply
  that parses beats a timeout.
- **Log everything as JSONL:** timestamp, question, every tool call + output,
  final reply. Serve the file at `/run.jsonl`. This doubles as your `log_url`
  *and* your debugging tool.

## 4. Pick a strong enough model (this matters!)

We tested the worked example ("Which state has the highest maternal mortality
rate based on MOSPI data?" — correct answer per SRS: **Assam**):

| Model | Answer | Verdict |
|---|---|---|
| gpt-4o-mini | Madhya Pradesh | ❌ |
| gpt-4.1-mini | Bihar | ❌ |
| **gpt-4o** | **Assam** | ✅ |

Small/cheap models confidently get real-world statistics **wrong**. Use a
frontier-class model (gpt-4o or better). The cost is negligible for the
number of graded questions.

Also make sure your API credentials **won't expire before grading** — grading
happens after the deadline. A weekly-expiring proxy token will silently kill
your bot; a direct API key is safer.

## 5. Deploy (Render free tier works)

1. Push your code to a **public GitHub repo** (required for grading).
2. Create a Render **Web Service** from the repo:
   - Build: `pip install -r requirements.txt`
   - Start: `uvicorn bot:app --host 0.0.0.0 --port $PORT`
   - Env vars: `BOT_TOKEN`, your LLM API key, `BASE_URL=https://<service>.onrender.com`
3. Free instances **spin down after ~15 min idle** and a cold start can eat
   the grader's patience. Fix: a thread inside the app requests its own public
   `/health` URL every 10 minutes. (An external pinger like UptimeRobot also
   works.)
4. Remember: on Render, changing env vars does **not** restart the service —
   trigger a deploy afterwards.

Verify after deploying:

```bash
curl https://<your-host>/health      # {"ok": true, ...}
wget https://<your-host>/run.jsonl   # must download publicly
```

## 6. Test like the grader tests

- Message your bot from your own Telegram account (you're a user account —
  exactly what the grader is). Send the worked example and check you get one
  clean JSON back.
- Clone the public pipeline repo, point it at your bot, and add your own
  questions to `evals/questions.json` for a full dress rehearsal.
- Test a multi-turn flow: send "I will send data next.", then the data +
  question. The bot must reply to both.
- `wget` your `log_url` from a different network to confirm it's truly public.

## 7. Register on SEEK

One box, both values, comma-separated:

```
https://github.com/<you>/<your-repo>, your_bot_username
```

- Repo URL first, then the bot **username** (no `@`), which must end in `bot`.
- Passing validation auto-awards 0.1 marks; the rest is graded after the
  deadline from your live bot + repo.
- Press **Check**, then **Save**.

## 8. Checklist before you walk away

- [ ] Bot replies to a fresh Telegram message with exactly one JSON object
- [ ] `answer` shape matches whatever the message asked for
- [ ] `log_url` in the reply is wget-able and shows the run you just did
- [ ] Multi-turn: bot replies to *every* message
- [ ] Reply always arrives well under 300 s (test a hard question)
- [ ] Repo is public; no secrets committed (tokens live in env vars only)
- [ ] Host stays awake (keep-warm ping working)
- [ ] LLM credentials will still be valid weeks from now
- [ ] Registered on SEEK, Checked, **Saved**

## Common failure modes

| Symptom | Cause |
|---|---|
| `format_error` in grading | Prose/fences around the JSON, or two messages sent |
| `timeout` | Cold-started host, slow dataset fetch with no answer budget |
| Wrong answers on stats questions | Model too small — upgrade it |
| Bot dead at grading time | Expired API token, or free host asleep |
| Multi-turn question scored zero | Bot only replied to the last message |
| `bad_bot` | Wrong username registered / bot never started |

Good luck! 🚀
