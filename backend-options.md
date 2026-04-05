# Seminarios Explorer: Backend Options

Notes on how to turn the single-player localStorage map into a collaborative, multi-player event log. Written April 2026.

## Current State

`explorer.html` stores all events and notes in `localStorage` as JSON. This is perfect for solo use — zero infrastructure, instant, works offline. The export/import JSON feature bridges the gap for manual collaboration: everyone exports their JSON, one person merges, distributes the master, everyone imports.

The "backend" question is really: *how do we sync JSON between browsers automatically?*

---

## Option 1: Git-as-Backend (JSON file in the repo)

**How it works:** A `data/events.json` file lives in the repo. The explorer page fetches it on load and merges with localStorage. To contribute, players either:
- Edit the JSON directly and push/PR (GitHub-pilled friends only)
- Use a simple form that commits via the GitHub API

**Pros:**
- No server, no hosting costs, no database
- Full version history for free (it's git!)
- Works with GitHub Pages (the site already lives here)
- The "export JSON" feature you already have is 90% of this

**Cons:**
- GitHub API writes require a personal access token or OAuth app
- Merge conflicts if two people commit simultaneously (rare for a small group)
- Not real-time — you'd need to refresh to see others' changes

**Effort:** Low-medium. Fetch JSON on page load is trivial. Writing back via GitHub API is ~50 lines of JS + a token.

**Best for:** Your exact situation — small group of friends, async contributions, already on GitHub.

---

## Option 2: Firebase / Supabase (Real-time Database)

**How it works:** A cloud database (Firestore, Supabase Postgres, etc.) stores events. The explorer connects directly from the browser — no server code needed. Changes sync in real-time across all connected browsers.

**Pros:**
- Real-time sync — everyone sees new events instantly
- Built-in auth (Google sign-in, magic links, etc.)
- Generous free tiers (Firebase Spark: 1GB storage, 50k reads/day; Supabase: 500MB, unlimited API)
- No server to maintain

**Cons:**
- External dependency (vendor lock-in, though data is exportable)
- Slightly more complex JS (Firebase SDK or Supabase client)
- Need to set up a project, configure auth rules

**Firebase specifically:**
- Firestore is document-based — maps naturally to your events/notes structure
- `onSnapshot` gives you live updates with ~3 lines of code
- Free tier is absurdly generous for a campaign tracker

**Supabase specifically:**
- Postgres under the hood — more flexible queries if you want them later
- Real-time subscriptions via websockets
- Row-level security lets you say "users can only delete their own events"
- Open source — you could self-host on the Raspberry Pi if you wanted

**Effort:** Medium. ~1-2 hours to set up a Firebase project and swap localStorage calls for Firestore calls. The data model is basically identical.

**Best for:** If you want real-time sync and your friends aren't all GitHub-comfortable.

---

## Option 3: Simple JSON Server (Self-hosted)

**How it works:** A tiny Node.js/Python server (or even `json-server`, which is literally one npm package) serves and accepts JSON over HTTP. Host it on the Raspberry Pi, a free Render/Fly.io instance, or anywhere.

**Pros:**
- Full control over everything
- Dead simple — `json-server` turns a JSON file into a REST API with zero code
- Can run on your rpi5 (it's already in your fleet)

**Cons:**
- Need to keep a server running (the rpi is perfect for this)
- Need to handle conflicts if two people POST simultaneously
- No built-in auth (though for a friend group, maybe that's fine)
- Not real-time without adding websockets

**`json-server` example:**
```bash
npm install -g json-server
echo '{"events":[],"notes":{}}' > db.json
json-server --watch db.json --port 3000
```
That's it. You now have `GET/POST/PUT/DELETE /events` and `/notes`. The explorer JS swaps `localStorage` calls for `fetch()` calls.

**Effort:** Low. Genuinely ~30 minutes if you already have Node on the rpi.

**Best for:** Tinkerers who want full control and have a machine that's always on.

---

## Option 4: Shared Google Sheet / Notion API

**How it works:** Events are rows in a Google Sheet or entries in a Notion database. The explorer reads/writes via their respective APIs.

**Pros:**
- Your friends already know how to use Sheets/Notion
- Can edit data outside the map (nice for bulk entry from notebooks)
- Free

**Cons:**
- API rate limits (Google: 60 req/min; Notion: 3 req/sec)
- Slower reads than a real database
- Google Sheets API requires OAuth setup; Notion needs an integration token
- Data model is flattened (tabular) which is a bit awkward for nested notes

**Effort:** Medium. Google Sheets API is well-documented but auth is a pain. Notion API is cleaner but slower.

**Best for:** If the party wants to also browse/edit events outside the map interface.

---

## Recommendation

For your situation, I'd go in this order:

1. **Now:** Export/Import JSON (already done). Manual but works today.

2. **Soon:** Git-as-backend. Commit a `data/events.json` to the repo. The explorer fetches it on load and merges. You (as repo owner) periodically merge everyone's exported JSONs and push. Your friends don't even need to know git — they just export and send you the file.

3. **When you're ready:** Firebase or Supabase. This is the "real" solution. Firebase is probably the fastest path — you'd create a project, drop in the SDK, and replace ~6 function calls. Supabase if you want Postgres and the option to self-host later.

4. **For fun:** json-server on the rpi5. Because running your campaign's backend on a Raspberry Pi is objectively cool.

---

## Data Model

Whatever backend you pick, the data shape stays the same. This is what `localStorage` already stores:

```json
{
  "events": [
    {
      "id": "m1abc123",
      "district": "12_market",
      "day": "day-3",
      "time": "evening",
      "text": "Gondgieaux haggled with the potion merchant and got scammed",
      "character": "gondgieaux",
      "timestamp": 1712345678000
    }
  ],
  "notes": {
    "12_market": "The potion merchant's name is Vex. Don't trust him.",
    "23_red-light": "We haven't been here yet but the GM keeps hinting..."
  }
}
```

For a multi-user backend, you'd add a `userId` or `characterId` to each event, and potentially make notes per-user rather than shared. But even that is a small tweak.

---

## On "Crowdsourced Data"

You basically reinvented the concept, yeah. The progression is:
1. One person has data locally (localStorage)
2. People exchange data files manually (export/import JSON)
3. A central place stores everyone's data (database)
4. Everyone's browser talks to that central place (API)
5. Changes appear instantly for everyone (real-time sync)

Each step adds a little complexity but removes a manual step. You're at step 2 now, which is honestly fine for a group of friends playing every week or two. Step 3-4 is worth doing when the manual merging gets annoying.
