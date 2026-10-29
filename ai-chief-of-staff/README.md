# AI Chief of Staff for a CEO

<img width="3002" height="1630" alt="luca-dashboard" src="https://github.com/user-attachments/assets/38f6abc6-dff8-466c-9702-dcb0db51f842" />

*The production dashboard. Account details are redacted, and this view shows system status only, with no business data.*

---

**LUCA (Live Unified Command Agent)** is an AI operating system I designed, built and run for the CEO of a renewable-energy company. Every day it pulls live data from five business tools (finance, sales, operations, calendar and team chat) and gives the CEO a spoken morning brief before they've spoken to anyone. Through the day the CEO can ask it anything by voice or chat: *"what's the bank balance?"*, *"how's ops looking?"*, *"does anything in the team chat affect ops today?"*

**✅ This is not a demo. LUCA is in daily production use**. The code, prompts and business data are confidential, so this is an anonymised case study: what it does, how it's built, and every design decision behind it. 

---

## 📊 At a Glance

| | |
|---|---|
| **Status** | Live in production |
| **Build** | 57 commits, 13 delivery phases |
| **Users** | CEO (primary) and the builder (admin) |
| **Channels** | Voice (dashboard mic + "Hey LUCA" wake word), Google Chat, live web dashboard |
| **Data sources** | Xero, HubSpot, monday.com, Google Calendar, Google Chat |
| **AI** | Anthropic Claude (Sonnet for analysis, Haiku for routing and speech), ElevenLabs (speech-to-text and text-to-speech) |
| **Infrastructure** | Python + FastAPI on Google Cloud Run, Cloud Storage, Secret Manager, Cloud Scheduler, Cloudflare Pages + Access |

---

## 🧠 Overview

### The problem
A CEO's business information was spread across five systems. Cash and payables were in the accounting system, deals and leads in the CRM, jobs and maintenance work in the project boards, plus a calendar and a busy team chat. Getting a picture of the business meant opening each one, every morning, and asking people for updates.

### The solution
LUCA does that morning round for them. It:

- **Delivers a spoken morning brief**, one continuous narrative covering finance and sales, operations and maintenance, the sales pipeline, what the team is talking about, and the CEO's day
- **Answers questions by voice or chat** and routes each one to the right specialist agent, or several at once for cross-cutting questions
- **Remembers the conversation**, so a follow-up question in chat can continue one started by voice
- **Shows everything on a live dashboard** that syncs with the audio: as the brief moves from finance to pipeline, the matching panel opens
- **Writes a log of every brief** to an Obsidian knowledge vault, for a searchable history
- **Monitors itself** every 15 minutes and flags problems on the dashboard before the CEO notices them

---

## 🏗️ Architecture

```
                         ┌───────────────────────────────┐
  CEO speaks / types ──▶ │  Dashboard (Cloudflare Pages)  │ ◀── email one-time-code login
                         │  or Google Chat space          │     (Cloudflare Access allow-list)
                         └──────────────┬────────────────┘
                                        │  server-side proxy adds the secret token
                                        ▼
                         ┌───────────────────────────────┐
                         │  FastAPI webhook (Cloud Run)   │
                         │  1. Speech-to-text (ElevenLabs)│
                         │  2. Intent classifier (Haiku)  │
                         └──────────────┬────────────────┘
                 ┌──────────────────────┼───────────────────────────┐
                 ▼                      ▼                           ▼
        single question         cross-domain question        "give me the brief"
        → 1 agent               → agents in parallel         → 5 brief sections
                                → Haiku cross-references        in parallel
                 │                      │                           │
                 └──────────┬───────────┴───────────────┬───────────┘
                            ▼                           ▼
              Specialist agents: direct API     Python does all the maths;
              calls → Python bucketing →        Claude only writes the words
              one Claude (Sonnet) call
                            │
                            ▼
              Speech rewrite (Haiku) → Text-to-speech (ElevenLabs) → MP3 reply
                            │
                            ▼
         Cloud Storage: brief state · cached audio · conversation memory · health status
         Google Drive:  brief log → Obsidian vault
```

---

## 🤖 The Agents

Five specialist agents, each with one data source:

| Agent | Domain | Source | Connection |
|---|---|---|---|
| **Millie** | Finance | Xero | OAuth 2.0, tokens rotated automatically and stored in Secret Manager |
| **Magic** | Sales | HubSpot | REST API |
| **Archie** | Operations | monday.com | GraphQL API |
| **Obi Wan** | Calendar | Google Calendar | Service account with domain-wide delegation |
| **Max** | Team comms | Google Chat | Service account with domain-wide delegation |

**Every agent works the same way:** call the tool's API directly, then sort and count the results in plain Python, then make **one** Claude call to turn the prepared facts into language.

---

## 📰 The Morning Brief

The full brief runs five sections **in parallel**, then joins them into one flowing spoken narrative with no "Section A, Section B" announcements. Its structure is modelled on a video the CEO recorded of how they'd want the brief to sound.

| Section | Covers | Sources blended |
|---|---|---|
| **A: Financial & Sales** | Deals signed this month, this week's bills due, cash position | CRM + accounting + a manually updated cash board |
| **B: Operations & Maintenance** | Open, active and critical maintenance jobs; support tickets opened vs closed this month | Project boards + CRM tickets |
| **C: Pipeline** | Deals waiting on proposals, new leads this month, deals missing key information | CRM |
| **D: Comms** | What the team has been raising in the main team chat | Google Chat |
| **E: Your Day** | Today's meetings and urgent tasks | Calendar + task board |

Each section reuses the matching agent's data fetching rather than duplicating it, and adds its own synthesis prompt.

---

## 🧭 Intent Routing

Every message goes through a fast, cheap classifier (Claude Haiku) that returns one of these:

| Classifier says | What happens |
|---|---|
| `brief` | All 5 sections in parallel → one continuous narrative |
| one agent (e.g. `millie`) | That agent answers directly |
| several agents (e.g. `max,archie`) | Those agents run in parallel, then a second Haiku call **cross-references** their answers into one reply rather than stacking them |
| `chat` | Small talk ("hey LUCA", "thanks") gets a natural reply, not a business report |
| `none` | A help message, only for genuinely unrecognisable input |

---

## 🧩 Design Decisions

Every significant decision, why I made it, and what it cost.

### AI & data accuracy

**1. Direct API calls instead of letting the AI browse tools.**
The original plan had the AI call each tool itself through connectors. I switched to my own code fetching the data, Python sorting and counting it, and then a single Claude call to write it up.
*Why:* the model only ever sees a small, pre-digested set of facts. That keeps it fast and cheap, and it can't drift onto the wrong records.
*Trade-off:* every new data point needs code, not just a prompt change.

**2. Python does all the maths; Claude only narrates.**
Projected balances, created-minus-closed counts and totals are all calculated in code and handed to the model as finished numbers.
*Why:* language models make arithmetic mistakes, and a CEO acting on a wrong cash figure is the worst failure this system could have.

**3. "Pure data" mode for finance.**
The finance agent reports exactly what the accounting system shows (balance, overdue receivables, overdue payables) with no forecasts, shortfall maths or advice.
*Why:* there was no in-house accountant to check calculated figures, so LUCA reports facts, not opinions.
Related rules:
- top 5 overdue items by amount by default, with the full list on request
- "overdue" means strictly past due
- overdraft is always shown separately and never netted against the operating balance

**4. Prompts stay static; live data only goes in the message.**
System prompts never contain dates, times or IDs.
*Why:* static prompts can be cached by the AI provider, which cuts cost and speeds up responses. Any changing detail in the prompt breaks the cache.

**5. "Never invent data."**
If a source returns nothing or errors, LUCA says so and moves on. It never fills the gap.

**6. Every brief number is traceable.**
Each brief section returns its raw, pre-AI data alongside the narrative, so when a number looks wrong I can tell in seconds whether the source data or the AI is at fault.

### The brief

**7. One continuous narrative, not five reports.**
The brief reads like a chief of staff talking, matching the CEO's own recorded example, rather than five separately announced blocks.

**8. Sections run in parallel.**
Five independent Claude calls run at the same time instead of one after another, which cuts brief time to roughly the slowest single section.

**9. "Your Day" is protected from the length limit.**
Text-to-speech has a character limit. Sections are joined in a fixed order and "Your Day" comes last, so a crude cut would always lose the CEO's afternoon meetings. LUCA now drops whole earlier sections first (finance, then pipeline, then operations, then comms) and never trims "Your Day". As a last resort it ends on a full sentence, never mid-word.

**10. Don't read job codes aloud.**
CRM deal names are often internal job or quote codes. Hearing a string of letters and digits read aloud is useless, so the prompts describe each deal in three words or fewer from its description, or refer to it generically.

**11. Spoken numbers stay exact.**
Every dollar figure is spoken in full and never rounded, and freshness notes ("as of 7:40am") stay attached to the fact they qualify. The one deliberate exception is the new-leads pipeline total: it's an estimate, so it's rounded to a natural spoken figure ("about X million").

**12. Two speaking modes.**
A live voice reply continues the conversation naturally and never repeats a greeting the CEO already gave. The pre-generated morning audio opens with its own natural lead-in.

**13. Field-gap alerts name the exact missing field.**
Instead of "some deals have missing data", LUCA says which deal is missing which field (amount, gross profit, system size, lead source). It only checks this month's leads, not years of old records, so the alert stays actionable.

### Voice

**14. Wake word runs in the browser, for detection only.**
"Hey LUCA" uses the browser's built-in speech recognition purely as a trigger. The actual command is a second phrase after a short pause, the same pattern as Siri and Alexa.
*Trade-off:* you can't say it all in one breath, because that would need a constant rolling audio buffer.

**15. One microphone stream, not two.**
Auto-stop on silence uses an audio level detector on the *same* microphone stream as the recording. Running two speech recognisers on one mic was a real bug that silently corrupted recordings.

**16. LUCA can't wake itself up.**
The wake-word listener is switched off whenever LUCA is speaking. Before this fix, LUCA's own voice through the speakers could trigger "Hey LUCA" and cut the brief off mid-sentence.

**17. Honest about iOS limits.**
Continuous wake-word listening doesn't work in iPhone web browsers, which is a platform limitation I confirmed on a real device. LUCA detects iOS and shows "tap to talk" instead of promising a feature that can't work. A second iOS issue (Safari blocking audio playback after the multi-second processing gap) is fixed by "unlocking" the audio player with a silent clip during the user's tap.

**18. Shared memory across voice and chat.**
LUCA keeps the last 5 exchanges per person in one store, so a conversation can move between voice and chat.

**19. One warm server.**
The backend keeps one instance always running, so voice replies don't wait for a cold start.

### Dashboard

**20. Audio and visuals stay in sync.**
When the brief audio is generated, LUCA records when each section starts. The dashboard uses that timing map both ways:
- as the audio plays, the matching panel opens
- clicking a section chip jumps the audio to that section

**21. Playback speed.**
The brief plays at 1×, 1.25×, 1.5× or 2×. Busy people listen fast.

**22. No secrets in the browser, ever.**
The dashboard's server-side functions sign every storage request and attach the API token themselves. The browser never sees a key, service-account credential or token.

**23. Login without passwords.**
Cloudflare Access sits in front of the whole site, using an email one-time code and a two-person allow-list. There are no passwords to manage or leak.

**24. Responsive design verified the hard way.**
An early check across 13 screen sizes looked like a pass, but missed two real overflow bugs that only showed up on an actual laptop with browser zoom. I changed my method to reproduce each bug with a screenshot first, fix it, and confirm with a screenshot. Then I re-verified across realistic laptop settings and an automated 320–2560px sweep.

### Cost

**25. Refresh on demand, not on a timer.**
LUCA used to regenerate the brief every 30 minutes, 24/7. During a 9-day stretch with zero usage, it burned through a real API credit top-up producing briefs nobody read. I confirmed this from the logs before changing anything, including ruling out a leaked key.
Now the dashboard checks the age of the cached brief when someone actually opens it, and only refreshes if it's more than 20 minutes old. If the refresh fails, it serves the last good brief instead of an error.
*Result:* an idle week costs nothing.
*Trade-off:* the first load after a long idle stretch takes about 8–10 seconds.
The old timer is **paused, not deleted**, so the change can be reverted.

**26. Right model for each job.**
Sonnet handles the analysis and synthesis. Haiku handles classification, cross-referencing, speech rewriting and the health ping, which are fast, cheap and high-volume.

### Reliability

**27. A health check that runs on a timer, even though the brief no longer does.**
Every 15 minutes LUCA makes a real AI call and checks that its state is fresh and that the sections aren't all failing at once (the signature of a system-wide outage). The result goes into a health file, and the dashboard shows a red banner with the exact problem.
*Why it stays on a timer:* its whole job is catching outages during quiet periods, which an on-demand check would miss. It costs one tiny AI call per run.
*Born from an incident:* a billing outage once made every brief section fail at the same time, and nobody knew until someone opened the dashboard.

**28. A missing health file means "unknown", not "unhealthy".**
So the system doesn't raise a false alarm on its first run.

**29. Every deploy checks itself.**
After every deployment, an automated safety check runs against the real live systems, not mocks:
- the code loads
- the classifier still routes sample questions correctly
- a full live voice round trip works
- a headless browser loads the dashboard with zero JavaScript errors

If anything fails, the deploy reports failure.

**30. Tests clean up after themselves without touching real data.**
The live voice test saves a copy of the CEO's conversation memory first. Afterwards it removes only its own test exchange and keeps anything real said in the meantime. The write only succeeds if nothing changed mid-cleanup. (An earlier version deleted the whole memory file, which I caught and fixed.)

**31. Safe deploy settings.**
Deployments only *update* the server's environment settings and never replace them wholesale, because one wrong flag would wipe every credential on the live server.

### Security

**32. Hard limits on where LUCA can post.**
These were added after a local test script posted into a team chat space under the CEO's identity, a real incident. There are three checks, enforced inside the posting code itself:
1. the main team space is on a **permanent denylist** with no override
2. LUCA can only post to a space that is handling a real incoming message in that same execution, so a hard-coded test script structurally cannot post anywhere
3. posting as a real company identity outside a real event is blocked unless an explicit local-only flag is set, and that flag never exists on the server

Manual testing goes through a dry-run function that prints instead of posting.

**33. Token refresh without wasted calls.**
The finance agent decodes the access token's expiry time locally instead of making a live test call every time. That saves a full network round trip on every request, and about 2.7 seconds whenever the token had expired.

### Portability

**34. Runs on Windows and macOS.**
I migrated LUCA from Windows to macOS and made it portable:
- file paths built with path libraries
- cloud tools found on the system path instead of hard-coded locations
- developer tools installed in the home folder, not the cloud-synced project folder, whose sync client locks files mid-install

---

## 🔥 Production Incidents & What I Learned

| Incident | Root cause | Fix | Lesson |
|---|---|---|---|
| Finance section stopped working | The accounting app shared one client secret with an unrelated automation. Rotating it for one silently broke the other. | Full re-authorisation flow written straight to Secret Manager. Documented that a separate app per integration is the long-term fix. | One shared credential is one shared point of failure. |
| API credit burned with nobody using LUCA | A 30-minute refresh timer ran 24/7 | On-demand refresh (#25) | Cost should follow usage, not the clock. |
| Every brief section failed at once | AI provider billing outage | Scheduled health check + dashboard banner (#27) | Deploy-time tests don't catch failures that start hours later. |
| Brief cut off mid-sentence | LUCA's own voice triggered its wake word | Listener paused during playback (#16) | Test the voice loop with real speakers, not just headphones. |
| Afternoon meetings missing from the brief | Blunt character-limit cut always hit the last section | Section-aware budgeting (#9) | Know which content is most important before you cut. |
| Voice replies failed end-to-end | Speech-to-text API started requiring a model field | Added the field and documented it | Pin and document third-party API quirks. |
| Test message posted as the CEO | Local script could post anywhere | Three-layer posting safeguard (#32) | Make the dangerous action structurally impossible, not just discouraged. |
| A paused job looked like a bug | I wrongly re-enabled the deliberately paused timer | Reverted it and documented why it's paused | Check the history for *why* before "fixing" something that's off. |

### API quirks found along the way
- The accounting API **silently ignores** its invoice-type filter, so all filtering happens in code, and pagination uses the raw page count so it doesn't stop early.
- The accounting system's live bank-feed access is restricted to certified partners, so LUCA uses the reconciled balance-sheet figure and labels it accurately.
- The accounting system rotates its refresh token on every use, so both tokens are saved together every time.
- The CRM's ticket "created" field is empty on every ticket, so LUCA uses a different date field.
- The chat API's service-account access doesn't return sender display names, so senders are matched by their ID.

---

## 🛠️ Tech Stack

| Tool | Role |
|---|---|
| Python, FastAPI | Backend webhook, agents, orchestration |
| Anthropic Claude (Sonnet + Haiku) | Synthesis, intent classification, cross-referencing, speech rewriting |
| ElevenLabs | Speech-to-text and text-to-speech |
| Google Cloud Run | Hosts the backend (Sydney region) |
| Google Cloud Storage | Brief state, cached audio, conversation memory, health status |
| Google Secret Manager | Credentials and auto-rotating OAuth tokens |
| Google Cloud Scheduler | 15-minute health check |
| Cloudflare Pages + Pages Functions | Dashboard hosting and the secure server-side proxy |
| Cloudflare Access | Passwordless email one-time-code login |
| Web Speech API + Web Audio API | Browser wake word and silence detection |
| Puppeteer | Headless-browser dashboard test |
| Xero, HubSpot, monday.com, Google Calendar, Google Chat | Data sources |
| Google Drive + Obsidian | Brief log and knowledge vault |
| Claude Code | AI pair-programming during the build |

---

## 🗺️ Build Timeline

| Phase | Delivered |
|---|---|
| 1 | Project shell, knowledge-vault connection, development safety hooks |
| 2 | Five direct-API agents + cloud webhook |
| 3 | Morning brief assembly + saved state |
| 4 | Voice in and out |
| 5 | Brief logs written to the knowledge vault |
| 6 | Google Chat posting |
| 7 | Cross-domain questions |
| 8 | Live dashboard |
| 9 | Brief V2: single-narrative brief modelled on the CEO's own example |
| 10 | Pre-launch audit (security, speed, caching, accuracy, memory) + posting safeguard |
| 11 | Dashboard voice, wake word, audio-synced panels, small-talk handling |
| 12 | Post-launch fixes + automated safety check on every deploy |
| 13 | Finance outage recovery, voice and playback improvements, deploy reliability |
| Ongoing | On-demand refresh, scheduled health monitoring, Windows → macOS migration, safer live tests |

---

## 🔒 Confidentiality Note

LUCA is proprietary to my current employer. **No source code, prompts, credentials, internal IDs, URLs, names of people, client details or business figures are included here.** The screenshot is the production interface with account details redacted, showing system status only. The repository licence covers this write-up, not the LUCA system itself.


