# Agent-IDOR-Scanner-
Developed an Agent configured with Claude code and linked the two to open  google chrome, me browser login in with intigriti  


Built on OpenClaw with Claude as the reasoning engine, driving an authenticated Chrome session. Runs on cron schedule, writes a dated Markdown report and never touches a programs assets. 


WHAT IT DOES 

The scanner runs in two phases against the researcher dashboard: 

Phase 1 - List diff: Reads the program list names, bounty ranges, researcher counts, in-scope asset counts. Keeps only paid programs, skipping VDP no reward listings. Diffs the current paid set against a saved snapshot to catch newly launched programs and bounty range changes. 

Phase 2 - Scope check: (Queued,Capped). For new or changed programs, it opens each programs scope page one at a time and extracts only facts stated on the page:

• Are REST/GrahphQL APIs or a mobile/app backed in scope 
• Is self-registration permitted, or are multiple test accounts provided?  (You need two accounts to test IDOR period.)
• Is IDOR/BOLA broken access control on the excluded list?
• Is automated tooling prohibited? 

A program is flagged IDOR-STARTER only if all three hold: API or authenticated surface in scope, multi-account access obstainable, and IDOR/BOLA not excluded. Results are sorted by reacher count ascending - fewest researchers first and least picked over program. 

OUTPUT: A dated report at ~/bounty/reports/intigriti-YYYY-MM-DD.md with flagged programs, the quoted scope line behind each judgment and a Baseline progress: X done / Y pending line. 

NOTE: The scan cannot tell you a program has easy bouts - nothing on scape page says that. It ranks programs by how accessible and uncontested they are to begin on, which is the closest legitimate proxy. Treat a flag as a "Reason place to start, not a guaranteed finding. 

-------------------------------------------------------------------------

DESIGN 
                    ┌─────────────────────────┐
   cron (weekly) ─▶ │   OpenClaw Gateway       │
                    │   isolated agent session │
                    └───────────┬──────────────┘
                                │ drives
                    ┌───────────▼──────────────┐
                    │  Authenticated Chrome     │  ← you log in by hand once
                    │  (openclaw browser profile)│
                    └───────────┬──────────────┘
                                │ reads (rendered text only)
                    ┌───────────▼──────────────┐
                    │  Researcher dashboard     │
                    └───────────────────────────┘

   State files (local):
     ~/bounty/intigriti-list.json    ← paid-program baseline
     ~/bounty/intigriti-queue.json   ← scope-check work queue (pending/done)
     ~/bounty/reports/*.md           ← dated output

--------------------------------------------------------------------------
Design decision: 

• Read only always: The agent never submits, follows, hides, comments or fills a form, or send request to any program. It reads the rendered page text. This is the single most important guardrail - the agent holds a live authenticated session while reading pages authored by other people. So keeping it read-only is what the limits blast radius if a page ever contains hostile text. 

• A Persistent queue per run: Scope pages are the expensive part. Rather than crawl everything on one run ( Which would hammer the server and risk IP flag) every paid programs is marked as pending and each run drains a capped batch. A page that fails to load stays pending - nothing is ever silently skipped. 

• Local Delivery: No Chat channel is configured, delivery is set to mode: none and output lands in dated files you read yourself. 

--------------------------------------------------------------------------

What was learned: 

1. The whole database cannot be scanned: The dashboard lists ~223 programs across ~10 paginated pages/ Reading every scope page in one unattended run would means dozen sequential loads - enough to strain the platforms servers and get an IP flagged as abusive. The capped-queue design exist specifically to avoid that. The right amount of automation here is polite and slow and not comprehensive and fast. 

2. Scope pages are secured behind program membership: The decisive finding, The program list is readable by any logged in researcher but scope detail page returns 404 forbiddon ("You actually don't have access to this page") until you've clicked into and joined a program. The automation can track the list for cheap forever but can only read the scope - and therefore only flag IDOR-STARTER - for programs you have already been accepted to.The platforms access model not the code is the real constraint. The next iteration points phase 2 only at the joined programs, where scope pages return 200. 

3. Cheaper model does not mean cheaper run: Moving the job from larger model to a smaller one increased token usage on the browser-heavy work( more entries and try attempt's to pages caused more page page retires). Model chose fro agent browsing isn't going to decades token usage - efficiency at this task matters more than sticker rate. Worth noting. 

4. Routing is separate from logic: Early runs were skipped with no-route - the scheduler fired fine but the run had nowhere to deliver. Fixing that meant understanding OpenClaws delivery layer (Heartbeat.target, delivery.mode), Which is entirely independent of what the automation actually does. 

--------------------------------------------------------------------------

The automation prompt:

Here's the full prompt driving the agent (Weekley run capped at 15 scope pages per run):

You read Intigriti program pages through the already-authenticated openclaw Chrome profile. You are READ-ONLY: never submit, follow, hide, comment, fill a form, or send any request to a program's assets. Read rendered page text only.

If the researcher programs page redirects to login or shows a logged-out state, STOP and reply exactly: SESSION EXPIRED — re-login needed. Do nothing else.

PHASE 1 — LIST DIFF (every run, cheap):
Open the researcher programs page in list view. Page through the list. For each program extract ONLY what the list shows: name, bounty range, researcher count, in-scope asset count. Count a program as PAID only if it shows a numeric bounty range (e.g. $500–$30,000); skip no-reward / VDP-only.
Compare the PAID set to ~/bounty/intigriti-list.json.
- If that file does not exist: write it from what you read, then seed ~/bounty/intigriti-queue.json with every paid program marked "pending".
- Otherwise identify newly listed paid programs and paid programs whose bounty range changed; append each (deduped) to the queue as "pending". Leave existing queue entries alone.

PHASE 2 — SCOPE CHECK (from the queue, HARD CAP 15 per run):
Take up to 15 "pending" entries from ~/bounty/intigriti-queue.json, ordered new/changed first, then lowest researcher count first. Open each scope page one at a time. For each, extract ONLY facts stated on the page:
- Are REST/GraphQL APIs or a mobile/app backend in scope?
- Is self-registration permitted, OR are multiple authenticated test accounts provided?
- Does IDOR / BOLA / broken access control appear in the out-of-scope / excluded list?
- Is automated tooling prohibited?
Quote the line each judgment rests on. If the page doesn't state something, write UNKNOWN — never infer.
Mark an entry "done" after you read its page. If a page fails to load, leave it "pending" — never silently skip it.
Flag a program IDOR-STARTER only if ALL are true from the page text: API or authenticated surface in scope, AND self-registration or multiple test accounts obtainable, AND IDOR/BOLA not excluded. Show its researcher count next to the flag.

REPORT — write to ~/bounty/reports/intigriti-YYYY-MM-DD.md (run date):
List newly listed paid programs, bounty range changes, and IDOR-STARTER flags (with quoted lines + researcher counts), sorted by researcher count ascending. Include a "Baseline progress: X done / Y pending" line. Any pending entries not reached this run go under "Queued for next run."
Then update ~/bounty/intigriti-list.json to match Phase 1. Do NOT touch ~/bounty/intigriti-snapshot.json.
Only if Phase 1 found no new paid programs, no bounty changes, AND the queue has zero pending entries: write a one-line NO CHANGES report and still update the list snapshot.

Do NOT test, probe, or request any asset. Do NOT rank by likelihood of finding bugs or estimate difficulty. Do NOT fabricate a diff — if a page won't load or data is missing, say so.

--------------------------------------------------------------------------

Setup

Prerequisites: OpenClaw gateway running continuously, Claude API  key with credit, and chrome. 

1. Authenticate once, by hand. Open the platform in the OpenClaw-controlled browser and log in, so the profile holds a live session:

   openclaw browser open <researcher-dashboard-url>

2.Create the automation with a weekly cron and isolated session: 

   openclaw automations create "0 9 * * 1" \
     "<paste the prompt above>" \
     --name "Intigriti IDOR Scanner" \
     --session isolated

0 9 * * 1 = every Monday, 9am. 3.Confirm delivery routing  With no chat channel, set delivery to none so runs write files instead of erroring on missing channel 5. Trigger a Manuel run to play down the baseline and check the token cost before trusting the schedule: 

   openclaw automations list          # get the job id
   openclaw automations run <jobId>

5. Read the output each Monday at ~/bounty/reports/.
--------------------------------------------------------------------------

Ethics & safety
Reads my own authorized account only, one page at a time, on a slow schedule. No asset is ever contacted.
Rate-limit aware by design — the per-run cap exists to keep load light and avoid IP flagging.
Check the platform's automation terms. Reading your own account slowly is about as polite as automated reading gets, but it is still automation of someone else's platform — know where their line is before you run it on a schedule.
Prompt-injection posture: the agent is strictly read-only with no write/submit tools, which is the mitigation for reading attacker-controllable page content while holding a session.

--------------------------------------------------------------------------

Tech
OpenClaw (Gateway scheduler + browser automation) · Claude Code (reasoning engine) · authenticated Chrome profile · cron · local JSON state.




Built by @ZenBlackhood. A learning project in agentic automation and bug bounty tooling.





