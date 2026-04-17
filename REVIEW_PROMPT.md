# Noto — Master Review Prompt for Claude Opus

## Instructions for Opus
You are reviewing the complete technical documentation for a web app called **Noto** (ノート). Your job is to act as a senior software engineer doing a pre-build audit. Find every oversight, inconsistency, missing piece, security hole, logical flaw, or incorrect assumption across the documentation below.

Be ruthless. Do not summarize what the docs say — only report problems. Structure your output as a numbered list of issues, each with:
- **Location:** which doc and section
- **Issue:** what is wrong
- **Impact:** why it matters
- **Fix:** what should change

Check specifically for:
1. Schema columns referenced in features or API routes that don't exist in the schema
2. API routes referenced in features that don't exist in api-routes
3. Features described in features.md not covered by any route or client-side logic
4. Prompt output fields that don't match schema column names
5. Free vs premium gating inconsistencies
6. Security holes (exposed keys, missing auth checks, bypassable premium gates)
7. Free tier limit violations (Groq, Supabase, Vercel)
8. Build order steps that depend on things not yet built
9. Contradictions between not-included.md and features.md
10. Missing edge cases in acceptance criteria that would cause bugs in production
11. Anything that would cause a Claude Code agent to make wrong assumptions

---

## Project Overview

**App:** Noto (ノート) — web app for APU (Ritsumeikan Asia Pacific University) students
**Purpose:** Generate structured notes and Anki flashcards from lecture transcripts or audio recordings
**Stack:** Next.js 15 (App Router) + TypeScript + Tailwind + Supabase + Groq API + Vercel
**Auth:** Supabase Auth only. Google OAuth + APU email (@apu.ac.jp). No Auth.js.
**Guiding principle:** Keep it lean, keep it free. Client-side execution preferred. Stay within free tiers.
**Built with:** Claude Code (AI-assisted development). Shimul directs, Claude Code executes.

**Free vs Premium:**
| Feature | Free | Premium |
|---------|------|---------|
| Note + flashcard generation | Yes | Yes |
| Session history | 24hr only | Indefinite |
| Note export (PDF/DOCX/TXT) | 6/day | Unlimited |
| Folders + tags | No | Yes |
| Audio recording + transcription | No | Yes |

**Premium is a fake paywall** — collects name + student email, stores in Supabase, updates profiles.plan = 'premium'. No real payments.

---

## DATABASE SCHEMA

```sql
-- profiles
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique not null,           -- @username format, set on first login
  display_name text,                       -- pulled from Google or entered manually
  plan text default 'free' check (plan in ('free', 'premium')),
  audio_consent boolean default false,     -- true after user consents to Groq audio processing
  created_at timestamptz default now()
);

-- sessions
create table sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  title text not null,
  source text not null check (source in ('recording', 'import', 'paste')),
  transcript text,
  status text default 'processing'
    check (status in ('recording', 'transcribing', 'generating', 'ready', 'error')),
  language text default 'en' check (language in ('en', 'ja', 'mixed')),
  duration_seconds integer,                -- populated on transcription complete (audio sessions)
  expires_at timestamptz,                  -- null for premium, created_at + 24hrs for free
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- notes
create table notes (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references sessions(id) on delete cascade,
  summary text,
  key_terms jsonb,        -- [{term: string, example: string}, ...]
  main_points jsonb,      -- [{heading: string, explanation: string}, ...]
  details jsonb,          -- [{fact: string}, ...]
  action_items jsonb,     -- [{item: string, type: 'TODO' | 'REC'}, ...]
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- flashcards
create table flashcards (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references sessions(id) on delete cascade,
  front text not null,
  back text not null,
  template text default 'basic' check (template in ('basic', 'cloze')),
  field_order text default 'front-back' check (field_order in ('front-back', 'back-front')),
  rating text check (rating in ('easy', 'hard', 'again')),
  ease_factor real default 2.5,
  created_at timestamptz default now()
);

-- premium_signups
create table premium_signups (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  full_name text not null,
  student_email text not null,
  created_at timestamptz default now()
);

-- notifications
create table notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  session_id uuid references sessions(id) on delete set null,
  message text not null,
  read boolean default false,
  created_at timestamptz default now()
);

-- export_counts
create table export_counts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  format text not null check (format in ('pdf', 'docx', 'txt')),
  count integer default 0,
  date date default current_date,
  unique (user_id, date, format)
);

-- error_logs
create table error_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete set null,
  session_id uuid references sessions(id) on delete set null,
  error_type text not null
    check (error_type in ('note_generation', 'flashcard_generation', 'transcription', 'file_parse')),
  error_message text,
  created_at timestamptz default now()
);

-- folders (premium only)
create table folders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  name text not null,
  created_at timestamptz default now()
);

-- tags (premium only)
create table tags (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  name text not null,
  created_at timestamptz default now(),
  unique (user_id, name)
);

-- session_tags
create table session_tags (
  session_id uuid references sessions(id) on delete cascade,
  tag_id uuid references tags(id) on delete cascade,
  primary key (session_id, tag_id)
);

-- session_folders
create table session_folders (
  session_id uuid references sessions(id) on delete cascade,
  folder_id uuid references folders(id) on delete cascade,
  primary key (session_id, folder_id)
);

-- Full-text search index
create index sessions_title_search_idx
  on sessions using gin(to_tsvector('english', title));
```

---

## API ROUTES (10 total)

### 1. POST /api/auth/signup-apu
- No auth required
- Validates email ends with @apu.ac.jp
- Validates password >= 8 chars
- Calls Supabase Admin auth.admin.createUser()
- Returns { success: true }

### 2. DELETE /api/auth/delete-account
- Auth required
- Calls Supabase Admin auth.admin.deleteUser(session.user.id)
- DB cascade handles all data deletion

### 3. POST /api/notes/generate
- Auth required
- Body: { transcript, language, session_id? }
- Calls Groq LLM with note generation prompt
- Parses response into: summary, key_terms, main_points, details, action_items
- Creates new session if no session_id, updates if session_id provided
- Sets expires_at = now() + 24h for free users, null for premium
- Upserts notes row
- Logs to error_logs on failure
- Returns: { session_id, notes: { summary, key_terms, main_points, details, action_items } }

### 4. POST /api/flashcards/generate
- Auth required
- Body: { session_id, notes_content }
- Calls Groq LLM with flashcard prompt
- Deletes existing flashcards for session, inserts new ones
- Default: template='basic', rating=null, ease_factor=2.5
- Logs to error_logs on failure
- Returns: { flashcards: [{ id, front, back, template }] }

### 5. POST /api/transcribe
- Auth required, premium only
- Body: multipart/form-data (audio chunk, session_id)
- Sends chunk to Groq Whisper (whisper-large-v3-turbo)
- Returns rate limit headers for client queue management
- Returns: { transcript, remaining_requests, reset_at }

### 6. POST /api/parse/vision
- Auth required
- Body: multipart/form-data (image file)
- Sends image to Groq vision (llama-4-scout multimodal)
- Returns: { text }

### 7. POST /api/admin/metrics
- Auth required, ADMIN_EMAIL only (returns 404 for others)
- Uses service role key
- Returns aggregate metrics: total_users, dau, wau, notes_generated, flashcards_generated, premium_signups, upload_breakdown, audio_sessions, export_counts

### 8. GET /api/admin/errors
- Auth required, ADMIN_EMAIL only
- Returns 50 most recent error_logs rows

### 9. GET /api/admin/users
- Auth required, ADMIN_EMAIL only
- Query param: username
- Returns matching profiles (id, username, display_name, plan, created_at)

### 10. POST /api/cron/cleanup
- Vercel cron secret header required
- Deletes sessions where expires_at < now()
- Deletes notifications where created_at < now() - 30 days
- Runs daily at midnight via Vercel cron

---

## FEATURES (19 total, summarized)

1. Google OAuth sign-in via Supabase
2. APU email + password sign-in — validated via /api/auth/signup-apu
3. Username selection on first login — saved to profiles.username
4. Account deletion — DELETE /api/auth/delete-account, cascade handles data
5. Paste transcript → notes — POST /api/notes/generate
6. Note viewer with inline editing (5 sections: Summary, Key Terms, Main Points, Details, Action Items) — direct Supabase update
7. Session expiry countdown for free users — client-side from expires_at
8. Regenerate notes — POST /api/notes/generate with session_id
9. Session list + full-text search — direct Supabase query
10. Flashcard generation — POST /api/flashcards/generate
11. Flashcard review (flip, rate Easy/Hard/Again) — direct Supabase update on flashcards.rating
12. Anki export editor (edit cards, field order, Basic/Cloze template) + .apkg download — client-side only, ease factors: Easy=2.7, Hard=1.8, Again=1.3, unrated=2.5
13. Note export PDF/DOCX/TXT — client-side (jsPDF, docx.js, Blob). Free users: 6/day tracked in export_counts
14. File upload → notes — pdfjs (text PDF), mammoth (DOCX), FileReader (TXT), POST /api/parse/vision (images + image PDFs)
15. Fake premium gate — modal collects full_name + student_email → premium_signups table, updates profiles.plan='premium'. Privacy consent stored in profiles.audio_consent
16. Audio recording + transcription queue (premium) — MediaRecorder API, IndexedDB queue, POST /api/transcribe per chunk, exponential backoff on 429
17. Folders + tags (premium) — direct Supabase CRUD on folders, tags, session_folders, session_tags
18. In-app notification bell — Supabase table + Realtime subscription for badge count
19. Admin dashboard — POST /api/admin/metrics + GET /api/admin/errors + GET /api/admin/users

---

## LLM PROMPTS

### Note Generation (POST /api/notes/generate)
System instructs Groq to output JSON:
```json
{
  "quote": "*motivating quote*",
  "summary": "2-3 paragraphs",
  "key_terms": [{ "term": "string", "example": "string" }],
  "main_points": [{ "heading": "string", "explanation": "string" }],
  "details": [{ "fact": "string" }],
  "action_items": [{ "item": "string", "type": "TODO" | "REC" }]
}
```
User message includes: Language + Transcript

### Flashcard Generation (POST /api/flashcards/generate)
System instructs Groq to output JSON:
```json
{
  "flashcards": [{ "front": "string", "back": "string", "template": "basic" | "cloze" }]
}
```
Cloze format uses {{c1::answer}} syntax. Min 5, max 30 cards.
User message includes: key_terms, main_points, details (JSON stringified)

### Vision OCR (POST /api/parse/vision)
System instructs model to extract and label text with structural tags: [HEADING], [SUBHEADING], [BODY], [BULLET], [NUMBERED], [TABLE]
Returns plain structured text string.

---

## WHAT IS NOT INCLUDED
- Real payments or Stripe
- Auth.js / NextAuth
- Social or sharing features
- Real-time collaboration (Supabase Realtime used for notification bell only)
- Email notifications of any kind
- Dark mode
- In-app SRS algorithm (Anki handles SRS after export)
- AI chat or Q&A on notes
- Custom note templates (structure is fixed)
- Browser extension
- Native mobile app
- AI quiz or test mode
- Note version history
- Public API
- Multi-account support
- Profile photos
- Anki card style editor
- Offline mode / service workers / PWA
- Multi-language UI (English only)
- PPTX parsing library (users export as PDF)
- Server-side export generation (all exports are client-side)
- User-to-user messaging

---

## ENVIRONMENT VARIABLES
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
GROQ_API_KEY=
ADMIN_EMAIL=
CRON_SECRET=
NEXT_PUBLIC_APP_URL=
```

---

## BUILD ORDER (7 sessions)
- Session 1: Next.js setup + Supabase config + both auth methods + username selection + account deletion + app shell + Vercel deploy
- Session 2: Note generation + note viewer + inline editing + session expiry + session list + search
- Session 3: Flashcard generation + review + Anki export editor + note export (PDF/DOCX/TXT)
- Session 4: File upload (all types)
- Session 5: Premium gate + privacy consent + audio recording + transcription queue (NO bell UI)
- Session 6: Folders + tags + notification bell + admin dashboard + landing page
- Session 7: Beta testing + prompt tuning + bug fixes

---

Now audit everything above. Report every issue you find.
