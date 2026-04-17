# CLAUDE.md — Noto

Noto (ノート) is a web app for APU (Ritsumeikan Asia Pacific University) students to generate structured notes and Anki flashcards from lecture transcripts or audio recordings. Built by Shimul Al Imran using AI-assisted development (Claude Code).

**Guiding Principle:** Keep it lean, keep it free. Prefer client-side execution over server compute. Stay within free tiers at all times.

**Tagline:** 2 clicks from lecture to flashcards.

---

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

---

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

---

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

---

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

## 5. Build Rules

- Build sessions strictly in order from `docs/build-order.md`. Do not jump ahead.
- Never build features listed in `docs/not-included.md` under any circumstances.
- Do not suggest features not in `docs/features.md` as improvements.
- Commit to GitHub after each session with a descriptive commit message.
- Test every acceptance criterion in `docs/features.md` before marking a feature complete.
- All Groq API calls are server-side only — never call Groq from the client.
- All exports (PDF, DOCX, TXT, Anki .apkg) are client-side only — never use a server route for exports.
- Never expose `SUPABASE_SERVICE_ROLE_KEY` or `GROQ_API_KEY` to the client.
- Use Supabase JS client directly for all DB operations not requiring service role key.
- RLS is enabled on all tables. Never disable it.

---

## 6. Doc Imports

@docs/stack.md
@docs/schema.md
@docs/features.md
@docs/not-included.md
@docs/build-order.md
@docs/api-routes.md
@docs/env.md
@docs/prompts.md

---

## 7. Quick Reference

**Stack:** Next.js 15 (App Router) + TypeScript + Tailwind + Supabase + Groq + Vercel

**Auth:** Supabase Auth only. Google OAuth + APU email (@apu.ac.jp). No Auth.js.

**API Routes:**
- `POST /api/auth/signup-apu` — APU email domain validation
- `DELETE /api/auth/delete-account` — deletes Supabase Auth user
- `POST /api/notes/generate` — note generation + regeneration
- `POST /api/flashcards/generate` — flashcard generation
- `POST /api/transcribe` — single audio chunk → Whisper
- `POST /api/parse/vision` — image → Groq vision → extracted text
- `POST /api/admin/metrics` — admin dashboard metrics
- `GET /api/admin/errors` — admin error log
- `GET /api/admin/users` — admin username search
- `POST /api/cron/cleanup` — delete expired sessions + old notifications

**Key Tables:** profiles, sessions, notes, flashcards, premium_signups, notifications, export_counts, error_logs, folders, tags, session_tags, session_folders

**Free vs Premium:**
| Feature | Free | Premium |
|---------|------|---------|
| Note + flashcard generation | Yes | Yes |
| Session history | 24hr | Indefinite |
| Note export (PDF/DOCX/TXT) | 6/day | Unlimited |
| Folders + tags | No | Yes |
| Audio recording + transcription | No | Yes |

**Environment Variables:** NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, GROQ_API_KEY, ADMIN_EMAIL, CRON_SECRET, NEXT_PUBLIC_APP_URL
