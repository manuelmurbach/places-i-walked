# Setting up the online Scratch Map

This folder is a multi-user version of the scratch map: each visitor signs in
with an email magic link and gets their own saved scratch data, stored in a
free Supabase project. The page itself is static and can be hosted on GitHub
Pages — Supabase is just an API it talks to over HTTPS.

You need to do the account/project creation steps yourself (steps 1–4) since
that's your Supabase account. Everything after that is just editing/pushing
files.

## 1. Create a Supabase project

1. Go to https://supabase.com and sign up (free tier is enough).
2. Click **New project**. Pick any name/region and a database password
   (you won't need the password day-to-day — Supabase manages auth for you).
3. Wait ~2 minutes for it to finish provisioning.

## 2. Create the data table

In the Supabase dashboard, open **SQL Editor** → **New query**, paste this,
and run it:

```sql
create table public.scratches (
  user_id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{"world":[],"switzerland":[]}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table public.scratches enable row level security;

create policy "select own row"
  on public.scratches for select
  using (auth.uid() = user_id);

create policy "insert own row"
  on public.scratches for insert
  with check (auth.uid() = user_id);

create policy "update own row"
  on public.scratches for update
  using (auth.uid() = user_id);
```

Row Level Security is what keeps one user from ever reading or overwriting
another user's scratch data, even though everyone shares the same public
anon key.

## 3. Configure email auth

Supabase has email magic-link sign-in on by default, but the redirect needs
to point at your real URL:

1. Go to **Authentication → URL Configuration**.
2. Set **Site URL** to your future GitHub Pages URL, e.g.
   `https://YOUR-USERNAME.github.io/YOUR-REPO/`
3. Add the same URL under **Redirect URLs**.
   (While testing locally, also add `http://localhost:PORT/` here.)

## 4. Get your API keys

Go to **Settings → API**. Copy:
- **Project URL** (looks like `https://abcxyz.supabase.co`)
- **anon public** key (long string starting with `eyJ...`)

This anon key is *meant* to be public — it's safe to commit to a public
GitHub repo. It only allows what your RLS policies above allow.

## 5. Paste the keys into the app

Open [index.html](index.html) and replace these two lines near the top of
the `<script>` block:

```js
const SUPABASE_URL = "https://YOUR-PROJECT-REF.supabase.co";
const SUPABASE_ANON_KEY = "YOUR-ANON-PUBLIC-KEY";
```

## 6. Deploy to GitHub Pages

1. Push this repo to GitHub (public repo is fine — the anon key is safe,
   and the site itself will be public either way once Pages is on).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment**, choose **Deploy from a branch**, pick
   your branch, and set the folder to `/scratch-map/online` (or move
   `index.html` + `favicon.svg` to their own repo root/branch if you'd
   rather not fuss with the subfolder path).
4. GitHub gives you the live URL — make sure it matches what you set in
   step 3 above (Site URL / Redirect URLs in Supabase).

## Trying it out

1. Visit your Pages URL.
2. Enter your email, leave "Keep me logged in" checked, click **Send magic
   link**.
3. Open the email Supabase sends and click the link — it'll redirect back
   into the app, now signed in.
4. Scratch some areas, reload the page — they should still be there.
5. Uncheck "Keep me logged in" before signing in on a shared/public
   computer — that keeps the session in `sessionStorage` instead of
   `localStorage`, so it clears when the browser tab closes.

## What didn't come along from the offline version

- The hardcoded personal travel-history dots (`LOCATION_DATA`) were removed
  on purpose — anything baked into this HTML is visible to every visitor via
  "View Source", regardless of whether the git repo is public or private.
  Your original offline copy in `../scratch_map.html` still has them; keep
  using that one for your own personal map.
- The "Save to local file" (File System Access) button was dropped since
  Supabase is now the source of truth. **Export**/**Import** (JSON download/
  upload) are still there as a manual backup option.
