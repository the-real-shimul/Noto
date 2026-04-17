# Noto — Features & Acceptance Criteria

## Rules
- Every feature is described as a user story with explicit acceptance criteria.
- Acceptance criteria are testable and specific — no vague language.
- Do not build features not listed here without explicit instruction.
- "Premium only" features are gated by profiles.plan = 'premium' checked server-side.

---

## 1. Google OAuth Sign-In

**As a student, I want to sign in with my Google account so that I can access Noto without creating a new password.**

Acceptance criteria:
- "Sign in with Google" button visible on landing/login page
- Clicking it triggers Supabase Google OAuth flow
- On success, user is redirected to username selection if first login, or dashboard if returning
- On failure, error message shown: "Sign-in failed. Please try again."
- Session persisted via @supabase/ssr across page refreshes

---

## 2. APU Email + Password Sign-In

**As an APU student, I want to sign in with my APU email and a password I create so that I have an alternative to Google login.**

Acceptance criteria:
- Login page has two options: "Sign in with Google" and "Sign in with APU email"
- APU email sign-up form has fields: APU email, password, confirm password
- Validation via Next.js API route rejects any email not ending in @apu.ac.jp before calling Supabase signUp (do not use Supabase Auth hooks — requires paid plan)
- Warning displayed prominently on signup form: "Do not use your APU portal password"
- Password minimum 8 characters enforced
- No email verification step — account active immediately on signup
- On login failure, error shown: "Invalid email or password."
- Existing APU email users cannot re-register with the same email

---

## 3. Username Selection on First Login

**As a new user, I want to choose a username on first login so that I have an identity in the app.**

Acceptance criteria:
- After first successful login (any method), user is redirected to username selection page before dashboard
- Username field requires @ prefix format (@username)
- Username is 3-20 characters, alphanumeric and underscores only (after the @)
- Real-time availability check as user types
- Taken usernames show: "This username is already taken"
- On confirm, username saved to profiles.username
- User never sees username selection page again after completing it
- Returning users skip directly to dashboard

---

## 4. Account Deletion

**As a user, I want to permanently delete my account so that my data is removed from Noto.**

Acceptance criteria:
- "Delete Account" button visible in settings page
- Clicking shows a confirmation modal: "This will permanently delete all your notes, flashcards, and sessions. This cannot be undone."
- User must type "DELETE" to confirm
- On confirm, profiles row deleted (cascades to all sessions, notes, flashcards, notifications, folders, tags)
- User signed out immediately after deletion
- Redirected to landing page with message: "Your account has been deleted."
- Supabase Auth user record also deleted

---

## 5. Paste Transcript → Generate Notes

**As a student, I want to paste my lecture transcript and generate structured notes so that I can study without formatting manually.**

Acceptance criteria:
- New session page has a text area for pasting transcript
- Minimum 100 characters required before "Generate Notes" button is enabled
- Language selector available: English / Japanese / Mixed
- Clicking "Generate Notes" triggers Groq LLM API call via Next.js API route
- Loading state shown: "Generating your notes..."
- On success, user redirected to note viewer page
- Session saved to Supabase with source = 'paste'
- For free users, expires_at set to created_at + 24 hours
- For premium users, expires_at is null
- On Groq API failure, error logged to error_logs table and user shown: "Note generation failed. Please try again."

---

## 6. Note Viewer + Inline Editing

**As a student, I want to view and edit my generated notes so that I can correct errors and add my own context.**

Acceptance criteria:
- Notes displayed in four sections: Summary, Key Terms, Main Points, Action Items
- Each section is individually editable inline (contenteditable or textarea)
- "Save" button appears when any section is edited
- Saving triggers a Supabase update on the notes table
- Success feedback shown: "Notes saved"
- Ctrl+Z (undo) works natively within each editable section
- Notes are read-only until user clicks into a section
- Mobile-responsive layout

---

## 7. Session Expiry Countdown (Free Users)

**As a free user, I want to see how much time is left before my session expires so that I know when to export my notes.**

Acceptance criteria:
- Countdown timer shown on note viewer and session list for free users: "Expires in Xh Ym"
- Timer updates every minute
- When under 1 hour remaining, timer turns red
- When session expires, it is deleted from Supabase automatically
- Expired sessions do not appear in session list
- Premium users see no countdown — their sessions do not expire
- A subtle "Upgrade to Premium" prompt shown next to countdown

---

## 8. Regenerate Notes

**As a student, I want to regenerate my notes so that I can get a fresh AI output if the first result was poor.**

Acceptance criteria:
- "Regenerate Notes" button visible on note viewer page
- Clicking shows confirmation: "This will replace your current notes. Any edits will be lost."
- On confirm, new Groq LLM call made with the same transcript
- Loading state shown during generation
- Existing notes record in Supabase overwritten with new output
- Flash cards generated from previous notes are NOT deleted automatically
- On failure, error logged and user shown: "Regeneration failed. Please try again."

---

## 9. Session List + Search

**As a student, I want to see all my sessions and search them by title so that I can find past notes quickly.**

Acceptance criteria:
- Dashboard shows list of sessions sorted by created_at descending
- Each session card shows: title, source (paste/import/recording), status badge, created date, expiry countdown (free users only)
- Search bar at top filters sessions using PostgreSQL full-text search on title
- Search results update as user types (debounced 300ms)
- Empty state shown when no sessions exist: "No sessions yet. Create your first one."
- Empty search state shown: "No sessions match your search."
- Free users see expired sessions removed automatically
- "Delete Session" button on each card with confirmation modal

---

## 10. Flashcard Generation

**As a student, I want to generate flashcards from my notes so that I can study key concepts efficiently.**

Acceptance criteria:
- "Generate Flashcards" button visible on note viewer page
- Clicking triggers Groq LLM API call via Next.js API route using the notes content
- Loading state shown: "Generating flashcards..."
- Minimum 5 flashcards generated per session
- Each flashcard has front (question) and back (answer) fields
- Flashcards saved to Supabase with default template = 'basic', rating = null
- On success, user redirected to flashcard review page
- On failure, error logged and user shown: "Flashcard generation failed. Please try again."
- Regenerating flashcards overwrites existing ones for that session after confirmation

---

## 11. Flashcard Review

**As a student, I want to flip through flashcards and rate them so that I know which concepts I know well and which need more work.**

Acceptance criteria:
- Flashcard review page shows one card at a time
- Card shows front (question) by default
- Clicking card or pressing Space flips to back (answer)
- Three rating buttons shown after flip: Easy, Hard, Again
- Rating saved to flashcards.rating in Supabase immediately on click
- Progress shown: "Card 4 of 18"
- After last card, summary shown: X Easy, Y Hard, Z Again
- "Review Again" button restarts the session
- Keyboard shortcuts: Space = flip, 1 = Easy, 2 = Hard, 3 = Again
- Cards with no rating shown first, then rated cards

---

## 12. Anki Export Editor + .apkg Export

**As a student, I want to edit and export my flashcards as an Anki deck so that I can continue studying with spaced repetition.**

Acceptance criteria:
- "Export to Anki" button on flashcard review page and session detail page
- Export page shows all flashcards in an editable list
- Each card row has editable front and back text fields
- Field order toggle per card: front-back or back-front
- Template selector per card: Basic or Cloze
- "Add Card" button to manually add new cards
- "Delete Card" button per card with immediate removal (no confirmation needed)
- "Export" button generates .apkg file using anki-apkg-export library
- Initial ease factors baked into export based on ratings: Easy = 2.7, Hard = 1.8, Again = 1.3, unrated = 2.5
- Download triggered automatically in browser
- Export works entirely client-side — no server call

---

## 13. Note Export (PDF / DOCX / TXT)

**As a student, I want to export my notes in different formats so that I can share or store them outside the app.**

Acceptance criteria:
- Export options shown on note viewer page: PDF, DOCX, TXT
- PDF generated client-side using jsPDF
- DOCX generated client-side using docx.js
- TXT generated as plain text Blob download
- Exported file named: [session-title]-notes.[ext]
- For free users: export count checked against export_counts table before download
- If free user has reached 6 exports today, show: "Daily export limit reached. Upgrade to Premium for unlimited exports."
- Export count incremented in Supabase after successful download
- Premium users have no export limit check
- All exports include all four note sections: Summary, Key Terms, Main Points, Action Items

---

## 14. File Upload → Generate Notes

**As a student, I want to upload a lecture file and generate structured notes so that I don't have to paste text manually.**

Acceptance criteria:
- New session page has drag-and-drop file upload zone alongside paste text area
- Accepted file types: PDF, DOCX, TXT, JPG, PNG
- PPTX files show message: "Please export your PowerPoint as PDF before uploading."
- File size limit: 10MB
- Text-based PDF: parsed client-side using pdfjs-dist
- If pdfjs returns empty string: "This PDF appears to be image-based. Sending to AI for processing..." then routes to Groq vision API
- DOCX: parsed client-side using mammoth.js
- TXT: read via FileReader API
- JPG/PNG: sent directly to Groq vision API via Next.js API route
- Extracted text shown in preview before generating notes (user can edit)
- Session saved with source = 'import'
- All other note generation acceptance criteria from Feature 5 apply

---

## 15. Fake Premium Gate + Privacy Consent

**As a student interested in premium, I want to sign up for premium access so that I can use audio recording features.**

Acceptance criteria:
- "Get Premium" button visible in navbar and on restricted feature pages
- Clicking opens a modal with: Full Name field, Student Email field, Submit button
- Form validates that both fields are filled before enabling Submit
- On submit, data saved to premium_signups table in Supabase
- profiles.plan updated to 'premium' for that user immediately
- Modal closes and premium features unlock without page reload
- Before audio recording is available, a one-time privacy consent screen shown:
  - Text: "Audio is processed by Groq (US). It is not stored after transcription."
  - Checkbox: "I understand and consent"
  - Consent state stored in Supabase (boolean column on profiles)
- Users who have already consented skip the consent screen on subsequent recordings
- "Get Premium" button replaced with "Premium" badge after signup

---

## 16. Audio Recording + Transcription Queue (Premium)

**As a premium student, I want to record my lecture directly in the app and receive structured notes automatically so that I don't need to do anything manually during class.**

Acceptance criteria:
- Recording page accessible to premium users only
- Free users see: "Audio recording is a Premium feature" with upgrade prompt
- "Start Recording" button activates microphone via MediaRecorder API
- Recording timer shown: "Recording: 12:34"
- "Stop Recording" button ends capture
- Audio saved to IndexedDB immediately on stop (survives tab close)
- Audio chunked into 30-second segments in native MediaRecorder format (WebM/OGG/MP4 — no WAV conversion)
- Each chunk sent sequentially to Groq Whisper via Next.js API route
- Progress shown: "Transcribing... 4/18 chunks"
- On 429 rate limit: exponential backoff (2s, 4s, 8s, 16s, max 60s), user shown: "Rate limited. Resuming in ~Xs"
- Groq rate limit headers monitored: x-ratelimit-remaining-requests, x-ratelimit-reset
- On network loss: "Waiting for connection..." shown, retries automatically
- Queue state persisted in IndexedDB — resumes on page reload
- On completion: transcript stitched, note generation triggered automatically
- Browser notification fired: "Your notes for [session title] are ready"
- In-app notification created in notifications table
- Session saved with source = 'recording'

---

## 17. Folders + Tags (Premium)

**As a premium student, I want to organize my sessions into folders and tag them so that I can find related notes easily.**

Acceptance criteria:
- Folders and tags visible only to premium users
- "New Folder" button on dashboard creates a folder (name required, max 50 chars)
- Sessions can be added to one or more folders via a dropdown on session card
- "New Tag" button creates a tag (name required, max 30 chars, unique per user)
- Sessions can be tagged with one or more tags via a tag picker on session card
- Session list can be filtered by folder or tag
- Filtering by folder shows only sessions in that folder
- Filtering by tag shows only sessions with that tag
- Folders and tags can be renamed and deleted
- Deleting a folder does not delete its sessions
- Deleting a tag removes it from all sessions

---

## 18. In-App Notification Bell

**As a student, I want to see a notification history in the app so that I don't miss updates if I close the browser notification.**

Acceptance criteria:
- Bell icon in navbar with unread count badge
- Clicking bell opens a dropdown showing last 20 notifications
- Each notification shows: message, relative time ("2 minutes ago")
- Unread notifications highlighted
- Clicking a notification marks it as read and navigates to the relevant session
- "Mark all as read" button in dropdown header
- Notifications older than 30 days not shown (deleted by scheduled function)
- Bell badge count updates in real-time via Supabase Realtime subscription
- Empty state: "No notifications yet."

---

## 19. Admin Dashboard

**As Shimul (admin), I want a dashboard showing usage metrics and errors so that I can monitor the app and fix issues.**

Acceptance criteria:
- Admin dashboard route: /admin
- Gated server-side: process.env.ADMIN_EMAIL === session.user.email. Returns 404 for all other users.
- Metrics displayed:
  - Total registered users (count from profiles)
  - Daily active users (sessions created today)
  - Weekly active users (sessions created this week)
  - Notes generated (count from notes)
  - Flashcards generated (count from flashcards)
  - Premium signups (list with name, email, created_at)
  - File upload type breakdown (count by sessions.source)
  - Audio transcription sessions (count + avg duration where source = 'recording')
  - Note export count by format (PDF/DOCX/TXT from export_counts)
  - Error log list: error_type, error_message, user_id, session_id, created_at (most recent 50)
- Username search: input field to find user by @username
- All data fetched via Supabase service role key (bypasses RLS) in Next.js API route
- Admin dashboard has no edit or delete capabilities — read only
- Dashboard is not linked from any user-facing page
