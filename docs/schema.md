# Noto — Database Schema

## Rules
- RLS (Row Level Security) is enabled on ALL tables.
- Never disable RLS on any table.
- All user-facing queries must filter by auth.uid().
- Do not add columns or tables not defined here without explicit instruction.
- Use Supabase JS client for all queries. No raw SQL in application code.

---

## profiles
Extends Supabase auth.users. Created automatically on first login via trigger.

```sql
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  username text unique not null,           -- @username format, set on first login
  display_name text,                       -- pulled from Google or entered manually
  plan text default 'free'
    check (plan in ('free', 'premium')),
  audio_consent boolean default false,     -- true after user consents to Groq audio processing
  created_at timestamptz default now()
);

-- RLS
alter table profiles enable row level security;
create policy "Users can read own profile"
  on profiles for select using (auth.uid() = id);
create policy "Users can update own profile"
  on profiles for update using (auth.uid() = id);
```

---

## sessions
One per recording or file import or paste. Tracks source type and expiry.

```sql
create table sessions (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  title text not null,
  source text not null
    check (source in ('recording', 'import', 'paste')),
  transcript text,
  status text default 'processing'
    check (status in ('recording', 'transcribing', 'generating', 'ready', 'error')),
  language text default 'en'
    check (language in ('en', 'ja', 'mixed')),
  duration_seconds integer,                -- populated on transcription complete (audio sessions)
  expires_at timestamptz,                  -- null for premium, created_at + 24hrs for free
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- RLS
alter table sessions enable row level security;
create policy "Users can CRUD own sessions"
  on sessions for all using (auth.uid() = user_id);
```

---

## notes
One set of structured notes per session.

```sql
create table notes (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references sessions(id) on delete cascade,
  summary text,
  key_terms jsonb,        -- [{term: string, example: string}, ...]
  main_points jsonb,      -- [{heading: string, explanation: string}, ...]
  details jsonb,          -- [{fact: string}, ...] dense factual content for flashcards
  action_items jsonb,     -- [{item: string, type: 'TODO' | 'REC'}, ...]
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- RLS
alter table notes enable row level security;
create policy "Users can CRUD own notes"
  on notes for all using (
    auth.uid() = (select user_id from sessions where id = notes.session_id)
  );
```

---

## flashcards
Q&A pairs generated per session. Supports basic and cloze templates.

```sql
create table flashcards (
  id uuid primary key default gen_random_uuid(),
  session_id uuid references sessions(id) on delete cascade,
  front text not null,                     -- question or cloze text
  back text not null,                      -- answer
  template text default 'basic'
    check (template in ('basic', 'cloze')),
  field_order text default 'front-back'
    check (field_order in ('front-back', 'back-front')),
  rating text check (rating in ('easy', 'hard', 'again')),
  ease_factor real default 2.5,            -- baked into Anki export
  created_at timestamptz default now()
);

-- RLS
alter table flashcards enable row level security;
create policy "Users can CRUD own flashcards"
  on flashcards for all using (
    auth.uid() = (select user_id from sessions where id = flashcards.session_id)
  );
```

---

## premium_signups
Collected via fake premium gate. Used as validated demand evidence.

```sql
create table premium_signups (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  full_name text not null,
  student_email text not null,
  created_at timestamptz default now()
);

-- RLS
alter table premium_signups enable row level security;
create policy "Users can insert own signup"
  on premium_signups for insert with check (auth.uid() = user_id);
create policy "Users can read own signup"
  on premium_signups for select using (auth.uid() = user_id);
```

---

## notifications
In-app notification bell entries per user.

```sql
create table notifications (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  session_id uuid references sessions(id) on delete set null,  -- for navigation on click
  message text not null,
  read boolean default false,
  created_at timestamptz default now()
);

-- RLS
alter table notifications enable row level security;
create policy "Users can read and update own notifications"
  on notifications for all using (auth.uid() = user_id);
```

Note: A scheduled Supabase function or cron should delete notifications older than 30 days.

---

## export_counts
Tracks daily note exports for free users. Enforces 6 exports/day limit.

```sql
create table export_counts (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  format text not null check (format in ('pdf', 'docx', 'txt')),
  count integer default 0,
  date date default current_date,
  unique (user_id, date, format)
);

-- RLS
alter table export_counts enable row level security;
create policy "Users can read and update own export counts"
  on export_counts for all using (auth.uid() = user_id);
```

Logic: On each export, upsert a row for (user_id, today). If count >= 6 and plan = 'free', block export and show upgrade prompt.

---

## error_logs
Failed note/flashcard generation and transcription errors. Visible in admin dashboard.

```sql
create table error_logs (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete set null,
  session_id uuid references sessions(id) on delete set null,
  error_type text not null
    check (error_type in ('note_generation', 'flashcard_generation', 'transcription', 'file_parse')),
  error_message text,
  created_at timestamptz default now()
);

-- RLS: admin only via service role key in API route
alter table error_logs enable row level security;
create policy "No direct user access"
  on error_logs for all using (false);
```

---

## folders
Premium only. User-created folders for organizing sessions.

```sql
create table folders (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  name text not null,
  created_at timestamptz default now()
);

-- RLS
alter table folders enable row level security;
create policy "Users can CRUD own folders"
  on folders for all using (auth.uid() = user_id);
```

---

## tags
Premium only. User-created tags attached to sessions.

```sql
create table tags (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id) on delete cascade,
  name text not null,
  created_at timestamptz default now(),
  unique (user_id, name)
);

create table session_tags (
  session_id uuid references sessions(id) on delete cascade,
  tag_id uuid references tags(id) on delete cascade,
  primary key (session_id, tag_id)
);

-- RLS
alter table tags enable row level security;
create policy "Users can CRUD own tags"
  on tags for all using (auth.uid() = user_id);

alter table session_tags enable row level security;
create policy "Users can CRUD own session tags"
  on session_tags for all using (
    auth.uid() = (select user_id from sessions where id = session_tags.session_id)
  );
```

---

## session_folders
Junction table linking sessions to folders (premium only).

```sql
create table session_folders (
  session_id uuid references sessions(id) on delete cascade,
  folder_id uuid references folders(id) on delete cascade,
  primary key (session_id, folder_id)
);

-- RLS
alter table session_folders enable row level security;
create policy "Users can CRUD own session folders"
  on session_folders for all using (
    auth.uid() = (select user_id from sessions where id = session_folders.session_id)
  );
```

---

## Full-Text Search
Sessions are searchable by title using PostgreSQL full-text search.

```sql
-- Add index for search performance
create index sessions_title_search_idx
  on sessions using gin(to_tsvector('english', title));
```

Query pattern:
```sql
select * from sessions
where user_id = auth.uid()
and to_tsvector('english', title) @@ plainto_tsquery('english', 'search term')
order by created_at desc;
```
