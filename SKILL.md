\---

name: openclaw\_watchtracker

description: >

&#x20; OpenClaw is a personal TV show watch tracker that manages episode progress across

&#x20; multiple concurrent series using air-date interleaving. Use this skill whenever

&#x20; the user wants to track what they're watching, find out what episode to watch next,

&#x20; mark episodes as watched, check their progress on a show, add a new show to their

&#x20; watchlist, or asks anything like "where was I in \[show]", "what should I watch next",

&#x20; "track \[show] for me", "I want to rewatch \[show]", or "am I caught up on \[show]".

&#x20; This skill handles ALL watch-tracking tasks including concurrent series (e.g., watching

&#x20; Stargate SG-1 and Atlantis simultaneously in air-date order). Always use this skill

&#x20; for anything TV/show-tracking related — do not attempt to handle these tasks without

&#x20; consulting this skill first.

metadata:

&#x20; openclaw:

&#x20;   requires:

&#x20;     bins:

&#x20;       - python3

\---



\# OpenClaw — TV Watch Tracker



OpenClaw tracks episode progress across one or more shows simultaneously,

interleaving them by original air date so the user watches everything in the

correct chronological order.



\---



\## Storage Layout



All data lives in `\~/.openclaw/` on the user's local machine.



```

\~/.openclaw/

├── watchlist.json          # Index of all tracked shows

└── shows/

&#x20;   ├── stargate\_sg1.json   # One file per show

&#x20;   ├── stargate\_atlantis.json

&#x20;   └── ...

```



\### watchlist.json schema

```json

{

&#x20; "shows": \[

&#x20;   {

&#x20;     "id": "stargate\_sg1",

&#x20;     "title": "Stargate SG-1",

&#x20;     "file": "shows/stargate\_sg1.json",

&#x20;     "status": "active"

&#x20;   }

&#x20; ]

}

```



`status` can be: `active` | `completed` | `paused`



\### Show file schema (e.g., stargate\_sg1.json)

```json

{

&#x20; "id": "stargate\_sg1",

&#x20; "title": "Stargate SG-1",

&#x20; "added": "2026-03-26",

&#x20; "status": "active",

&#x20; "crossovers": \["stargate\_atlantis"],

&#x20; "episodes": \[

&#x20;   {

&#x20;     "season": 1,

&#x20;     "episode": 1,

&#x20;     "title": "Children of the Gods",

&#x20;     "air\_date": "1997-07-27",

&#x20;     "watched": false,

&#x20;     "skipped": false

&#x20;   }

&#x20; ]

}

```



Episodes may optionally include `"type": "special"` or `"type": "movie"` for

non-standard entries.



\---



\## Workflow: Adding a New Show



1\. \*\*Check if already tracked\*\* — read `\~/.openclaw/watchlist.json` and look for a

&#x20;  matching title or ID. If found, tell the user and ask if they want to reset progress.

2\. \*\*Initialize directory if needed\*\* — if `\~/.openclaw/` does not exist, run:

&#x20;  ```bash

&#x20;  mkdir -p \~/.openclaw/shows

&#x20;  echo '{"shows":\[]}' > \~/.openclaw/watchlist.json

&#x20;  ```

3\. \*\*Fetch episode data\*\* — use web search to find the complete episode list.

&#x20;  - Search query: `"\[Show Title]" complete episode list all seasons air dates`

&#x20;  - Preferred sources: Wikipedia episode tables, TVMaze, TVDB

&#x20;  - Collect for every episode: season number, episode number, episode title,

&#x20;    original air date

&#x20;  - Include specials and TV movies if they have air dates

4\. \*\*Check for crossovers\*\* — search for known crossover episodes or companion shows.

&#x20;  - Examples: SG-1 → suggest Stargate Atlantis and Stargate Universe

&#x20;  - Superhero shows → suggest other shows in the same universe

&#x20;  - After adding the show, tell the user: "I found that \[Show] has crossovers with

&#x20;    \[Other Show]. Would you like to track that too?"

5\. \*\*Generate the show ID\*\*

&#x20;  - Lowercase the full title

&#x20;  - Replace spaces and non-alphanumeric characters with underscores

&#x20;  - Collapse multiple underscores into one

&#x20;  - Examples: `"Stargate SG-1"` → `stargate\_sg1`, `"Star Trek: TNG"` → `star\_trek\_tng`

6\. \*\*Create the show file\*\* at `\~/.openclaw/shows/\[show\_id].json`

7\. \*\*Update watchlist.json\*\* — append the new show entry using Python (see File

&#x20;  Operations section below)

8\. \*\*Confirm to user\*\* — report the show name, total episode count, season count,

&#x20;  and any crossover suggestions



\---



\## Workflow: What's Next?



This is the core interleaving feature. Use it when the user asks "what do I watch

next", "what's up next", "what should I watch", or similar.



1\. Load `\~/.openclaw/watchlist.json`

2\. For each show where `status == "active"`, read its show file

3\. Find the earliest episode where `watched == false` AND `skipped == false`

4\. Collect those "next candidates" from all active shows

5\. Sort the candidates by `air\_date` ascending

6\. Present the sorted list to the user



\*\*Output format:\*\*

```

📺 Up Next (air-date order):



1\. Stargate SG-1 — S8E01 "New Order, Part 1" (2004-07-09)

2\. Stargate Atlantis — S1E01 "Rising, Part 1" (2004-07-16)

3\. Stargate Atlantis — S1E02 "Rising, Part 2" (2004-07-23)

4\. Stargate SG-1 — S8E02 "New Order, Part 2" (2004-07-23)



Tip: Episodes 3 and 4 share the same air date — you can watch either first.

```



\*\*Tie-breaking:\*\* When two episodes share the same air date, note this to the user.

Default ordering: the show that was added to the watchlist first goes first.



\*\*How many to show:\*\* Default to showing the next 5 entries. If the user asks for

more or fewer, adjust accordingly.



\---



\## Workflow: Mark as Watched



Triggered by phrases like "I just watched SG-1 S8E01", "mark atlantis s1e1 watched",

"I finished the pilot of \[show]", "I watched up through S3E5".



1\. Identify the show using fuzzy title matching (e.g., "atlantis" matches

&#x20;  "Stargate Atlantis")

2\. Locate the episode by season/episode number or title keyword

3\. Set `"watched": true` on the matched episode

4\. Save the file using the Python JSON pattern (see File Operations)

5\. Confirm to the user and show what's up next for that show



\*\*Bulk marking:\*\* If the user says "I already watched all of season 1" or "mark

everything through S2E10 as watched", iterate and mark all matching episodes.



\*\*Range marking:\*\* "I watched S1E1 through S1E6" — mark all episodes in that

inclusive range.



\---



\## Workflow: Progress Summary



Triggered by "how far am I on \[show]", "show my progress", "what have I watched".



For a single named show:

```

📊 Stargate SG-1 Progress

Seasons: 10 | Total Episodes: 214

Watched: 87 (41%) | Remaining: 127

Currently on: S5E01 — "Enemies"

```



For all shows (no specific show named):

Show the per-show block for every tracked show (active, paused, completed), then

a combined summary line:

```

🎬 Combined Queue: 243 unwatched episodes across 2 active shows

```



\---



\## Workflow: Jump to Episode



Triggered by "start me at season 3", "skip to the Atlantis pilot", "begin from S4E1".



1\. Identify the target season/episode

2\. If jumping \*\*forward\*\* from current position: ask the user "Should I mark all

&#x20;  prior episodes as watched?" before doing so

3\. If jumping \*\*backward\*\*: do not alter any watched flags; just confirm the new

&#x20;  reference point

4\. Confirm the jump to the user



\---



\## Workflow: Skip an Episode



Triggered by "skip this one", "I don't want to watch \[episode]", "skip S3E5".



Set `"skipped": true` on the identified episode. Skipped episodes are excluded from

the "What's Next" queue but remain in the file for record-keeping.



\---



\## Workflow: Show Watchlist



Triggered by "show my watchlist", "what am I tracking", "list my shows".



Read `\~/.openclaw/watchlist.json` and display all shows with their status and a

brief progress note pulled from their show file.



Example output:

```

📋 OpenClaw Watchlist



▶ Stargate SG-1 (active) — S5E01, 87/214 watched

▶ Stargate Atlantis (active) — S1E01, 0/200 watched

⏸ Farscape (paused) — S2E14, 33/88 watched

✅ Firefly (completed) — 14/14 watched

```



\---



\## Workflow: Pause / Resume a Show



"Pause \[show]" → set `status: "paused"` in both watchlist.json and the show file.

"Resume \[show]" → set `status: "active"`.



Paused shows are excluded from the "What's Next" interleaved queue but remain in

the watchlist.



\---



\## File Operations



Use these patterns for all JSON reads and writes. Always use Python for writes to

avoid corrupting the JSON.



\### Read watchlist

```bash

cat \~/.openclaw/watchlist.json

```



\### Read a show file

```bash

cat \~/.openclaw/shows/\[show\_id].json

```



\### Add a new show to watchlist.json

```bash

python3 << 'EOF'

import json, os

path = os.path.expanduser("\~/.openclaw/watchlist.json")

with open(path) as f:

&#x20;   wl = json.load(f)

wl\["shows"].append({

&#x20;   "id": "SHOW\_ID",

&#x20;   "title": "SHOW TITLE",

&#x20;   "file": "shows/SHOW\_ID.json",

&#x20;   "status": "active"

})

with open(path, "w") as f:

&#x20;   json.dump(wl, f, indent=2)

EOF

```



\### Write a new show file

```bash

python3 << 'EOF'

import json, os

data = { ... }  # full show object

path = os.path.expanduser("\~/.openclaw/shows/SHOW\_ID.json")

with open(path, "w") as f:

&#x20;   json.dump(data, f, indent=2)

EOF

```



\### Mark an episode watched

```bash

python3 << 'EOF'

import json, os

path = os.path.expanduser("\~/.openclaw/shows/SHOW\_ID.json")

with open(path) as f:

&#x20;   data = json.load(f)

for ep in data\["episodes"]:

&#x20;   if ep\["season"] == SEASON and ep\["episode"] == EPISODE:

&#x20;       ep\["watched"] = True

&#x20;       break

with open(path, "w") as f:

&#x20;   json.dump(data, f, indent=2)

EOF

```



\---



\## Tips \& Edge Cases



\- \*\*Air date unknown:\*\* If an episode has no air date (common for specials or OVAs),

&#x20; append it to the end of its season with `"air\_date": null`. It will sort to the

&#x20; bottom of the queue.

\- \*\*Specials/Movies:\*\* Include them if they have air dates and are relevant to the

&#x20; viewing order. Add `"type": "special"` or `"type": "movie"` to the episode object.

\- \*\*Show not found by web search:\*\* Ask the user for the season and episode counts

&#x20; so you can build a skeleton file. They can fill in air dates later.

\- \*\*Directory missing:\*\* Always check for `\~/.openclaw/` before any operation and

&#x20; create it if absent.

\- \*\*Corrupt JSON:\*\* If a file fails to parse, report the exact error and offer to

&#x20; rebuild the file from scratch using a fresh web search.

\- \*\*Fuzzy show matching:\*\* When the user refers to a show by a short name (e.g.,

&#x20; "atlantis", "sg1", "trek"), match it to the closest tracked show title.



\---



\## Command Summary (Quick Reference)



| User Says | Action |

|---|---|

| "Add \[show] to OpenClaw" | Fetch episodes, create show file, check crossovers |

| "What's next?" / "What do I watch?" | Interleaved next-up list (default 5 entries) |

| "Mark \[show] S1E1 watched" | Update watched flag, confirm + show next |

| "How far am I on \[show]?" | Progress summary for that show |

| "Show my progress" | Progress summary for all shows |

| "Skip \[episode]" | Set skipped flag |

| "Start \[show] from S3" | Jump to season/episode (ask before bulk-marking) |

| "Show my watchlist" | List all tracked shows with status and position |

| "Pause \[show]" | Set show status to paused |

| "Resume \[show]" | Set show status back to active |

