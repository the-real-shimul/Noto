# ノート — Noto

> 2 clicks from lecture to flashcards.

Noto is a web app for APU (Ritsumeikan Asia Pacific University) students to generate structured notes and Anki-ready flashcards from lecture transcripts or audio recordings. Built with AI-assisted development using Claude Code.

---

## What It Does

1. **Paste or upload** your lecture transcript (or record audio directly — premium)
2. **Generate structured notes** — summary, key terms, main points, details, action items
3. **Generate flashcards** automatically from your notes
4. **Export** notes as PDF, DOCX, or TXT — and flashcards as an Anki `.apkg` deck with ease factors baked in

No setup. No prompt engineering. Just your lecture content and your notes.

---

## Features

### Free
- Google OAuth and APU email sign-in
- Paste transcript or upload files (PDF, DOCX, TXT, JPG, PNG)
- AI-generated structured notes (5 sections)
- Inline note editing with save
- Flashcard generation and review (Easy / Hard / Again)
- Anki export editor (edit cards, swap front/back, Basic/Cloze templates)
- Note export — PDF, DOCX, TXT (6 exports/day)
- Session search
- 24-hour session history

### Premium
- Direct in-browser audio recording and transcription
- Unlimited session history
- Unlimited note exports
- Folders and tags for organizing sessions
- In-app notification bell

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router) + TypeScript |
| Styling | Tailwind CSS |
| Auth | Supabase Auth (Google OAuth + APU email) |
| Database | Supabase PostgreSQL |
| Storage | Supabase Storage |
| AI — Notes + Flashcards | Groq API (Llama 4 Scout) |
| AI — Transcription | Groq Whisper (whisper-large-v3-turbo) |
| AI — Image OCR | Groq Vision (Llama 4 Scout multimodal) |
| File Parsing | pdfjs-dist, mammoth.js, FileReader API |
| Note Export | jsPDF, docx.js, Blob |
| Anki Export | anki-apkg-export |
| Hosting | Vercel (free tier) |

**Guiding principle:** Keep it lean, keep it free. All exports are client-side. Server routes only exist where API keys must be protected.

---

## Project Structure

```
noto/
├── CLAUDE.md              # AI agent behavioral rules and project context
├── REVIEW_PROMPT.md       # Master audit prompt for pre-build review
├── README.md
└── docs/
    ├── stack.md           # Full tech stack decisions
    ├── schema.md          # PostgreSQL schema (all tables + RLS policies)
    ├── features.md        # All 19 features with user stories and acceptance criteria
    ├── not-included.md    # Explicit exclusions list (prevents scope creep)
    ├── build-order.md     # 7-session AI-assisted build plan
    ├── api-routes.md      # All 10 API routes with request/response specs
    ├── env.md             # Environment variable reference
    └── prompts.md         # Groq LLM prompt structures
```

---

## Database Schema

Key tables:

| Table | Purpose |
|-------|---------|
| `profiles` | User accounts, plan (free/premium), audio consent |
| `sessions` | One per recording or import, tracks expiry |
| `notes` | Structured note output (5 JSONB sections) |
| `flashcards` | Q&A pairs with template and rating |
| `premium_signups` | Validated demand collection |
| `notifications` | In-app bell entries |
| `export_counts` | Free tier export limit tracking |
| `error_logs` | AI failure reports for admin dashboard |
| `folders` / `tags` | Premium session organization |

RLS is enabled on all tables. Full schema in `docs/schema.md`.

---

## API Routes

| Route | Purpose |
|-------|---------|
| `POST /api/auth/signup-apu` | APU email domain validation |
| `DELETE /api/auth/delete-account` | Account + data deletion |
| `POST /api/notes/generate` | Note generation + regeneration |
| `POST /api/flashcards/generate` | Flashcard generation |
| `POST /api/transcribe` | Audio chunk → Groq Whisper |
| `POST /api/parse/vision` | Image → Groq vision → text |
| `POST /api/admin/metrics` | Admin dashboard metrics |
| `GET /api/admin/errors` | Admin error log |
| `GET /api/admin/users` | Admin username search |
| `POST /api/cron/cleanup` | Daily cleanup (Vercel cron) |

All Groq calls are server-side only. Full specs in `docs/api-routes.md`.

---

## Environment Variables

Create `.env.local` in the project root:

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
GROQ_API_KEY=
ADMIN_EMAIL=
CRON_SECRET=
NEXT_PUBLIC_APP_URL=
```

See `docs/env.md` for where to find each value and Vercel setup instructions.

---

## Getting Started

### Prerequisites
- Node.js 18+
- A Supabase project (free tier)
- A Groq API key (free at console.groq.com)
- A Vercel account (free tier)

### Local Setup

```bash
# Clone the repo
git clone https://github.com/the-real-shimul/noto.git
cd noto

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env.local
# Fill in your values in .env.local

# Run the development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

### Supabase Setup

1. Create a new Supabase project
2. Run all SQL from `docs/schema.md` in the Supabase SQL editor
3. Configure Google OAuth in Supabase dashboard → Authentication → Providers
4. Add redirect URLs: `http://localhost:3000/**` and your Vercel URL

### Vercel Deploy

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy
vercel
```

Add all environment variables in the Vercel dashboard. Add `vercel.json` for cron:

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

---

## Build Plan

Noto is built in 7 sessions using Claude Code (AI-assisted development):

| Session | Focus |
|---------|-------|
| 1 | Foundation — auth, shell, Vercel deploy |
| 2 | Core feature — note generation, viewer, search |
| 3 | Study tools — flashcards, Anki export, note export |
| 4 | File upload — all input types |
| 5 | Premium — gate, audio recording, transcription queue |
| 6 | Polish — folders, tags, notifications, admin dashboard |
| 7 | Beta — real user testing, prompt tuning, bug fixes |

Full session specs in `docs/build-order.md`.

---

## What's Not Included

By design, Noto does not include:

- Real payments (premium is a validated-demand gate)
- Email notifications
- Social or sharing features
- Dark mode
- Native mobile app
- AI chat or Q&A on notes
- Offline mode
- Browser extension
- In-app SRS algorithm (Anki handles this after export)

Full exclusions list in `docs/not-included.md`.

---

## Note Output Format

Each session generates notes in 5 sections:

- **Summary** — 2-3 paragraph overview of the lecture
- **Key Terms** — important terms with natural example sentences
- **Main Points** — headings with explanations for each major concept
- **Details** — dense factual content (stats, dates, theories, case studies)
- **Action Items** — tagged as `[TODO]` (professor-stated) or `[REC]` (AI-inferred)

Flashcards are generated from Key Terms, Main Points, and Details only. The AI auto-identifies Basic vs Cloze format per card.

---

## License

Private project. Not open source.

---

Built by [Shimul Al Imran](https://github.com/the-real-shimul) · APU Class of 2026
