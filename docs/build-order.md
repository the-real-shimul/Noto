# Noto — Build Order

## Rules
- Build features strictly in session order. Do not jump ahead.
- Each session has a defined scope. Do not add features outside that scope.
- Every session ends with a working, deployable product.
- Commit to GitHub after each session.
- Test each feature before marking it complete.
- Claude Code is the primary builder. Shimul directs and troubleshoots.

## Context
This app is built using AI-assisted development (Claude Code). Each "session" represents one focused Claude Code working session, not a calendar week. Sessions can be completed in 2-4 hours each. The build order is designed to deliver a demonstrable product increment after every session, supporting the learn-build-measure framework.

---

## Session 1 — Foundation
**Goal:** Authenticated shell deployed to Vercel. Students can sign in.

Tasks:
- Initialize Next.js 15 project with TypeScript and Tailwind CSS (App Router only)
- Configure Supabase project: create all tables from schema.md, enable RLS, set up policies
- Configure Supabase Auth: Google OAuth provider, APU email + password provider
- Implement Google OAuth sign-in flow using @supabase/ssr
- Implement APU email + password sign-up and sign-in (@apu.ac.jp validation server-side)
- Warning on APU signup form: "Do not use your APU portal password"
- Username selection page (first login only, @username format, availability check)
- Account deletion in settings (confirmation modal, "DELETE" typed, cascade delete)
- Basic app shell: navbar with logo (ノート), notification bell placeholder, sidebar nav
- Empty placeholder pages: Dashboard, New Session, Settings
- Deploy to Vercel, configure environment variables
- Set up GitHub repo, auto-deploy from main branch

Do NOT build in this session:
- Note generation
- Any AI features
- Flashcards
- File upload

**Deliverable:** Student can sign in with Google or APU email, set a username, and see the app shell live on Vercel.

**Verify:**
- Google OAuth flow works end to end
- APU email rejects non @apu.ac.jp addresses
- Username is saved to profiles.username
- Returning users skip username selection
- Account deletion removes all data and signs user out
- App is live on Vercel URL

---

## Session 2 — Core Feature: Notes
**Goal:** Student can paste a transcript and receive structured notes.

Tasks:
- New Session page: text area for pasting transcript (min 100 chars to enable submit)
- Language selector: English / Japanese / Mixed
- Groq LLM API integration via Next.js API route (server-side, API key never exposed)
- Note generation prompt (see prompts.md)
- Loading state: "Generating your notes..."
- Note viewer page: five sections (Summary, Key Terms, Main Points, Details, Action Items)
- Inline editing per section (contenteditable), Save button, Supabase update on save
- Regenerate Notes button with confirmation modal
- Session saved to Supabase with source = 'paste'
- Free users: expires_at = created_at + 24 hours
- Premium users: expires_at = null
- Session expiry countdown shown on note viewer for free users ("Expires in Xh Ym", red under 1 hour)
- Session list on dashboard: title, source badge, status badge, created date, expiry countdown
- Search bar on dashboard (PostgreSQL full-text search on title, debounced 300ms)
- Delete Session button with confirmation
- Error logging to error_logs table on Groq failure
- User-facing error: "Note generation failed. Please try again."

Do NOT build in this session:
- File upload
- Flashcards
- Anki export
- Note export

**Deliverable:** Student pastes a lecture transcript and receives structured notes they can edit and search.

**Verify:**
- Notes generate correctly from a sample transcript
- All four sections display and are editable
- Save persists changes to Supabase
- Free user sees countdown timer
- Session appears in dashboard list
- Search filters sessions correctly
- Groq failure logs to error_logs and shows user-facing error

---

## Session 3 — Flashcards + All Exports
**Goal:** Full study and export pipeline working.

Tasks:
- Flashcard generation from notes via Groq LLM (see prompts.md)
- Flashcard review page: one card at a time, flip on click or Space
- Rating buttons: Easy / Hard / Again (saved to Supabase immediately)
- Progress indicator: "Card 4 of 18"
- Post-review summary: X Easy, Y Hard, Z Again
- Review Again button
- Keyboard shortcuts: Space = flip, 1 = Easy, 2 = Hard, 3 = Again
- Anki export editor page:
  - Editable front/back per card
  - Field order toggle per card (front-back / back-front)
  - Template selector per card (Basic / Cloze)
  - Add Card button
  - Delete Card button
  - Export button → .apkg download via anki-apkg-export (client-side)
  - Ease factors baked in: Easy = 2.7, Hard = 1.8, Again = 1.3, unrated = 2.5
- Note export buttons on note viewer: PDF (jsPDF), DOCX (docx.js), TXT (Blob)
- Exported filename: [session-title]-notes.[ext]
- Free user export limit: check export_counts before download, block at 6/day
- Message when limit reached: "Daily export limit reached. Upgrade to Premium for unlimited exports."
- Export count incremented in Supabase after successful download
- Premium users: no export limit check

Do NOT build in this session:
- File upload
- Audio recording
- Premium gate
- Folders or tags

**Deliverable:** Student generates flashcards, reviews them, exports an Anki deck, and downloads their notes as PDF/DOCX/TXT.

**Verify:**
- Flashcards generate from a sample note set
- Flip and rating work correctly
- Ratings saved to Supabase
- Anki .apkg downloads and imports into Anki correctly
- Ease factors reflect ratings
- PDF, DOCX, TXT downloads work
- Free user blocked after 6 exports
- Premium user has no limit

---

## Session 4 — File Upload
**Goal:** All file input types supported alongside paste.

Tasks:
- New Session page updated: drag-and-drop file upload zone alongside paste text area
- Accepted types: PDF, DOCX, TXT, JPG, PNG
- File size limit: 10MB (client-side validation)
- PPTX: show message "Please export your PowerPoint as PDF before uploading."
- TXT: parse via FileReader API
- DOCX: parse client-side via mammoth.js
- PDF (text-based): parse client-side via pdfjs-dist
- PDF (image-based): detected when pdfjs returns empty string → show "This PDF appears to be image-based. Sending to AI for processing..." → route to Groq vision API via Next.js API route
- JPG/PNG: sent directly to Groq vision API via Next.js API route
- Extracted text shown in editable preview before generating notes
- Session saved with source = 'import'
- All note generation acceptance criteria from Session 2 apply

Do NOT build in this session:
- Audio recording
- Premium gate
- Folders or tags

**Deliverable:** Student uploads any supported file type and receives structured notes.

**Verify:**
- Text-based PDF extracts correctly
- Image-based PDF routes to Groq vision
- DOCX extracts correctly via mammoth
- JPG/PNG routes to Groq vision
- PPTX shows correct message
- 10MB limit enforced
- Extracted text preview is editable before generating

---

## Session 5 — Premium Gate + Audio Recording
**Goal:** Premium flow complete. Audio recording works end to end.

Tasks:
- "Get Premium" button in navbar and on restricted feature pages
- Premium modal: Full Name field, Student Email field, Submit button
- On submit: save to premium_signups, update profiles.plan = 'premium', unlock premium immediately
- "Get Premium" replaced with "Premium" badge after signup
- Privacy consent screen (one-time, before first recording):
  - Text: "Audio is processed by Groq (US). It is not stored after transcription."
  - Checkbox: "I understand and consent"
  - Consent stored in Supabase (boolean column on profiles)
- Recording page (premium only):
  - Free users see upgrade prompt
  - "Start Recording" activates MediaRecorder API
  - Recording timer: "Recording: 12:34"
  - "Stop Recording" ends capture
  - Audio saved to IndexedDB immediately
  - Audio chunked into 30-second segments (native format: WebM/OGG/MP4 — no WAV)
  - Queue processor sends chunks sequentially to Groq Whisper via Next.js API route
  - Progress: "Transcribing... 4/18 chunks"
  - Rate limit headers monitored: x-ratelimit-remaining-requests, x-ratelimit-reset
  - Exponential backoff on 429: 2s, 4s, 8s, 16s, max 60s
  - User shown: "Rate limited. Resuming in ~Xs"
  - Network loss: "Waiting for connection..." with auto-retry
  - Queue state persisted in IndexedDB (survives tab close and page reload)
  - On completion: transcript stitched, note generation triggered automatically
  - Browser notification: "Your notes for [session title] are ready"
  - In-app notification created in notifications table
  - Audio deleted from IndexedDB after transcription (audio is never uploaded to Supabase Storage)
  - Session saved with source = 'recording'

Do NOT build in this session:
- Folders or tags
- Admin dashboard
- In-app notification bell UI (write to notifications table only — bell UI is built in Session 6)

**Deliverable:** Premium student signs up, records a lecture, receives a browser notification when notes are ready.

**Verify:**
- Premium modal saves to Supabase and unlocks features immediately
- Privacy consent shown only once
- Recording starts and stops correctly
- Audio chunks saved to IndexedDB
- Transcription queue processes chunks sequentially
- Rate limit backoff works on 429
- Queue resumes after page reload
- Browser notification fires on completion
- In-app notification created
- Audio deleted after transcription

---

## Session 6 — Polish + Admin
**Goal:** Premium features polished. Admin dashboard complete. App ready for beta.

Tasks:
- Folders (premium only):
  - "New Folder" button on dashboard
  - Sessions assignable to folders via dropdown
  - Filter session list by folder
  - Rename and delete folders (deleting folder does not delete sessions)
- Tags (premium only):
  - "New Tag" button on dashboard
  - Sessions taggable via tag picker
  - Filter session list by tag
  - Rename and delete tags
- In-app notification bell:
  - Bell icon in navbar with unread count badge
  - Dropdown showing last 20 notifications
  - Relative timestamps ("2 minutes ago")
  - Click notification → mark as read + navigate to session
  - "Mark all as read" button
  - Supabase Realtime subscription for live badge count updates
  - Empty state: "No notifications yet."
- Admin dashboard (/admin):
  - Gated by process.env.ADMIN_EMAIL server-side (returns 404 for others)
  - Metrics: total users, DAU, WAU, notes generated, flashcards generated
  - Premium signups list (name, email, created_at)
  - Upload type breakdown
  - Audio session count + avg duration
  - Export count by format
  - Error log list (most recent 50)
  - Username search
  - Read-only — no edit or delete capabilities
- Landing page:
  - Headline: "2 clicks from lecture to flashcards"
  - Free vs Premium comparison
  - Sign in button
  - Clean, minimal design

**Deliverable:** App is fully polished, admin dashboard live, landing page complete, ready for APU student beta.

**Verify:**
- Folders and tags work for premium users only
- Free users cannot access folders or tags
- Notification bell updates in real-time
- Admin dashboard accessible only via ADMIN_EMAIL
- All metrics display correctly
- Error logs visible
- Landing page renders correctly on mobile and desktop

---

## Session 7 — Beta + Iteration
**Goal:** Real APU students use the app. Bugs fixed based on feedback.

Tasks:
- Share app with 5-10 APU students
- Monitor error_logs in admin dashboard
- Fix bugs surfaced by beta users
- Tune note generation prompts based on real lecture content (see prompts.md)
- Tune flashcard generation prompts
- Collect premium signup count for Venture Entrepreneurship presentation
- Prepare validated demand slide: X students clicked "Get Premium" knowing it costs 500 yen/mo

**Deliverable:** Stable, tested app with real user data. Ready for class presentation.
