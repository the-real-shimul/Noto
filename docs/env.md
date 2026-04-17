# Noto — Environment Variables

## Rules
- Never commit `.env.local` to Git. It is already in `.gitignore` by default in Next.js.
- Variables prefixed with `NEXT_PUBLIC_` are exposed to the browser. Never put secrets in them.
- Server-side only variables (no `NEXT_PUBLIC_` prefix) are never accessible client-side.
- All variables listed here are required. The app will not function without them.
- Set all variables in Vercel dashboard under Project → Settings → Environment Variables for production.

---

## .env.local (local development)

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=

# Groq
GROQ_API_KEY=

# Admin
ADMIN_EMAIL=

# Vercel Cron
CRON_SECRET=

# App
NEXT_PUBLIC_APP_URL=
```

---

## Variable Reference

### NEXT_PUBLIC_SUPABASE_URL
- **Safe client-side:** Yes
- **Where to find:** Supabase dashboard → Project Settings → API → Project URL
- **Used for:** Initializing Supabase JS client on both client and server

### NEXT_PUBLIC_SUPABASE_ANON_KEY
- **Safe client-side:** Yes (designed to be public, protected by RLS)
- **Where to find:** Supabase dashboard → Project Settings → API → anon public key
- **Used for:** All client-side Supabase queries (RLS enforced)

### SUPABASE_SERVICE_ROLE_KEY
- **Safe client-side:** NO — server-side only
- **Where to find:** Supabase dashboard → Project Settings → API → service_role secret key
- **Used for:** Admin routes (bypasses RLS), account deletion, cron cleanup
- **Warning:** Never expose this key. If leaked, it bypasses all row-level security.

### GROQ_API_KEY
- **Safe client-side:** NO — server-side only
- **Where to find:** console.groq.com → API Keys
- **Used for:** All Groq API calls (note generation, flashcard generation, transcription, vision)
- **Warning:** Never expose this key. All Groq calls go through Next.js API routes.

### ADMIN_EMAIL
- **Safe client-side:** NO — server-side only
- **Value:** Your email address (e.g. your Google login email)
- **Used for:** Gating /api/admin/* routes and the /admin dashboard page
- **Note:** Must exactly match the email in your Supabase Auth user record

### CRON_SECRET
- **Safe client-side:** NO — server-side only
- **Value:** Generate a random string (e.g. `openssl rand -base64 32` in terminal)
- **Used for:** Validating that /api/cron/cleanup requests come from Vercel cron, not external callers
- **Set in Vercel:** Yes — Vercel automatically attaches this as a Bearer token to cron requests

### NEXT_PUBLIC_APP_URL
- **Safe client-side:** Yes
- **Local value:** `http://localhost:3000`
- **Production value:** Your Vercel deployment URL (e.g. `https://noto.vercel.app`)
- **Used for:** Supabase OAuth redirect URLs — ensures Google OAuth redirects correctly in both local and production environments

---

## Vercel Setup Checklist
1. Go to Vercel dashboard → your project → Settings → Environment Variables
2. Add all 7 variables above
3. For `NEXT_PUBLIC_APP_URL` set production value to your actual Vercel URL
4. For `CRON_SECRET` use the same value as in `.env.local`
5. Redeploy after adding variables

## Supabase OAuth Redirect Setup
In Supabase dashboard → Authentication → URL Configuration:
- Site URL: `https://your-app.vercel.app`
- Redirect URLs: add both `http://localhost:3000/**` and `https://your-app.vercel.app/**`
