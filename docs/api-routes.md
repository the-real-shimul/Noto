# Noto — API Routes

## Rules
- All routes live in `app/api/` as Next.js Route Handlers.
- All Groq API calls are server-side only. Never call Groq from the client.
- All routes (except cron) validate the Supabase session before executing.
- Admin routes use the Supabase service role key (bypasses RLS). Never expose the service role key to the client.
- All routes return consistent error shape: `{ error: string }`
- Client-side operations (file parsing, exports, Anki generation) have NO API routes.
- Do not create routes not listed here without explicit instruction.

## Auth Validation Pattern
Used in every non-admin, non-cron route:
```ts
const supabase = createRouteHandlerClient({ cookies })
const { data: { session } } = await supabase.auth.getSession()
if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
```

## Admin Auth Pattern
Used in all /api/admin/* routes:
```ts
const supabase = createRouteHandlerClient({ cookies })
const { data: { session } } = await supabase.auth.getSession()
if (!session || session.user.email !== process.env.ADMIN_EMAIL) {
  return NextResponse.json({ error: 'Not found' }, { status: 404 })
}
const adminClient = createClient(process.env.NEXT_PUBLIC_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY)
```

---

## 1. POST /api/auth/signup-apu
**Purpose:** Validates APU email domain before creating a Supabase user. Supabase Auth hooks require paid plan — this route replaces that.

**Auth required:** No (pre-auth)

**Request body:**
```json
{
  "email": "student@apu.ac.jp",
  "password": "minimum8chars"
}
```

**Logic:**
1. Validate email ends with @apu.ac.jp — return 400 if not
2. Validate password is minimum 8 characters — return 400 if not
3. Call Supabase Admin `auth.admin.createUser({ email, password, email_confirm: true })`
4. Return success

**Response (200):**
```json
{ "success": true }
```

**Errors:**
- 400: `{ "error": "Email must end with @apu.ac.jp" }`
- 400: `{ "error": "Password must be at least 8 characters" }`
- 409: `{ "error": "An account with this email already exists" }`
- 500: `{ "error": "Account creation failed. Please try again." }`

---

## 2. DELETE /api/auth/delete-account
**Purpose:** Deletes the Supabase Auth user record. DB cascade handles all user data. Requires service role key — cannot be done client-side.

**Auth required:** Yes

**Request body:** None

**Logic:**
1. Validate session
2. Call Supabase Admin `auth.admin.deleteUser(session.user.id)`
3. DB cascade deletes: profiles → sessions → notes → flashcards → notifications → folders → tags → premium_signups → export_counts

**Response (200):**
```json
{ "success": true }
```

**Errors:**
- 401: `{ "error": "Unauthorized" }`
- 500: `{ "error": "Account deletion failed. Please try again." }`

---

## 3. POST /api/notes/generate
**Purpose:** Generates structured notes from a transcript using Groq LLM. Handles both new sessions and regeneration (when session_id is provided).

**Auth required:** Yes

**Request body:**
```json
{
  "transcript": "lecture transcript text...",
  "language": "en" | "ja" | "mixed",
  "session_id": "uuid (optional — provided for regeneration)"
}
```

**Logic:**
1. Validate session
2. Call Groq LLM with note generation prompt (see prompts.md)
3. Parse structured response into: summary, key_terms, main_points, details, action_items
4. If no session_id: create new sessions row, set expires_at based on plan (free = now + 24h, premium = null)
5. If session_id provided: update existing sessions row (regeneration)
6. Upsert notes row for the session
7. On Groq failure: log to error_logs table, return 500

**Response (200):**
```json
{
  "session_id": "uuid",
  "notes": {
    "summary": "...",
    "key_terms": [{ "term": "...", "example": "..." }],
    "main_points": [{ "heading": "...", "explanation": "..." }],
    "details": [{ "fact": "..." }],
    "action_items": [{ "item": "...", "type": "TODO" }]
  }
}
```

**Errors:**
- 400: `{ "error": "Transcript is required" }`
- 401: `{ "error": "Unauthorized" }`
- 500: `{ "error": "Note generation failed. Please try again." }`

---

## 4. POST /api/flashcards/generate
**Purpose:** Generates Q&A flashcard pairs from notes content using Groq LLM.

**Auth required:** Yes

**Request body:**
```json
{
  "session_id": "uuid",
  "notes_content": "combined notes text for LLM context"
}
```

**Logic:**
1. Validate session
2. Verify session belongs to authenticated user
3. Call Groq LLM with flashcard generation prompt (see prompts.md)
4. Parse response into array of { front, back } pairs (minimum 5)
5. Delete existing flashcards for this session_id (regeneration case)
6. Insert new flashcards with template = 'basic', rating = null, ease_factor = 2.5
7. On Groq failure: log to error_logs table, return 500

**Response (200):**
```json
{
  "flashcards": [
    { "id": "uuid", "front": "...", "back": "...", "template": "basic" }
  ]
}
```

**Errors:**
- 400: `{ "error": "session_id and notes_content are required" }`
- 401: `{ "error": "Unauthorized" }`
- 403: `{ "error": "Session not found" }`
- 500: `{ "error": "Flashcard generation failed. Please try again." }`

---

## 5. POST /api/transcribe
**Purpose:** Sends one audio chunk to Groq Whisper and returns the transcript segment. Called sequentially per chunk by the client-side queue processor.

**Auth required:** Yes (must be premium user)

**Request body:** `multipart/form-data`
- `audio`: audio file chunk (WebM, OGG, or MP4)
- `session_id`: uuid

**Logic:**
1. Validate session
2. Check profiles.plan = 'premium' — return 403 if free user
3. Forward audio chunk to Groq Whisper API (`whisper-large-v3-turbo`)
4. Return transcript segment text
5. Read and return Groq rate limit headers for client-side queue management
6. On Groq failure: log to error_logs table, return 500

**Response (200):**
```json
{
  "transcript": "segment text...",
  "remaining_requests": 1843,
  "reset_at": "2026-04-17T12:00:00Z"
}
```

**Errors:**
- 401: `{ "error": "Unauthorized" }`
- 403: `{ "error": "Audio recording is a Premium feature" }`
- 429: `{ "error": "Rate limited", "reset_at": "..." }` (forwarded from Groq)
- 500: `{ "error": "Transcription failed. Please try again." }`

---

## 6. POST /api/parse/vision
**Purpose:** Sends an image file (JPG, PNG, or image-based PDF page) to Groq vision model and returns extracted text.

**Auth required:** Yes

**Request body:** `multipart/form-data`
- `file`: image file (JPG or PNG)

**Logic:**
1. Validate session
2. Forward image to Groq vision API (`llama-4-scout` multimodal)
3. Prompt: extract all readable text from this image faithfully, preserving structure
4. Return extracted text string
5. On Groq failure: log to error_logs, return 500

**Response (200):**
```json
{
  "text": "extracted text content..."
}
```

**Errors:**
- 400: `{ "error": "Image file is required" }`
- 401: `{ "error": "Unauthorized" }`
- 500: `{ "error": "Image parsing failed. Please try again." }`

---

## 7. POST /api/admin/metrics
**Purpose:** Returns all admin dashboard aggregate metrics. Uses service role key to bypass RLS.

**Auth required:** Yes (ADMIN_EMAIL only — returns 404 for all others)

**Request body:** None

**Logic:**
1. Validate session and ADMIN_EMAIL match
2. Use Supabase service role client to query:
   - Total users: `count(*) from profiles`
   - DAU: `count(*) from sessions where created_at >= today`
   - WAU: `count(*) from sessions where created_at >= 7 days ago`
   - Notes generated: `count(*) from notes`
   - Flashcards generated: `count(*) from flashcards`
   - Premium signups: `select full_name, student_email, created_at from premium_signups order by created_at desc`
   - Upload type breakdown: `count(*) from sessions group by source`
   - Audio sessions: `count(*), avg(duration_seconds) from sessions where source = 'recording'`
   - Export counts by format: `sum(count) from export_counts group by format`

**Response (200):**
```json
{
  "total_users": 42,
  "dau": 12,
  "wau": 28,
  "notes_generated": 156,
  "flashcards_generated": 892,
  "premium_signups": [...],
  "upload_breakdown": { "paste": 80, "import": 60, "recording": 16 },
  "audio_sessions": { "count": 16, "avg_duration_seconds": 3420 },
  "export_counts": { "pdf": 44, "docx": 12, "txt": 8 }
}
```

**Errors:**
- 404: (returned for all non-admin users, no distinguishing error)
- 500: `{ "error": "Failed to load metrics" }`

---

## 8. GET /api/admin/errors
**Purpose:** Returns the 50 most recent error log entries for the admin dashboard.

**Auth required:** Yes (ADMIN_EMAIL only)

**Query params:** None

**Logic:**
1. Validate session and ADMIN_EMAIL match
2. Use service role client: `select * from error_logs order by created_at desc limit 50`

**Response (200):**
```json
{
  "errors": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "session_id": "uuid",
      "error_type": "note_generation",
      "error_message": "...",
      "created_at": "..."
    }
  ]
}
```

**Errors:**
- 404: (returned for all non-admin users)
- 500: `{ "error": "Failed to load error logs" }`

---

## 9. GET /api/admin/users
**Purpose:** Searches users by @username for the admin dashboard.

**Auth required:** Yes (ADMIN_EMAIL only)

**Query params:**
- `username`: string (the @username to search, with or without the @ prefix)

**Logic:**
1. Validate session and ADMIN_EMAIL match
2. Strip @ prefix if present
3. Use service role client: `select id, username, display_name, plan, created_at from profiles where username ilike '%search%' limit 20`

**Response (200):**
```json
{
  "users": [
    {
      "id": "uuid",
      "username": "@shimul",
      "display_name": "Shimul Al Imran",
      "plan": "premium",
      "created_at": "..."
    }
  ]
}
```

**Errors:**
- 400: `{ "error": "username query param is required" }`
- 404: (returned for all non-admin users)
- 500: `{ "error": "User search failed" }`

---

## 10. POST /api/cron/cleanup
**Purpose:** Deletes expired free user sessions and notifications older than 30 days. Called by Vercel cron at midnight daily.

**Auth required:** Vercel cron secret header

**Request body:** None

**Vercel cron config (vercel.json):**
```json
{
  "crons": [
    {
      "path": "/api/cron/cleanup",
      "schedule": "0 0 * * *"
    }
  ]
}
```

**Security:** Vercel automatically sets `Authorization: Bearer <CRON_SECRET>` on cron requests. Validate with:
```ts
if (request.headers.get('Authorization') !== `Bearer ${process.env.CRON_SECRET}`) {
  return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
}
```

**Logic:**
1. Validate cron secret header
2. Use service role client:
   - Delete expired sessions: `delete from sessions where expires_at < now()`
   - Delete old notifications: `delete from notifications where created_at < now() - interval '30 days'`
3. Return counts of deleted rows

**Response (200):**
```json
{
  "deleted_sessions": 14,
  "deleted_notifications": 3
}
```

**Errors:**
- 401: `{ "error": "Unauthorized" }`
- 500: `{ "error": "Cleanup failed" }`

---

## Environment Variables Required
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
GROQ_API_KEY=
ADMIN_EMAIL=
CRON_SECRET=
```
