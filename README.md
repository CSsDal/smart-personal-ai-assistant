# Smart Personal AI Assistant

A multi-agent personal assistant (Google Calendar + Gmail + internal Knowledge Base) built on
**LangGraph's Functional API** (`@task` / `@entrypoint`), with retrieval-augmented generation,
real human-in-the-loop approval, short-term + long-term memory, and LangSmith tracing.

**Name** _: Dalal Albaijan_

## What's in this repo

- `Smart_Personal_AI_Assistant.ipynb` — the full notebook (Google Colab)
- `README.md` — this file

## Architecture

A **Routing**-pattern supervisor reads each user message and dispatches it to exactly one of
three specialist agents:

- **Calendar agent** — list / create / search / update calendar events, and *propose* deletions
- **Email agent** — list / read unread email, and *propose* sending a new email
- **Knowledge agent** — Agentic RAG over a 6-document internal knowledge base (usage policy +
  FAQs), retrieved via `InMemoryVectorStore` + free `HuggingFaceEmbeddings`

Everything is orchestrated by a single `@entrypoint` function (`assistant_workflow`) that calls
`@task`-decorated steps (`route_decision_task`, `calendar_task`, `email_task`,
`knowledge_task`, `confirm_action_task`) — there is no `StateGraph` in this project.

### Human-in-the-loop

Deleting a calendar event or sending an email are never executed directly by the agents. They
only ever *propose* the action; the workflow then calls `confirm_action_task`, which pauses the
whole run with a real `interrupt()`. The caller inspects the paused proposal and resumes the
same run with `Command(resume="yes")` or `Command(resume="no")` — only a "yes" actually triggers
`delete_calendar_event` / `send_email`.

### Memory

- **Short-term:** `MemorySaver`, keyed by `thread_id` — the running conversation within one chat.
- **Long-term:** `InMemoryStore`, keyed by `user_id` — durable facts (e.g. preferred meeting
  length) that persist even into a brand-new `thread_id` for the same user. Demonstrated in
  Module 15 of the notebook.

### Error handling

- `calendar_task` carries a `RetryPolicy(max_attempts=3)` for transient Google Calendar API
  failures.
- `route_decision_task` raises an **uncaught** `ValueError` if the supervisor produces a
  decision that isn't `calendar` / `email` / `knowledge`, so a broken routing decision fails
  loudly instead of being silently swallowed.

## Setup (Google Colab)

1. Open `Smart_Personal_AI_Assistant.ipynb` in Google Colab.
2. In Colab's Secrets panel (🔑 icon), add:
   - `GOOGLE_API_KEY` — a Gemini API key from https://aistudio.google.com
   - `LANGSMITH_API_KEY` — a free key from https://smith.langchain.com
3. Upload your Google Cloud OAuth `client_secret_*.json` (Calendar + Gmail scopes enabled) when
   prompted in Module 2, and complete the OAuth flow once to produce `credentials/token.json`.
4. Run all cells top to bottom.

## LangSmith trace observation

The two runs that hit `interrupt()` inside `confirm_action_task` (asking to delete a calendar
event) show by far the highest latency in the project — 40.02s and 21.49s, versus 2–3s for every
other run — which is just the wall-clock time the run sat paused waiting for
`Command(resume=...)`, not actual compute. Excluding those, `knowledge_base_search_tool` is
essentially free (0.03s), and almost all of the 2–2.8s latency on a normal request comes from the
Gemini calls inside the react agents, consistent with `create_react_agent` making at least two
model calls per turn (tool selection, then response generation).

## Rubric write-up

See the final markdown cell of the notebook ("Module 17 — Rubric Write-up") for a paragraph per
rubric section (RAG, Functional API, human-in-the-loop, long-term memory, LangSmith, workflow
pattern).
