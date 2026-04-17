# Noto — Tech Stack

## Guiding Principle
Keep it lean, keep it free. Prefer client-side execution over server compute. Stay within free tiers at all times.

## Framework & Hosting

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js | 15 (App Router) |
| Language | TypeScript | Latest |
| Styling | Tailwind CSS | Latest |
| Hosting | Vercel | Free tier |

- Use App Router exclusively. No Pages Router.
- API routes live in `app/api/` as Route Handlers.
- Deploy from GitHub. Auto-deploy on push to main.

## Authentication

| Layer | Technology |
|-------|-----------|
| Auth provider | Supabase Auth |
| Session management | @supabase/ssr |
| Login methods | Google OAuth, APU email + password (@apu.ac.jp only) |

- Auth.js (NextAuth) is NOT used. Do not introduce it.
- Google OAuth configured in Supabase dashboard.
- APU email login restricted to @apu.ac.jp domain. Enforced via a Next.js API route that validates the email before calling Supabase signUp. Do NOT use Supabase Auth hooks (requires paid plan).
- No email verification.
- Username (@username format) collected on first login and stored in profiles.username (not profiles.display_name).
- profiles.display_name stores the user's full name pulled from Google or entered manually.
- Warning shown on APU email signup: "Do not use your APU portal password."

## Database & Storage

| Layer | Technology |
|-------|-----------|
| Database | Supabase PostgreSQL |
| File storage | Supabase Storage |
| ORM | None — use Supabase JS client directly |

- Free tier: 500MB DB, 1GB storage, 50,000 MAU.
- Audio files deleted from Supabase Storage immediately after transcription.
- Row Level Security (RLS) enabled on all tables. Use auth.uid() in policies.
- No raw SQL queries in application code — use Supabase JS client.

## AI & Transcription

| Purpose | Provider | Model |
|---------|---------|-------|
| Note generation | Groq API | llama-4-scout (or latest available) |
| Flashcard generation | Groq API | llama-4-scout (or latest available) |
| Audio transcription | Groq Whisper API | whisper-large-v3-turbo |
| Image/scanned PDF OCR | Groq API | llama-4-scout vision (multimodal) |

- All Groq calls go through Next.js API routes (server-side) to protect API keys.
- Groq free tier limits: 14,400 RPD (LLM), 2,000 RPD / 7,200 audio sec/hr (Whisper).
- Groq rate limit headers monitored: x-ratelimit-remaining-requests, x-ratelimit-reset.

## File Parsing (Client-Side)

| Format | Library | Notes |
|--------|---------|-------|
| PDF (text-based) | pdfjs-dist | Client-side text extraction |
| DOCX | mammoth.js | Client-side text extraction |
| TXT | Native FileReader API | No library needed |
| PPTX | Prompt user to export as PDF first | No PPTX parsing library |
| Images (JPG/PNG) | Sent to Groq vision API | Via Next.js API route |
| PDF (image-based/scanned) | Detected by empty pdfjs output → sent to Groq vision | Fallback handled gracefully |

- If pdfjs returns empty string, show message: "This PDF appears to be image-based. Sending to AI for processing..." then route to Groq vision.
- PPTX: show message "Please export your PowerPoint as PDF before uploading."

## Export (Client-Side)

| Format | Library |
|--------|---------|
| Notes → PDF | jsPDF |
| Notes → DOCX | docx.js |
| Notes → TXT | Native Blob download |
| Flashcards → Anki .apkg | anki-apkg-export |

- All exports are client-side. No server compute involved.
- Free users: 6 note exports per day (resets at midnight). Tracked in Supabase.
- Premium users: unlimited exports.

## Audio Recording (Premium Only)

| Feature | Technology |
|---------|-----------|
| Mic capture | MediaRecorder API (browser-native) |
| Audio format | Native MediaRecorder output: WebM (Chrome/Edge), OGG (Firefox), MP4 (Safari) |
| Queue persistence | IndexedDB (client-side, survives tab close) |
| Transcription | Groq Whisper API via Next.js API route |

- Do NOT convert audio to WAV. Use native MediaRecorder output format.
- Audio chunked into 30-second segments before sending to Whisper.
- Exponential backoff on 429 rate limit errors: 2s, 4s, 8s, 16s, max 60s.
- Queue state persisted in IndexedDB so it survives page refresh.

## Notifications

| Type | Technology |
|------|-----------|
| Browser notifications | Browser Notification API |
| In-app notifications | Supabase table + dropdown bell UI |

- Notification bell in navbar with dropdown (not a full page).
- Supabase table: notifications(id, user_id, message, read, created_at).
- Auto-clear notifications older than 30 days.
- No email notifications. Ever.

## Admin Dashboard

- Gated via ADMIN_EMAIL environment variable. Never hardcoded.
- Server-side check: process.env.ADMIN_EMAIL === session.user.email.
- Built in Week 8 after core features are stable.

## Key Libraries Summary

```
@supabase/ssr
@supabase/supabase-js
mammoth
pdfjs-dist
jspdf
docx
anki-apkg-export
```

## What Is NOT In This Stack

- Auth.js / NextAuth — not used
- Stripe — not used
- Prisma / any ORM — not used
- Redis / any cache layer — not used
- Socket.io — not used (Supabase Realtime IS used for notification bell badge updates only)
- React Native / Expo — not used
- Any CSS framework other than Tailwind — not used
- Service workers / PWA — not used
