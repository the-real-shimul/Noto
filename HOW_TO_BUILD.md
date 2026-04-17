# How to Build Noto with Claude Code

This file tells you exactly how to start a new Claude Code session and build each session of Noto. Read this before every build session.

---

## The Mental Model

Claude Code reads `CLAUDE.md` automatically when you open a session inside the `noto/` folder. That file imports all 8 docs. So when you start a session, Claude Code already knows:
- The full tech stack
- The entire database schema
- Every feature and acceptance criteria
- What NOT to build
- Which session to build next
- Every API route
- Every LLM prompt

**Your job is to direct. Claude Code's job is to execute.**

---

## Before Every Session

**Step 1 — Open Claude Code inside the noto folder**

In your terminal:
```bash
cd "/Users/shimul/Documents/Claude Playground/noto"
claude
```

Or open the noto folder directly in your IDE and launch Claude Code from there.

**Step 2 — Confirm Claude Code has context**

Type this first:
```
What session are we on and what is the goal for this session?
```

Claude Code should respond with the correct session number and goal from `docs/build-order.md`. If it seems confused, say:
```
Read CLAUDE.md and all docs in the docs/ folder before we begin.
```

**Step 3 — Start the session**

Once context is confirmed, use the session starter prompt below.

---

## Session Starter Prompts

Copy and paste the relevant prompt at the start of each build session.

---

### Session 1 — Foundation

```
We are starting Session 1 of the Noto build. 

Goal: Authenticated shell deployed to Vercel. Students can sign in.

Please read docs/build-order.md Session 1 for the full task list.
Before writing any code, confirm your understanding of:
1. The two auth methods (Google OAuth + APU email)
2. The profiles table schema
3. What NOT to build this session

Then state a brief plan and begin.
```

---

### Session 2 — Core Feature: Notes

```
We are starting Session 2 of the Noto build.

Goal: Student can paste a transcript and receive structured notes.

Please read docs/build-order.md Session 2 for the full task list.
Before writing any code, confirm your understanding of:
1. The POST /api/notes/generate route spec
2. The notes table schema (5 sections including Details)
3. The note generation prompt in docs/prompts.md
4. The session expiry logic (free vs premium)

Then state a brief plan and begin.
```

---

### Session 3 — Flashcards + All Exports

```
We are starting Session 3 of the Noto build.

Goal: Full study and export pipeline working.

Please read docs/build-order.md Session 3 for the full task list.
Before writing any code, confirm your understanding of:
1. The POST /api/flashcards/generate route spec
2. The flashcard table schema (template, field_order, ease_factor)
3. The flashcard generation prompt in docs/prompts.md
4. The Anki ease factor mapping (Easy=2.7, Hard=1.8, Again=1.3, unrated=2.5)
5. The export_counts table and free user 6/day limit

Then state a brief plan and begin.
```

---

### Session 4 — File Upload

```
We are starting Session 4 of the Noto build.

Goal: All file input types supported alongside paste.

Please read docs/build-order.md Session 4 for the full task list.
Before writing any code, confirm your understanding of:
1. Which file types are parsed client-side vs server-side
2. The POST /api/parse/vision route spec
3. The image-based PDF fallback flow
4. The PPTX message to show users

Then state a brief plan and begin.
```

---

### Session 5 — Premium Gate + Audio Recording

```
We are starting Session 5 of the Noto build.

Goal: Premium flow and audio recording working end to end.

Please read docs/build-order.md Session 5 for the full task list.
Before writing any code, confirm your understanding of:
1. The fake premium gate (no Stripe — just name + email collected)
2. The audio_consent column on profiles
3. The POST /api/transcribe route spec
4. The IndexedDB queue architecture
5. The exponential backoff logic on 429
6. What NOT to build this session (notification bell UI)

Then state a brief plan and begin.
```

---

### Session 6 — Polish + Admin

```
We are starting Session 6 of the Noto build.

Goal: Folders, tags, notification bell, admin dashboard, and landing page.

Please read docs/build-order.md Session 6 for the full task list.
Before writing any code, confirm your understanding of:
1. The folders, tags, session_folders, session_tags schema
2. The notification bell uses Supabase Realtime (not Socket.io)
3. The admin dashboard is gated by ADMIN_EMAIL env var (returns 404 for others)
4. The three admin routes: metrics, errors, users

Then state a brief plan and begin.
```

---

### Session 7 — Beta + Iteration

```
We are starting Session 7 of the Noto build.

Goal: Real APU students use the app. Bugs fixed. Prompts tuned.

Tasks for this session:
1. Share the app link with 5-10 APU students
2. Monitor the /admin dashboard for errors and usage
3. Fix any bugs reported
4. Review actual note output quality and tune prompts in docs/prompts.md
5. Count premium signups for the Venture Entrepreneurship presentation

For any bug fix, share the error and I will diagnose and fix it.
For prompt tuning, share example note output and I will suggest improvements.
```

---

## Mid-Session Tips

**If Claude Code goes off-track:**
```
Stop. Re-read docs/not-included.md and docs/features.md. 
You are building [feature name] and nothing else. Continue.
```

**If Claude Code adds something unwanted:**
```
Revert that change. That feature is in docs/not-included.md.
```

**If Claude Code seems unsure about a spec:**
```
Check docs/api-routes.md for the exact route spec before proceeding.
```

**If a Groq call fails:**
```
Check docs/prompts.md for the exact prompt structure. 
Make sure the request matches the spec exactly.
```

**If auth seems broken:**
```
Check docs/stack.md auth section. We use Supabase Auth only.
No Auth.js. APU email validation happens in /api/auth/signup-apu.
```

---

## After Every Session

1. **Test every acceptance criterion** from `docs/features.md` for that session
2. **Check the /admin dashboard** for any error logs
3. **Commit to GitHub:**
```bash
git add .
git commit -m "Session X complete: [brief description]"
git push origin main
```
4. **Update your memory** -- note what worked, what needed fixing, any prompt changes made

---

## Key Rules Claude Code Will Follow

These are already in `CLAUDE.md` but good to know:

- Never builds features from `docs/not-included.md`
- Never calls Groq from the client (always server-side)
- Never exposes `SUPABASE_SERVICE_ROLE_KEY` or `GROQ_API_KEY` to the browser
- Never converts audio to WAV (uses native MediaRecorder format)
- All exports (PDF, DOCX, TXT, Anki) are client-side only
- RLS is enabled on all tables, never disabled
- Commits after each session

---

## If Something Goes Seriously Wrong

**Nuclear option — start the session fresh:**
```
Forget everything we discussed this session. 
Read CLAUDE.md and all docs in docs/ from scratch.
We are on Session [X]. Read docs/build-order.md for the task list.
State your understanding before writing any code.
```

**If a feature is fundamentally broken:**
```
Stop building new features. 
The [feature name] is broken. Here is the error: [paste error]
Diagnose the root cause before proposing a fix.
```

---

## Environment Setup Reminder

Before Session 1, make sure `.env.local` exists with all 7 variables:
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
GROQ_API_KEY=
ADMIN_EMAIL=
CRON_SECRET=
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

See `docs/env.md` for where to find each value.

---

## Quick Reference

| Doc | What it's for |
|-----|--------------|
| `CLAUDE.md` | Auto-loaded by Claude Code — rules + context |
| `docs/build-order.md` | What to build each session |
| `docs/features.md` | Acceptance criteria for every feature |
| `docs/not-included.md` | What to NEVER build |
| `docs/schema.md` | Database tables and columns |
| `docs/api-routes.md` | Every server route, request/response |
| `docs/prompts.md` | Groq LLM prompts |
| `docs/stack.md` | Tech stack decisions |
| `docs/env.md` | Environment variables |
