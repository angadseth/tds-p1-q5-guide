# TDS Project 1 — Complete Guide (May 2026 term)

Step-by-step method guides for **all five questions** of Tools in Data Science
Project 1 (100 marks total). These explain the *approach* and the traps —
you do your own work and run your own submissions.

| Q | Task | Marks | Guide |
|---|------|-------|-------|
| 1 | Requirements Interview (Drive folder: questions.md + recording.mp3) | 12.5 | [Q1.md](Q1.md) |
| 2 | Differentiating Model Intelligence (one prompt, weak fails / strong passes) | 12.5 | [Q2.md](Q2.md) |
| 3 | AI Agent: GCS Bucket Setup | 12.5 | [Q3.md](Q3.md) |
| 4 | AI Agent: Dataset Upload to GCS (SHA-256 verified) | 12.5 | [Q4.md](Q4.md) |
| 5 | Data-Analyst Telegram Bot (LLM agent) | 37.5 | [Q5.md](Q5.md) |

## The five rules that apply to everything

1. **Read the exact contract.** Every question states precisely what the
   grader checks — a bucket name, a JSON shape, a file count, a time window.
   Exact-match beats "close enough" every time.
2. **Verify like the grader, not like a developer.** Test anonymously
   (`curl` without auth), from outside, against the public URL — that's what
   the server sees. "Works on my machine with my login" proves nothing.
3. **Half the marks are offline.** Q3/Q4 show max 6.25/12.5 on Check, and
   Q5 shows 0.1 — that is the **full visible score**. The rest is graded
   after the deadline from your logs/repo/live bot. Don't panic-debug a
   "half score" that is actually perfect.
4. **Keep everything alive until results.** Buckets, bot hosting, API
   credentials — grading happens *after* the deadline. An expired token or a
   deleted bucket turns full marks into zero silently.
5. **Check, then SAVE.** Check only evaluates; Save records. Every question,
   every time.

Good luck! 🚀
