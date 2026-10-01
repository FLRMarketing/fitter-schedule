# FLR Fitter Schedule

A mobile-first web app, in the same look as the [FLR Hub](https://flrmarketing.github.io/flr-hub/) (which links to it), that shows, for each day from 30 days ago to 30 days ahead (the window moves on by itself every day), which fitters are **free**, **working** or **off**, what each is working on, and which jobs on the Work Programme still **need a fitter**. It reads the Monday.com *Work Programme* board (ID 5094652880) through a small Cloudflare Worker, so no Monday token ever reaches a phone.

```
Phone / browser  ──►  GitHub Pages (index.html)  ──►  Cloudflare Worker (holds the Monday token)  ──►  monday.com API
                       static, public                  free tier, checks a team passcode
```

## What's in the repo

| File | Purpose |
|---|---|
| `index.html` | The whole app — HTML, CSS and JavaScript in one file. Edit `WORKER_URL` near the top of the script. |
| `manifest.webmanifest`, `icon*.png`, `icon.svg`, `apple-touch-icon.png` | Lets the team "Add to Home Screen" so it opens like an app. |
| `worker/worker.js` | The Cloudflare Worker. One fixed, read-only query; no arbitrary GraphQL from the browser. |
| `test/` | A snapshot of real board data (`fixture.json`) and a Playwright script used to check the page. Not needed in production; delete if you like. |

## How the schedule is worked out

The worker fetches every subitem on *Subitems of Work Programme* whose **Timeline end date is on or after the `from` date** the page asks for (so multi-day jobs that started earlier still show), together with its parent job's name, group, WFM number, site address and postcode, plus the full label list of the **Fitter** dropdown (the roster).

The page asks twice: from **today**, as it always has, and from **30 days ago**, which adds the subitems that had already ended. If that second request fails, ignores the date or comes back cut short (it must hold everything the first one does, and a round count such as 500 looks like a page limit), today onwards is unaffected and the earlier days say what's missing (*Not loaded*, or *Some earlier jobs may be missing*). It then does the rest:

| Parent job's group | Treated as |
|---|---|
| Scheduling Issue, Ready to schedule, Scheduled | **Work.** A subitem with no fitter is listed under *Needs a fitter* (on days already gone: *No fitter assigned*, as a record). |
| FLR jobs (Fleet Bookings, Training) | **Work**, tagged *FLR*. Never flagged as needing a fitter. |
| Holidays, Sickness, Unavailable & Leave | **Off** (Unavailable). |
| National Holidays & Weekend Rota | **Off** for the fitters listed on that weekend's *Not working* entry. Can be switched off in Settings. |
| Completed | Ignored unless *Include jobs already marked Completed* is on in Settings. |
| New Job, Supply Only, Cancelled | Ignored. |

Subitem **Status** refines this: `Flags` entries (e.g. "Needs Van") show as an amber tag on the fitter, not as work; `Annual Leave` / `Not Working` / `Unpaid Leave` / `Unpaid Sick` / `Appointments` count as Off wherever they appear; `Supply Only` is ignored.

A fitter's state on a day is **Working** if they have any work entry overlapping that day, otherwise **Off** if they have a leave entry, otherwise **Free**. Work *and* leave on the same day shows as **Clash**.

The roster is every label on the Fitter dropdown except deactivated labels and anything beginning `ZZ Left` / `ZZleft`. `ZFLR -` and `ZSub -` labels get a small *FLR* / *Sub* tag and can be hidden in Settings. The text after the dash on each label (the van code, e.g. `K4`) is shown as a chip.

Tapping any job opens it in Monday.

## Deploying

### 1. Cloudflare Worker (about 5 minutes)

1. Create a **monday.com API token**: Monday → your avatar → *Developers* → *My access tokens*. A token from a user with read access to the Work Programme board is enough. (A dedicated "FLR Apps" viewer user is the tidiest option.)
2. Sign in at <https://dash.cloudflare.com> (free plan is fine) → **Workers & Pages** → **Create** → **Create Worker**. Name it `flr-fitter-schedule` and deploy the placeholder.
3. Click **Edit code**, replace the contents with `worker/worker.js`, and **Deploy**.
4. Go to the worker's **Settings → Variables and Secrets** and add:
   - `MONDAY_TOKEN` — *Secret* — the token from step 1
   - `APP_KEY` — *Secret* — a passcode of your choosing that the team will type in once (e.g. a few words)
   - `ALLOWED_ORIGIN` — *Text* — your GitHub Pages origin, e.g. `https://flr-group.github.io` (optional but recommended; leave out while testing)
5. Note the worker URL, e.g. `https://flr-fitter-schedule.flr.workers.dev`. Test it in a browser: `https://…workers.dev/schedule?key=YOUR_PASSCODE` should return JSON starting `{"generatedAt": …`.

Cloudflare's free tier allows 100,000 requests a day. The worker also reuses a response for 45 seconds, so a team hammering Refresh can't run up the Monday API budget.

### 2. GitHub Pages

1. Open `index.html` and set `WORKER_URL` to your worker URL (no trailing slash).
2. Create a repository (public or private — Pages works with either on a paid plan; public on free), push these files to the `main` branch.
3. **Settings → Pages → Build and deployment → Source: Deploy from a branch → `main` / `(root)`** → Save.
4. After a minute the site is at `https://<org>.github.io/<repo>/`. Open it, enter the passcode, and you should see today's roster.

Optional: if you put the app on a custom domain, add that domain as `ALLOWED_ORIGIN` on the worker instead.

### 3. On the phones

Open the link in Safari (iPhone) or Chrome (Android) → *Share* / menu → **Add to Home Screen**. It opens full-screen like an app. The passcode is remembered per phone; the last-loaded schedule is kept so the page still opens with a "showing data from…" banner when there's no signal.

## Day-to-day

- **Refresh** asks the worker for fresh data (bypasses the 45-second cache). The page also refreshes itself when reopened after 10 minutes.
- **Day** tab: pick a date from the strip, which runs from 30 days back to 30 days ahead, opens on today, names each new month and greys the days already gone. Swipe it on a phone; with a mouse, use the arrows at its ends; the arrow keys and Home / End work too. *Today* (or tapping *Day* again) comes back to today. *Needs a fitter* is pinned at the top, then Working, Free and Unavailable. The All / Free / Working / Off control filters; search matches fitter or job names.
- **Fitter** tab: one fitter's days from today to 30 days ahead, with the 30 days before folded under *Earlier* — good for fitters bookmarking their own view (the selection is remembered).
- **Jobs** tab: each job in the window once, under the day it starts (jobs already running under today; jobs already over folded under *Earlier*, which opens by itself when a search finds one), with all its fitters together. Tap a job to open it: each person's dates, their subitem's status (*Scheduled*, or *TBC* still to be confirmed) and, for administrators, today's arrival and departure; then *Open in monday.com*. *Needs fitter* shows only jobs with a subitem that has no fitter yet and isn't over.
  - **What counts as one job:** a Work Programme item. Its subitems come together because they belong to it (the parent item's ID), so copies ("(copy)") and differently named subitems of one item show as one job, and separate items stay separate however alike their names are.
- **Settings** (cog): theme (Auto / Light / Dark), weekends, weekend rota, FLR/Sub rows, Completed jobs.
- **Photos and number plates**: in a browser that is signed in on the [FLR Hub](https://flrmarketing.github.io/flr-hub/) (same site), each fitter shows their staff photo and the registration FleetView has for them, as in Fleet Management. They come from the FLR database (`public.fitter_faces()`) after sign-in, never from this repository; without a Hub sign-in the page shows initials and van codes. The page loads the Hub's `flr-config.js` and `vendor/supabase-2.116.0.js` for this.
- **Arrivals (FLR administrators only)**: when the fitter's van reached the site, from FleetView.
  - **On the job card:** "On site · Arrived 07:27" (green), "Arrived 07:37 · left 13:49", or "Arrival not confirmed" (grey outline). Hovering shows why.
  - **Tapping the label** shows that job's arrivals day by day. The location-pin button in the header lists the last 30 days.
  - **In the Jobs tab:** each fitter's chip has today's time; an opened job has the labels and an *Arrivals* button with all its fitters' arrivals.
  - **Where it comes from:** the FLR database (`public.fitter_arrivals()`), kept up to date every two minutes by the `flr-arrivals` Edge Function (in the Cost Estimator repository, with its rules and setup).
  - **Everyone else** sees the page exactly as before.

## Changing the passcode or token

Update the secret in Cloudflare → *Variables and Secrets*. Phones will get "passcode was not accepted" and prompt for the new one; use *Change passcode* at the foot of the page at any time.

## Column and board IDs (for reference)

| | ID |
|---|---|
| Work Programme board | `5094652880` |
| Subitems board | `5094670077` |
| Subitem Timeline | `timerange_mm2eg6h0` |
| Subitem Fitter (dropdown) | `dropdown_mm2e5fnf` |
| Subitem Status | `color_mm2e63k0` |
| Job WFM Number / Site Address / Site Postcode | `text_mm2egghn` / `text_mm2n78j3` / `text_mm2nqs1a` |

If a column is renamed nothing breaks — IDs don't change. If a **group** is renamed, update the group names at the top of the script in `index.html`.

## Local testing

```
python3 -m http.server 8123
# then open http://localhost:8123/index.html?worker=http://localhost:8123/test
```

`test/schedule` is a symlink to `fixture.json`, so the page loads the snapshot instead of the worker. Any passcode works locally.
