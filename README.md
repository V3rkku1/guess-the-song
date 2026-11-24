# Guess the Song

Hear 0.1 s of a song, then more after each miss (up to 15 s). The fewer seconds you need, the more points you get. It's a single `index.html` with no build step. Song clips come from the iTunes Search API, and artist photos come from Deezer.

## Hosting (GitHub Pages)

1. In the repo, go to **Settings → Pages**.
2. Set **Source** to *Deploy from a branch*, **Branch** to `main`, and the folder to `/ (root)`. Click **Save**.
3. After a minute or two, the game is live at `https://<user>.github.io/guess-the-song/`.

On a free GitHub plan, the repo must be public for Pages to work.

## Online leaderboard (optional, free)

The leaderboard uses [Supabase](https://supabase.com), which has a free tier. Until you set it up, the game works normally and the Leaderboard page says "not set up yet".

1. Create a Supabase project.
2. Open **SQL Editor** and run:

```sql
create table public.scores (
  id         bigint generated always as identity primary key,
  created_at timestamptz not null default now(),
  name       text not null check (char_length(trim(name)) between 1 and 20),
  board      text not null check (board ~ '^(r(5|10|15|20)|daily-\d{4}-\d{2}-\d{2})$'),
  mode       text not null check (mode in ('type', 'mc')),
  label      text not null default '' check (char_length(label) <= 40),
  total      int  not null check (total between 1 and 20),
  correct    int  not null,
  score      int  not null,
  -- highest possible score: 1000 per round + 50 streak bonus per round already in the streak
  constraint correct_ok check (correct between 0 and total),
  constraint score_ok   check (score between 0 and total * 1000 + 25 * total * (total - 1))
);
create index on public.scores (board, mode, score desc);
create index on public.scores (created_at);

alter table public.scores enable row level security;
create policy "anyone can read"   on public.scores for select to anon using (true);
create policy "anyone can submit" on public.scores for insert to anon with check (true);
-- no update/delete policies: scores can't be edited or wiped from the browser
grant select, insert on public.scores to anon;

-- flood guard: at most 60 new scores per minute across everyone
create function public.scores_rate_limit() returns trigger language plpgsql as $$
begin
  if (select count(*) from public.scores where created_at > now() - interval '1 minute') >= 60 then
    raise exception 'rate limited' using errcode = 'P0001';
  end if;
  return new;
end $$;
create trigger scores_rate_limit before insert on public.scores
  for each row execute function public.scores_rate_limit();
```

3. In **Project Settings → API**, copy the **Project URL** and the **publishable** key (older projects call it the **anon** key).
4. Paste them at the top of the `<script>` in `index.html`:

```js
const SUPABASE_URL = 'https://xxxx.supabase.co';
const SUPABASE_KEY = 'sb_publishable_...';
```

5. Commit and push. Pages redeploys on its own.

**About the key:** this key is meant to be public, so it's safe to put in the page. The `service_role` / secret key must never go in the page.

**About cheating:** scores are calculated in the player's browser. Someone determined could post a fake score, but never one above the highest possible score. To remove a bad entry, open **Table Editor → scores** in Supabase and delete the row.

To delete old daily scores every so often:

```sql
delete from public.scores where board like 'daily-%' and created_at < now() - interval '30 days';
```

## Load notes

- Each player's browser talks to iTunes and Deezer directly, so traffic is spread across players' IP addresses. The site itself is static files.
- Home-page artwork and photos are cached in the browser for 7 days. Identical searches are reused for the whole session.
- The leaderboard is fetched only when someone opens that page, and it's cached for 30 seconds.
