# 📺 WatchLog

A shared anime release tracker for you and your friends. See what airs each day, track your progress, know when the **English dub** drops, and browse what everyone else is watching — all in one place, synced live.

WatchLog is a single self-contained HTML file. No build step, no framework, no server of your own — it runs entirely in the browser and stores shared data in a free [Supabase](https://supabase.com) database.

> **Live site:** https://qclayton15.github.io/watchlog/

---

## ✨ Features

- **Auto schedule from [AniList](https://anilist.co)** — add a show and it pulls the air day, air time, and a live next-episode countdown automatically.
- **Local timezones** — everyone sees air times converted to their own timezone from UTC.
- **Three views** — a Sun→Sat weekly calendar, a detailed list, and a poster wall.
- **Accounts & profiles** — email/password login; pick a display name, avatar (emoji or image URL), and accent color.
- **Personal lists, shared space** — everyone has their own list with their own progress/status/rating/notes, and can view anyone else's list read-only. Tap **＋ Add to mine** to copy a show you spotted on a friend's list.
- **Community dub tracking** — no public API exposes English dub dates, so the group tracks the latest dubbed episode together with a shared, bumpable counter.
- **Delay-aware** — auto-re-syncs when a countdown elapses, shows an honest "expected…/checking" state, and offers a shared "delayed this week" flag that clears itself once the episode airs.
- **Bulk import** from an AniList username (MyAnimeList via the AniList bridge — see notes).
- **Genre browse, trending, search-as-you-type, filters & sorting.**
- **Ratings** with a per-show group average, **streaming links** (Crunchyroll/Netflix/etc.), **calendar (.ics) export**, and optional **browser notifications** ~1h before a show airs.
- **Light/dark themes, per-user accent tint, and a mobile-friendly layout** (add to home screen for an app-like feel).

---

## 🧩 How it works

- **Frontend:** one `index.html` (HTML + CSS + vanilla JS, no dependencies bundled — Supabase's JS client loads from a CDN).
- **Anime data:** the public [AniList GraphQL API](https://docs.anilist.co) (no key required).
- **Storage & sync:** a Supabase Postgres database with Row Level Security and realtime. The app talks to it with a **publishable** key that's safe to ship in the file.

There is no backend to run — hosting is just serving a static file.

---

## 🚀 Self-hosting setup

### 1. Create a Supabase project
Sign up at [supabase.com](https://supabase.com), create a new project (free tier is fine), and wait for it to finish provisioning.

### 2. Create the database
Open **SQL Editor → New query**, paste the schema below, and **Run**:

```sql
-- ---------- Tables ----------
create table if not exists shows (
  id         uuid primary key default gen_random_uuid(),
  board_id   text not null default 'main',
  user_id    uuid default auth.uid(),
  anilist_id bigint,
  title      text,
  cover      text,
  site_url   text,
  status     text default 'Watching',
  progress   int  default 0,
  rating     int,
  dub_ep     int,
  delay_ep   int,
  notes      text default '',
  created_at timestamptz default now()
);

create table if not exists profiles (
  id         uuid primary key references auth.users on delete cascade,
  name       text,
  avatar     text,
  color      text,
  updated_at timestamptz default now()
);

-- ---------- Row Level Security ----------
-- Anyone may read (public viewing); only owners may change their own rows.
alter table shows enable row level security;
drop policy if exists "shows read"   on shows;
drop policy if exists "shows insert" on shows;
drop policy if exists "shows update" on shows;
drop policy if exists "shows delete" on shows;
drop policy if exists "shows claim"  on shows;
create policy "shows read"   on shows for select using (true);
create policy "shows insert" on shows for insert with check (auth.uid() = user_id);
create policy "shows update" on shows for update using (auth.uid() = user_id);
create policy "shows delete" on shows for delete using (auth.uid() = user_id);
create policy "shows claim"  on shows for update using (user_id is null) with check (auth.uid() = user_id);

alter table profiles enable row level security;
drop policy if exists "profiles read"   on profiles;
drop policy if exists "profiles insert" on profiles;
drop policy if exists "profiles update" on profiles;
create policy "profiles read"   on profiles for select using (true);
create policy "profiles insert" on profiles for insert with check (auth.uid() = id);
create policy "profiles update" on profiles for update using (auth.uid() = id);

-- ---------- Realtime ----------
alter publication supabase_realtime add table shows;
alter publication supabase_realtime add table profiles;
```

### 3. Make signup instant (optional but recommended)
**Authentication → Sign In / Providers → Email → turn off "Confirm email" → Save.** New users can then log in immediately instead of clicking an email link.

### 4. Add your keys to the app
Get them from **Project Settings → API** (URL) and **API Keys → Publishable key**. Open `index.html` in a text editor and fill in the `CONFIG` block near the top:

```js
const CONFIG = {
  SUPABASE_URL:      "https://YOUR-PROJECT.supabase.co",
  SUPABASE_ANON_KEY: "sb_publishable_xxxxxxxxxxxx",   // the Publishable key
};
```

> The publishable key is designed to be public — your data is protected by the RLS policies above, so it's safe to commit to a public repo.

### 5. Deploy
Any static host works. **GitHub Pages:** create a public repo, upload `index.html`, then **Settings → Pages → Deploy from a branch → `main` / root**. Your site publishes at `https://<username>.github.io/<repo>/`.

Share that link — everyone who opens it and logs in shares the same board. Use `?board=NAME` on the URL to run separate groups off the same database.

---

## 🔄 Updating
Replace `index.html` in your repo (edit or re-upload) and commit — Pages redeploys automatically in ~1 minute, same link. If a future version writes a new database field, add it with `alter table shows add column if not exists <name> <type>;`.

---

## 📖 Using WatchLog
A friendly end-user walkthrough (great for pasting into Discord) lives in [`watchlog-guide-discord.md`](./watchlog-guide-discord.md). In short: log in, set your profile, add shows via search / genre / trending / import, and track progress, dubs, and delays from the cards.

---

## ⚠️ Notes & limitations
- **Dub dates** are community-maintained (a shared "latest dub episode" counter) because no free API publishes English dub schedules. AniList auto-detection fills it in on the rare shows where AniList lists dubbed episodes.
- **MyAnimeList import**: MAL's public list API is restricted. Import your MAL list into AniList (AniList → Settings → Import) and then import from the AniList tab.
- **Notifications** fire only while the app tab is open (no background push).
- **Air schedules** reflect the original Japanese/sub broadcast (what AniList tracks).

---

## 🙏 Credits
Anime data from **[AniList](https://anilist.co)**. Database, auth, and realtime by **[Supabase](https://supabase.com)**. Built as a fun project for tracking anime with friends. 📺✨
