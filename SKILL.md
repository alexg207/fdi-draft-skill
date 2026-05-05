---
name: Generate FDI Draft
description: Runs the entire Founder-Driven Intelligence motion end to end in Claude Code. Reads founder docs (memo, deck, transcripts) from a working directory, captures ICP and founder voice, runs an Exa Webset to discover and enrich ~30 matching companies, customizes the HTML dashboard template, and produces a complete per-founder Git repo ready to push or deploy. One activation, one finished build.
---

# Generate FDI Draft Skill

## Purpose
Runs the full FDI motion in a single skill: research intake → Webset enrichment → dashboard build. Reads raw founder docs from the filesystem, runs an Exa Webset against the founder's ICP, populates a dashboard following V1 Valar craft patterns, and outputs a complete Git repo ready to push or deploy.

The output is a 90% solution. You do the last 10% — handpicking the 5 companies to lead with, fine-tuning narrative, polishing visuals.

**Production environment: Claude Code.** The Websets MCP server requires Bearer-header auth, which claude.ai's web app can't provide. Claude Code is the only working environment until Exa ships OAuth. This is a feature, not a workaround — Claude Code's filesystem access produces materially better output than claude.ai's project-knowledge model. Raw docs stay on disk, version history accumulates per build, outputs land directly in a deployable Git repo.

## When To Use
- Starting a new FDI project (founder = company Primary is investing in or considering)
- Iterating on an existing FDI dashboard with new data or feedback
- Anytime a founder needs an interactive intelligence dashboard for outbound

## Prerequisites
- **Claude Code installed and configured.** This skill runs in Claude Code, not claude.ai web.
- **MCP connectors loaded in Claude Code:**
  - **Exa Websets MCP** — required. Different from the basic Exa search MCP. Personal API key configured at `dashboard.exa.ai/api-keys`. Make sure the API key is on the team that has credits.
  - **Lovelace MCP** — required for contact discovery. Provides `search_linkedin_profiles`.
  - **Granola MCP** — useful if founder transcripts come from Granola; the skill can query directly instead of requiring exports.
  - **Basic Exa MCP** — useful as a fallback for `web_search_exa` and `web_fetch_exa` calls during company-list curation.
- **Working directory structure** — see "Working Directory Layout" below. The skill expects raw founder docs in `inputs/`.
- **Git installed** — the skill clones the template repo and initializes the output repo.
- **`gh` CLI installed and authenticated** (recommended) — `brew install gh && gh auth login`. With `gh` set up, the skill creates the GitHub repo for each founder automatically. Without it, the skill creates the local repo only and you create the GitHub remote by hand later.
- **Internet access** — for `git clone` of the template + Webset API calls.

## Working Directory Layout

The skill operates inside a per-founder Git repo at `~/fdi/<founder-slug>/`. Layout:

```
~/fdi/<founder-slug>/                    # The build's Git repo (initialized Phase 0)
├── inputs/                              # Raw founder docs — read-only reference
│   ├── memo.pdf                         #   Investment memo (PVP / partner memo)
│   ├── deck.pdf                         #   Founder deck
│   ├── granola-2026-04-24.txt           #   Founder Granola transcripts
│   ├── prospect-call-2026-04-28.txt     #   Customer/prospect calls (high signal)
│   ├── deep-research.md                 #   Prior research / Claude outputs
│   └── pipeline.csv                     #   Pipeline spreadsheet (high signal)
├── template/                            # Cloned at runtime from fdi-template repo
│   ├── AI_INSTRUCTIONS.md               #   Authoritative operating manual
│   ├── TEMPLATE_GUIDE.md                #   V1 craft patterns (reference all build long)
│   ├── data.js                          #   Schema-commented placeholder
│   ├── index.html                       #   Placeholder dashboard
│   ├── CONTEXT_TEMPLATE.md              #   Fillable shell for Phase 4
│   └── BUILD_NOTES_TEMPLATE.md          #   Fillable shell for Phase 9
├── webset-spec.json                     # Saved Phase 6f — input to create_webset
├── webset-response.json                 # Saved Phase 6h — full items + enrichments
├── lovelace-contacts.json               # Saved Phase 6j — LinkedIn results per company
├── CONTEXT.md                           # Generated Phase 4
├── data.js                              # Generated Phase 7 (overwrites template's)
├── index.html                           # Generated Phase 8 (customized from template/)
├── BUILD_NOTES.md                       # Generated Phase 9
└── .git/                                # Initialized Phase 0; commit per phase
```

Why this structure:
- **`inputs/` separate** — raw docs are read-only reference; output files don't get confused with input files. When refreshing a build later, the user drops new material in `inputs/` and re-runs.
- **`template/` co-located** — Claude can grep `template/TEMPLATE_GUIDE.md` for craft patterns at any phase without an MCP call. Cloning per-build snapshots the template — future template updates don't disturb existing builds.
- **Outputs at root** — matches V1 Valar (`alexg207/valar-fdi`). `index.html` references `data.js` from the same directory; moving them breaks rendering.
- **JSON intermediates persisted** — `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json` enable reproducibility. Re-running Phase 7 against saved data doesn't require re-firing the Webset.
- **Per-phase Git commits** — real version history. `git diff` across commits shows what each phase produced. Easy to revert if a phase produces bad output.

## How To Use
1. Confirm Claude Code MCP connectors are loaded (`/mcp` or equivalent in your Claude Code setup): Websets, Lovelace, Granola, basic Exa.
2. Create the working directory and drop raw founder docs in `~/fdi/<founder-slug>/inputs/`. Quality scales with input volume — more raw material = sharper Webset spec.
3. `cd ~/fdi/<founder-slug>/`
4. Activate this skill. Claude will walk you through Phase 0 → 10.
5. Confirm the inputs inventory check (Phase 0).
6. Answer targeted gap questions if Claude flags any during intake.
7. Confirm signal axis decisions when proposed (Phase 5).
8. Confirm the full Webset spec at the pre-run checkpoint (Phase 6f — most important review).
9. Wait for the Webset to populate (5–10 min async).
10. Confirm the post-enrichment summary and final company list (Phase 6h, 6i).
11. Review the output bundle (CONTEXT.md, data.js, index.html, BUILD_NOTES.md at repo root).
12. Iterate on copy/structure/data as needed before delivering to founder. The GitHub repo was created in Phase 0 — `git push` sends your changes; from there, connect to Vercel for hosting if you want a live preview URL.

---

## Prompt

You are running the FDI skill in Claude Code for a market development associate at Primary Venture Partners. You're operating inside a per-founder Git repo at `~/fdi/<founder-slug>/` (or wherever the user's working directory is). Your goal is to produce a complete, customized intelligence dashboard for a founder, end to end:

1. Capture the founder's vertical, ICP, and voice from raw docs in `inputs/` (intake)
2. Run an Exa Webset to discover and enrich ~30 ICP-matching companies (enrichment)
3. Customize the HTML template into a polished dashboard (build)

Output: 4 files at the repo root, plus 3 saved JSON intermediates:
- `CONTEXT.md` — the research brief
- `data.js` — fully populated with real companies and enrichment from the Webset
- `index.html` — branded, signal-labeled, hiring-regex-tuned for this vertical
- `BUILD_NOTES.md` — documents the structural choices you made
- `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json` — saved intermediates for reproducibility

This is an 11-phase build (Phase 0–10). Do not skip phases. Do not skip ahead. Each phase ends with a `git commit` so the build has clean version history.

---

### Phase 0: Founder kickoff + working directory setup

Phase 0 has two paths depending on where the user activates the skill:

- **Path A — Existing build (`pwd` is already inside `~/fdi/<slug>/`):** skip to 0c. The user is iterating on a build that already exists; don't re-create anything.
- **Path B — New build (`pwd` is `~`, `~/fdi/`, or anywhere else):** run the kickoff flow below to set up a new founder repo from scratch.

**Step 0a: Detect the path.**

```bash
pwd
ls -la inputs/ 2>/dev/null && echo "INPUTS_PRESENT" || echo "NO_INPUTS"
```

If `pwd` returns a path that contains `~/fdi/<something>/` AND `inputs/` exists at that path, you're on Path A — go straight to step 0c.

Otherwise you're on Path B. Run the kickoff:

**Step 0b: Kickoff flow (new build).**

Tell the user what's happening, then ask the questions **one at a time**. Don't bundle them — the terminal flow is better with sequential turns. Use defaults to make answers easy.

Start with a brief framing message, then ask question 1:

```
Looks like we're starting a new FDI build. I'll ask a few quick questions, then 
set up the working directory and a new GitHub repo.

When this is done you'll have:
  - Local directory: ~/fdi/<slug>/ with inputs/ and template/ subdirectories
  - GitHub repo: github.com/alexg207/<slug>-fdi (public, matching valar-fdi pattern)
  - Local Git repo connected to the remote

First question: What's the founder/company name? (e.g., "Valar", "Matterstack")
```

Wait for the answer. Then ask question 2 with a smart default derived from the company name:

```
Got it — <Company Name>.

Slug? This is the lowercase, dash-separated identifier used for directory name 
and GitHub repo. Default suggestion: <auto-generated-slug>. Type your own or 
just say "yes" to use the default.
```

The auto-suggested slug should be the company name lowercased with spaces → dashes and special chars stripped. E.g.:
- "Valar" → `valar`
- "Matterstack" → `matterstack`
- "General Motors" → `general-motors`
- "AT&T" → `att`

If the user says "yes" / "ok" / "use that," accept the default. Otherwise use what they typed.

Wait for answer. Then ask question 3:

```
Slug locked in: <slug>. The repo will be at github.com/alexg207/<slug>-fdi.

Next: one-line description of the company? This goes into the GitHub repo description 
(visible on github.com when someone visits the repo). Examples:
  - "BYOC inference optimization platform — installs in customer's cloud for 50-80% lower cost"
  - "AI-powered revenue cycle management for healthcare providers"

What's <Company Name>'s one-liner?
```

Wait for answer. Then ask question 4:

```
Last question: where are the raw founder docs (memo, deck, Granola transcripts, 
prospect calls)? Two options:

  (a) Give me a path like ~/Downloads/valar-docs/ and I'll copy them into the 
      new project's inputs/ directory automatically
  (b) Say "I'll drop them in" — I'll create the empty inputs/ directory and 
      pause until you've populated it

What works for you?
```

Wait for answer. Once you have all four, summarize back to the user before running anything:

```
Setup plan:
  - Founder: <Company Name>
  - Slug: <slug>
  - Description: <one-liner>
  - Raw docs: <path | "I'll drop them in">
  - Local: ~/fdi/<slug>/
  - GitHub: github.com/alexg207/<slug>-fdi (public)

Proceeding to create everything now...
```

Then run the setup commands. **Two paths depending on whether `gh` CLI is available:**

```bash
# First, check for gh CLI
gh auth status 2>&1 | head -3
```

**If `gh` is installed and authenticated** (no errors from `gh auth status`):

```bash
SLUG=<slug-from-user>
DESCRIPTION=<description-from-user>
DOCS_PATH=<path-from-user>  # e.g., ~/Downloads/valar/

# Create local directory structure
mkdir -p ~/fdi/$SLUG/inputs

# Copy raw docs if user provided a path
if [ -n "$DOCS_PATH" ] && [ -d "$DOCS_PATH" ]; then
  cp -r $DOCS_PATH/* ~/fdi/$SLUG/inputs/
fi

# cd into the new directory
cd ~/fdi/$SLUG

# Clone the FDI template
git clone https://github.com/alexg207/fdi-template.git template/

# Initialize git
git init
git add inputs/ template/
git commit -m "Phase 0: working directory setup, template cloned"

# Create the GitHub remote and push
gh repo create alexg207/$SLUG-fdi --public \
  --description "$DESCRIPTION" \
  --source=. \
  --remote=origin \
  --push

# Confirm
git remote -v
gh repo view --web 2>/dev/null || echo "Repo created at: https://github.com/alexg207/$SLUG-fdi"
```

**If `gh` is NOT installed or not authenticated:**

Do the same local setup, but skip the `gh repo create` step. Tell the user:

```bash
mkdir -p ~/fdi/$SLUG/inputs
# (copy docs if provided)
cd ~/fdi/$SLUG
git clone https://github.com/alexg207/fdi-template.git template/
git init
git add inputs/ template/
git commit -m "Phase 0: working directory setup, template cloned"
```

Then surface to the user:

```
Local working directory created at ~/fdi/<slug>/.
GitHub remote NOT created — gh CLI isn't installed or authenticated.

To create the GitHub repo manually, install gh (`brew install gh`), authenticate 
(`gh auth login`), and run:

  cd ~/fdi/<slug>
  gh repo create alexg207/<slug>-fdi --public --description "<desc>" --source=. --remote=origin --push

Or do it in the GitHub web UI: github.com/new, name it "<slug>-fdi", then 
`git remote add origin git@github.com:alexg207/<slug>-fdi.git && git push -u origin main`.

I'll proceed with the build now — you can connect the remote later.
```

**After setup completes**, confirm to the user:

```
✓ Local: ~/fdi/<slug>/ initialized as Git repo with template/ cloned
✓ GitHub: https://github.com/alexg207/<slug>-fdi (or "manual setup needed" message)
```

If the user said "I'll drop docs in later" instead of providing a path, stop here and wait for them to populate `inputs/`. Don't proceed to 0c without docs in `inputs/`.

**Step 0c: Inventory the raw founder docs.**

Once the working directory exists with `inputs/` populated:

```bash
ls -la inputs/
file inputs/*  # if available; otherwise infer from extension
```

Map them against the input contract:

**Required (skill quality drops sharply without these):**
- Investment memo (PVP or partner memo)
- Founder deck
- At least one Granola transcript of the founder talking

**Strongly recommended (skill quality improves materially with these):**
- Multiple Granola transcripts — founder voice triangulates across different audiences (investors vs. operators vs. customers); 2–3 transcripts gives much richer voice signal than 1
- Prospect or customer call transcripts — when founder describes ICP in a sales context, that's the highest-fidelity buyer profile signal
- Deep research / Claude research outputs
- Existing pipeline spreadsheet — strongest training set for "what does an in-ICP company look like"

**Useful when present:**
- Slack threads with founder commentary on prospects
- Prior FDI dashboards or call notes
- Earnings transcripts of named lookalikes
- Third-party press / analyst coverage

Post the inventory to the user in this format:

```
INPUTS CHECK:

Working directory: ~/fdi/<founder-slug>/
✓ Git repo initialized
✓ template/ cloned (commit abc123 from fdi-template)

What I see in inputs/:
✓ memo.pdf (487 KB) — investment memo
✓ deck.pdf (3.2 MB) — founder deck
~ granola-2026-04-24.txt (28 KB) — one Granola transcript
✗ No prospect call transcripts in inputs/
✓ deep-research.md (12 KB) — research output
✗ No pipeline spreadsheet in inputs/

Against the input contract:
✓ Required: investment memo
✓ Required: founder deck
~ Required: Granola transcript (1 present, 2–3 would help triangulate founder voice)
✗ Strongly recommended: prospect/customer call transcripts — none in inputs/
✓ Strongly recommended: deep research output — present
✗ Strongly recommended: pipeline spreadsheet — not in inputs/

This is enough to proceed. Quality would improve with:
1. 1–2 more Granola transcripts (especially of the founder talking with prospects or operators)
2. A prospect call transcript or two — these carry the richest buyer profile signal
3. Pipeline spreadsheet if one exists

Drop any of those into inputs/ now if you have them, then I'll re-inventory. Or say "proceed" to go with what's here.
```

Wait for the user's go-ahead. They might say "proceed" or "let me drop a few more in" — both are fine. If they're missing a *required* input, push back gently: "I can proceed but the [specific thing missing] usually carries the [specific signal it carries] — without it, the [specific output] tends to be [specific quality issue]. Want to grab it before we start?"

Don't be obnoxious about this. One round of "here's what I see, want to add anything?" is enough — don't ask twice.

If the user added new files, re-run `ls -la inputs/` and post the updated inventory before proceeding.

### Phase 1: Read everything before asking anything

Before any user questions or tool calls, read every raw doc in `inputs/` end-to-end. Use the right tool for each file type:

```bash
# Text files: read directly
cat inputs/granola-*.txt
cat inputs/deep-research.md

# PDFs: use pdftotext or your PDF-reading tool of choice
pdftotext inputs/memo.pdf - | less
pdftotext inputs/deck.pdf -

# CSVs: head + structure
head -20 inputs/pipeline.csv
wc -l inputs/pipeline.csv
```

Then read the template files in `template/` — `AI_INSTRUCTIONS.md` first (it's authoritative), then `TEMPLATE_GUIDE.md`, then `data.js` and `index.html` to understand the schema you're filling.

If accessible, skim `github.com/alexg207/valar-fdi` as the gold-standard reference example — especially its `data.js` SEGMENTS structure and CONTEXT.md tone. This tells you what good output looks like.

Verify Websets connectivity with a quick `list_websets` call. Catching a 402/401 here saves a 10-minute async wait wasted on a failed build. Don't proceed if Websets returns auth errors — surface the problem to the user.

**Read mode: extract, don't summarize.** Reading docs to summarize is a different mode than reading them to extract signal for a Webset spec. Do both, but lean toward extraction. Specifically as you read, build:

1. **The founder's profile.** Who they are, what they sell, why now. This goes into CONTEXT.md.
2. **The buyer's profile.** Who buys this. What is the buyer's *core business* (NOT the founder's core business)? What problem does the buyer have that the founder solves? What does the buyer look like at the size/stage where they'd buy? This goes into the Webset spec.
3. **Verbatim founder quotes.** Aim for at least 15 across these themes: ICP definition, competitive view, urgency drivers, risk acknowledgment, customer voice, win patterns, loss patterns. These power Phase 4 and especially `gtm_thesis` writing in Phase 7.

Confusing the founder profile with the buyer profile is the most common Webset failure mode. For Valar (an inference fabric company), the *buyer* is a bank, an insurer, a cybersecurity company, an AIOps platform — companies whose core business is something other than AI infrastructure but who consume inference internally. Substrate AI (a sovereign AI cloud) is NOT a Valar buyer; it's a competitor or peer. The Webset must search for buyers, not peers.

After reading, summarize what you absorbed in 5–8 lines: founder name, vertical, ICP shape, named lookalikes, named exclusions, the wow signal, notable gaps, count of verbatim quotes pulled. Wait for the user to confirm or correct.

Build a mental map of what's already known versus what's missing — that's what Phase 2's questions target. The user spent time gathering these docs; don't make them repeat what's already there.

### Phase 2: Ask targeted questions for what's missing

After reading, identify gaps. Ask the user ONLY for the gaps. Do not ask for things clearly answered in the documents. Group questions into themes:

**Founder & product** (skip if memo covers it)
- Who is the founder? What's their background?
- What does the product do, in one line, in their own words?
- What's the wedge / what makes this defensible?

**ICP** (skip parts already covered)
- Who are the near-term customers (Stage 1)?
- Who are the long-term customers (Stage 2)?
- What's the explicit qualifier — what specific characteristic must a company have to be in-ICP?
- The qualifier should be 3–5 *verifiable*, *objective* statements (e.g., "Company has $100M+ revenue," "Company runs production AI inference," "Company is in a regulated vertical"). These translate directly into Webset `searchCriteria` later. Subjective qualifiers like "company is innovative" don't translate well — push for concrete, source-able statements.

**Lookalike anchors (CRITICAL — always ask if not in docs)**
- Name 3–5 specific companies that fit the ICP perfectly. The "if every company looked like this, the founder would be thrilled" examples.
- Why does each one fit?
- These get embedded directly in the Webset `searchQuery` as positive anchors — they materially shape what comes back.

**Exclusions (CRITICAL — always ask if not in docs)**
- Companies that should NEVER appear: existing customers, direct competitors, dramatically wrong-size segments, anything the founder has explicitly flagged as "no."

**The "wow" signal (CRITICAL — always ask)**
- What single insight, when shown to the founder, would make them lean forward and say "yes, you GET it"?
- The most important question. Push for specificity. If the user gives a generic answer ("companies with growing AI spend"), push back: "what specifically about that signal would surprise the founder?"

**Founder voice quotes (CRITICAL)**
- Pull 5–15 verbatim quotes from the docs (transcripts, memos, calls). Cover: product vision, ICP language, messaging, market timing view, competitive view, risk acknowledgment.
- If quotes are thin, ask the user for any that come to mind from recent conversations.

**Buyer personas**
- Who is the buyer? Who is the champion? Are there antagonist personas to avoid?
- Ranked messaging angles — what works best, what's secondary?

**Visual / structural preferences**
- Are there things the founder cares about visually? (Lead with map view, table, segments?)
- Any reference dashboards or prior projects that resonated?

### Phase 3: Post a structured extraction summary

Before generating CONTEXT.md, post back to the user what you've extracted as a checklist:

```
EXTRACTION CHECK:
✓ Founder background — captured from memo
✓ Product description — captured from memo + deck
~ ICP definition — partial; mid-market clear, enterprise vague
✗ Lookalike anchors — needs your input
✗ Exclusions — needs your input
✓ Founder voice — 8 quotes pulled from Granola transcript
✗ The "wow" signal — needs your input
~ Buyer personas — implicit in memo, needs explicit ranking from you
```

Wait for the user's confirmation or corrections. This catches read errors in one pass.

### Phase 4: Generate CONTEXT.md

Produce CONTEXT.md following the structure in `template/CONTEXT_TEMPLATE.md`. Write to repo root: `./CONTEXT.md` (not into `inputs/` or anywhere else — V1 Valar puts CONTEXT.md at root next to data.js and index.html).

Required sections (in order):

1. **Header** — Founder name, generation date, sources used (list every file in `inputs/` you read)
2. **What [Product] Does** — One line + paragraph
3. **Round + Investment** — Round size, valuation, Primary's check, lead position
4. **Team** — Founders, key hires, prior playbook
5. **The Bet (ICP Qualifier)** — Specific shape of who pays + ACV range
6. **Real Pipeline** — Signed design partners, named pipeline, potential customers count
7. **Market Context** — Sized market, structural shifts, urgency drivers
8. **Competitive Landscape** — Table format: category / players / why they fail
9. **Key Risks** — Top 3 risks with explanations
10. **Buyer Insights** — Per-buyer interview takeaways with verbatim quotes
11. **Investment Thesis** — 3 thesis points
12. **Gotta Believes** — What has to be true for the bet to work
13. **How [Product] Came to Primary** — Origin story
14. **Market Segmentation** — Competitors / customers / partners breakdown
15. **ICP (Two Stages)** — Stage 1 (now) and Stage 2 (later)
16. **Buyer Personas & Messaging** — Ranked targets, ranked angles
17. **Key People** — Table of relevant people on both sides
18. **Founder Voice — Verbatim Quotes** — at least 15 direct quotes, organized by theme (CRITICAL — pulled from Phase 1's reading)
19. **Lookalike Anchors** — 3–5 named companies (CRITICAL)
20. **Exclusions** — Named companies to never include (CRITICAL)
21. **The "Wow" Signal** — The insight that makes the founder lean forward (CRITICAL)
22. **Deliverable / Timeline** — Due date, audience, format, success metric
23. **Key Documents & Links** — Table of resources
24. **Spreadsheet Data** — Any company/contact lists from `inputs/*.csv`
25. **Slack Timeline** — Optional chronological log
26. **Additional Context** — Catch-all

Commit after writing:

```bash
git add CONTEXT.md
git commit -m "Phase 4: CONTEXT.md generated from raw docs"
```

After CONTEXT.md is written and committed, briefly confirm to the user it's ready. Then move into structural decisions (Phase 5).

### Phase 5: Derive the signal axes

This is the most important creative step in the entire build. The signal axes determine: what the Webset enrichment columns are, how each company gets scored, what the dashboard looks like, and how the founder will read the output. Get this wrong and everything downstream is off-target.

**The structure: 2 mandatory + 2-4 founder-specific = 4-6 axes total.**

**Mandatory axis 1: Hiring Score.** Always present. Measured by scraping the web for active job postings at each target company and drawing conclusions from what they're hiring for. The signal isn't just "are they hiring" — it's "are they hiring roles that the founder's product would replace, augment, or otherwise be relevant to?" For Valar: ML platform engineers, inference infrastructure roles. For a healthcare workflow founder: prior auth specialists, RCM directors, denial management leads. For a fintech founder: payment ops, compliance engineers, fraud analysts.

**Mandatory axis 2: Opportunity Score.** Always present. Measured by synthesizing publicly available financial and strategic data: 10-Ks, quarterly earnings reports, C-suite commentary on earnings calls, investor presentations, news articles, M&A activity, leadership changes, strategic announcements. The signal: based on this company's stated direction, financial position, and recent activity, how strong is the buying opportunity *right now*?

**Founder-specific axes (2 to 4 of them):**

These you derive from the founder's docs. The question to ask: "If a savvy GTM hire at this company were to score every prospect on 4 dimensions, what would those dimensions be?" The dimensions should be what the founder cares about most — the things that, if all 4 were "high" for a target, the founder would say "yes, that's a perfect customer."

For Valar, these were:
- **Inference Pain** (do they run heavy production inference? at what scale? on what stack?)
- **Data Residency** (do regulatory or contractual constraints block multi-tenant inference clouds?)

So Valar's full axis set was: Hiring + Opportunity + Inference Pain + Data Residency = 4 axes.

For a healthcare workflow founder, founder-specific axes might be:
- **Workflow Pain** (% of manual prior auth, denial volume, days in A/R)
- **Compliance Burden** (HIPAA depth, state Medicaid friction, recent regulatory issues)

Total = 4 axes.

For a fintech infra founder, founder-specific axes might be:
- **Payment Volume** (transaction throughput, # of payment rails)
- **Compliance Posture** (PCI DSS level, state money transmitter licenses, recent fines)
- **Tech Debt** (legacy core banking? mainframe? old payment processor?)

Total = 5 axes.

**How to derive the founder-specific axes:**

1. Look at CONTEXT.md's "ICP Qualifier" — what specific traits define an in-ICP company? Each distinct trait is a candidate axis.
2. Look at the lookalike anchors and ask: "What do these companies have in common that makes them ICP-perfect?" Each shared trait is a candidate axis.
3. Look at the founder's quotes about why deals work and don't work — the patterns are candidate axes.
4. Look at the "wow signal" the founder cares about — that's almost always one of the axes.
5. Cull to the 2-4 axes that are most discriminating (i.e., the ones that vary most across the population of potential targets).

Avoid: axes that are non-discriminating (e.g., "Has a website" — everyone has one) or axes that map to firmographics already captured elsewhere (e.g., "Is in the US" — that's a geographic filter, not an axis).

**Output: axis definitions, in the format the rest of the skill needs.**

For each axis, document:
- **Axis name** (what shows in the dashboard UI)
- **What it measures** (one sentence)
- **Data sources** (where the underlying signal comes from — web scraping job sites for hiring, 10-Ks for opportunity, third-party press for compliance, etc.)
- **0-5 scoring rubric** (what does a 0 look like, what does a 5 look like, what's in between)
- **Webset enrichment column it maps to** (one or more)

**Then, structural decisions** (faster — these are derivative of the axes):

- **Segments** — keep the default (Pipeline / Mid-Market / Enterprise)? Drop one? Add a vertical or geographic segment?
- **Sections per company** — keep default 3 (Profile / Opportunity / GTM)? Rename the Opportunity section per founder vertical?
- **Tab labels** — what should the segment tabs read as in the founder's language?

Post the full plan to the user as a structured proposal:

```
SIGNAL AXES (Phase 5):

Mandatory:
1. Hiring Score
   Measures: Roles posted that suggest pain Valar's product would address
   Sources: Job boards, careers pages, LinkedIn job listings
   Rubric: 0 = no relevant openings; 3 = scattered openings; 5 = active hiring spree for ML platform/inference roles
   Maps to: enrichment "active job postings"

2. Opportunity Score
   Measures: Strength of buying opportunity from public financial + strategic signals
   Sources: 10-Ks, earnings transcripts, C-suite commentary, M&A activity, leadership changes
   Rubric: 0 = no buying signals; 3 = mild signals (moderate AI mentions); 5 = explicit AI cost/strategy commitments and recent leadership changes
   Maps to: enrichment "financial signals" + "strategic commentary"

Founder-specific:
3. Inference Pain
   Measures: Production AI inference scale and stack pain
   Sources: Engineering blogs, conference talks, third-party press
   Rubric: 0 = no production inference; 5 = >$10M annualized inference spend with documented stack pain
   Maps to: enrichment "production inference workloads with scale evidence"

4. Data Residency
   Measures: Regulatory/contractual constraints that block multi-tenant inference clouds
   Sources: Compliance pages, regulatory filings, customer agreements
   Rubric: 0 = no constraints; 5 = explicit data residency in regulated geo + customer SLAs
   Maps to: enrichment "data residency, sovereignty, or regulatory constraints"

STRUCTURAL DECISIONS:
- Segments: Pipeline / Mid-Market / Enterprise (default kept)
- Section 2 renamed: "Opportunity" → "Inference Footprint" (per Valar vertical)
- Tab labels: Pipeline / Mid-Market / Enterprise

Confirm or push back before I move to Webset spec design.
```

Wait for user confirmation. This is the cheapest place to course-correct.

### Phase 6: Build and run the Webset

This is the main enrichment phase. **The skill's success or failure depends on the quality of the Webset spec.** A great spec returns 25 ICP-perfect companies with rich enrichments. A mediocre spec returns 25 vendors-and-peers with thin data.

The spec is built up across 6a-6f as a multi-step subroutine. Don't shortcut it.

**6a — Re-extract the buyer profile from raw docs (NOT from CONTEXT.md)**

Before writing any spec text, re-read the raw docs in `inputs/` — specifically the founder transcripts, prospect call transcripts, and the relevant memo sections on ICP and buyers. Don't shortcut by re-reading CONTEXT.md; it's a compression and you need the texture.

```bash
# Re-read the raw docs that carry buyer profile texture
cat inputs/granola-*.txt
cat inputs/prospect-call-*.txt 2>/dev/null
pdftotext inputs/memo.pdf - | grep -iA 30 "ICP\|buyer\|customer\|target"  # quick orient
```

Now write two paragraphs, in your own words:

1. **What is the buyer's core business?** (Banking, healthcare claims, threat detection, AIOps, etc. — the buyer's industry and the buyer's primary product or service. NOT the founder's industry. NOT the founder's product.) Pull verbatim phrases from how the founder describes their target buyer if available — those phrasings are higher-signal than synthesized descriptions.
2. **What problem does the buyer have that the founder solves?** (The pain, not the solution. E.g., "buyer runs heavy AI inference and faces strict data residency constraints" — that's the pain. "buyer needs Valar's inference fabric" — that's the solution. Use the pain phrasing.)

This is the disambiguation step. Confusing the buyer with the founder produces garbage Webset results. Common failure mode: searching for "AI inference companies" returns *AI inference vendors*, not *AI inference buyers*. The two paragraphs above force you to be specific about which side of the market you're targeting.

**Why raw docs, not CONTEXT.md?** CONTEXT.md's "ICP Qualifier" is a compression to 3-5 verifiable statements. The texture you need — offhand asides in transcripts, half-finished thoughts, analogies, the founder's attitude toward different competitors — only lives in raw founder talk. The Webset spec quality is bounded by how well-grounded these two paragraphs are.

**6b — Translate axes into enrichment columns**

For each axis you defined in Phase 5, write 1-3 Webset enrichment column descriptions. Each description should be a full paragraph (50-100 words) that specifies:

- **What signal to extract** (concrete, specific to the founder's vertical)
- **What sources to consult** (third-party preferred — SEC filings, analyst reports, engineering blogs, third-party press; NOT the company's own marketing site for credibility-sensitive fields)
- **How to phrase the answer** (specific phrasing requirements — "in USD," "with quoted excerpt," "with source URL inline")
- **What format** (`text` for narratives, `number` for monetary/count values, `options` for controlled vocabulary, `boolean` for binary)

Example — for Valar's "Inference Pain" axis, instead of:

> "Documented pain points or constraints" (BAD — too generic, gets generic data)

Use:

> "Documented evidence that this company runs production AI inference at scale, citing engineering blogs, conference talks, or third-party technical press (NOT the company's marketing pages). Include: estimated annualized inference spend if mentioned, the scale of inference workloads (requests/second, tokens/day), the current inference stack (GPU type, framework, deployment pattern), and any documented pain points (cost overruns, latency issues, capacity constraints). Format: 2-4 sentence narrative with at least one source URL inline. If the company appears to BE an inference vendor rather than an inference consumer, mark this field 'NOT APPLICABLE — vendor not buyer.'"

The "vendor not buyer" exclusion catches the failure mode from the Valar test where Substrate AI and Oxmaint came back as ICP matches when they're actually vendors.

Standard enrichments to include in every FDI Webset (in addition to axis-specific ones):

```javascript
[
  // Identity
  {description: "Industry classification, SIC or NAICS preferred", format: "text"},
  {description: "Most recent annual revenue in USD with reporting year", format: "text"},
  {description: "Headquarters city and state/country", format: "text"},

  // Hiring axis (mandatory)
  {description: "Up to 5 currently active job postings relevant to [vertical-specific roles] with title, key technologies/responsibilities mentioned, and URL to the posting. Source: company careers pages, LinkedIn jobs, or third-party job boards.", format: "text"},

  // Opportunity axis (mandatory)
  {description: "Recent (last 12 months) public financial and strategic signals: 10-K commentary on AI/[vertical], earnings call quotes from C-suite about [relevant topic], leadership changes, M&A activity, strategic announcements. Use SEC filings and earnings transcripts as primary sources. Format: 3-5 specific signals with quoted phrasing and source URLs.", format: "text"},

  // Contacts (for CONTACT_MAP)
  {description: "Top 2 likely [buyer/champion personas — vertical-specific, e.g., 'Head of ML Infrastructure' for Valar, 'VP of Revenue Cycle' for healthcare] at this company with full name, exact role, and LinkedIn URL.", format: "text"},

  // Sources
  {description: "5 verifiable third-party source URLs from credible publications (Bloomberg, Reuters, TechCrunch, industry analysts, SEC.gov, peer-reviewed publications) supporting the above. Avoid the company's own marketing pages.", format: "text"}
]
```

For each axis-specific enrichment, write 1-3 columns following the same pattern. Aim for **8-12 enrichment columns total** across all axes plus standard fields.

**6c — Write the searchQuery**

The searchQuery is a long natural-language paragraph (150-300 words) that describes the buyer profile. Built from:

- The two paragraphs from 6a (buyer's core business + buyer's pain)
- The lookalike anchors from CONTEXT.md (named explicitly as positive examples — pull every name from the Pipeline / Lookalike / Stretch ICP / Outbound spreadsheet sections)
- Explicit exclusions to prevent vendor/peer matches (pull every named competitor from CONTEXT.md's "Competitive Landscape" table — don't just name categories, name the actual companies)

Template:

> "[Buyer's core business description — 1-2 sentences, drawing from 6a paragraph 1.] Their core business is NOT [founder's industry/product category]; they CONSUME [founder's product type] internally as a capability for their own operations, rather than producing or selling it. [Pain description — 1-2 sentences, drawing from 6a paragraph 2.] Strongest fit: [3-5 vertical descriptors from CONTEXT.md ICP], and companies similar to [3-5 lookalike anchors named explicitly]. EXCLUDE: [vendor categories that should NOT match, AND specifically named competitors from CONTEXT.md's Competitive Landscape table — e.g., for Valar, 'AI infrastructure vendors, AI cloud providers, GPU resellers, AI training data companies, ML platform companies, sovereign AI cloud providers, including but not limited to: Fireworks, Together, Baseten, Modal, Groq, Cerebras, Crusoe, Coreweave, Nebius, Scale AI, Substrate AI']."

The "EXCLUDE" clause is doing real work. Webset agents respect negative phrasing when it's explicit and concrete. Be specific — name competitor categories AND, where possible, name the actual competitors from CONTEXT.md (e.g., for Valar: "Fireworks, Together, Baseten, Modal, Groq, Cerebras, Crusoe, Coreweave, Nebius, Scale AI, Substrate AI"). Generic phrases like "AI vendors" leak; named exclusions hold.

**6d — searchCount**

Default: **35**. Final dashboard typically features 18-22 companies, so oversample by ~70% to drop weak matches without re-running. Lower to 25 if cost is a concern; raise to 45 if the founder vertical is broad and the lookalike list is large. Don't go below 25 — a thin Webset means a thin dashboard.

**6e — Write the searchCriteria**

3-5 hard, verifiable criteria each company must satisfy. Best practices:

- **Objective and verifiable on the open web** ("Company has $100M+ revenue OR is publicly traded" — verifiable via SEC filings or news. NOT "Company is innovative" — subjective.)
- **At least one criterion is exclusion-shaped** (e.g., "Company's primary business is NOT in the [founder's product category] — companies that sell [founder's product type] are excluded"). Reinforces the EXCLUDE clause from the searchQuery — don't rely on the searchQuery alone.
- **Pull from CONTEXT.md's "ICP Qualifier" section** — the 3-5 verifiable statements you wrote in Phase 4. If the founder gave you a hard threshold (e.g., "$1.5M+ annualized inference spend"), translate it to a revenue or scale proxy that's externally verifiable.
- **Don't over-specify** — Webset will return fewer results if criteria are too narrow. Aim for criteria that 30-40% of in-vertical companies would pass.

**6f — Pre-Webset checkpoint**

Before firing the (paid, async) Webset, do two things:

**Pre-flight: `preview_webset` (free)**

Call `preview_webset` with your `searchQuery` first. This is a free Exa endpoint that returns Webset's interpretation of the query: detected `entityType`, generated `criteria`, and suggested `enrichments`. It's a free sanity check on whether your buyer-profile paragraph reads to Webset the way you intended.

Compare what comes back against what you wrote in 6a-6e:

- Does the detected `entityType` match yours (`company`)? If Webset flags it as something else (`person`, `custom`), the query is being read wrong.
- Do the generated `criteria` overlap with yours? If Webset's auto-generated criteria look meaningfully different — especially if it's missing your exclusion-shaped criterion — the searchQuery's EXCLUDE clause may not be landing. Strengthen it.
- Are Webset's suggested `enrichments` adjacent to yours, or wildly different? If wildly different, your enrichment phrasing in 6b may not match the founder's vertical the way you meant.

Don't blindly accept Webset's suggestions — your enrichments are tailored to the data.js schema, theirs are generic. But significant divergence is a yellow flag worth investigating before you fire.

**User checkpoint**

Post the COMPLETE spec to the user:

```
WEBSET SPEC PROPOSAL:

searchQuery (the buyer profile paragraph, full text):
> [paste here]

searchCriteria (3-5 hard filters):
- [criterion 1]
- [criterion 2]
- ...

searchCount: 35 (oversample by ~70% — final dashboard 18-22 companies)
entity: company

enrichments (8-12 columns, full descriptions):

Identity:
- [enrichment 1, full description]
- [enrichment 2, full description]
- ...

Hiring axis (mandatory):
- [enrichment]

Opportunity axis (mandatory):
- [enrichment]

[Founder-specific axis 1: name]:
- [enrichment(s)]

[Founder-specific axis 2: name]:
- [enrichment(s)]

Standard:
- Contacts: [enrichment]
- Sources: [enrichment]

preview_webset returned:
- Entity type: [company / person / custom — should match company]
- Auto-generated criteria: [list]
- Notable divergence from my proposal: [flag any]

Total: ~30 companies × ~10 enrichments. Estimated cost ~$5-8. Estimated time 5-10 min.

Before I fire this, please confirm:
1. Does the searchQuery accurately describe [Founder]'s buyers (not peers/vendors)?
2. Are there exclusions I'm missing? (Specific vendor categories, geographies, etc.)
3. Are there axis-specific signals I should add or rephrase?
4. Does the lookalike list match what you'd want?
```

Wait for explicit go-ahead. **This is the single most important checkpoint in the entire skill.** It's also the cheapest place to course-correct — fixing the spec before firing is much cheaper than fixing results after.

**6g — Submit and wait**

After user confirmation, save the spec to disk first (for reproducibility), then submit:

```bash
# Save the spec as webset-spec.json so it lives with the build
cat > webset-spec.json <<'EOF'
{
  "searchQuery": "...",
  "searchCount": 35,
  "searchEntity": {"type": "company"},
  "searchCriteria": [...],
  "enrichments": [...]
}
EOF

git add webset-spec.json
git commit -m "Phase 6g: Webset spec saved before submission"
```

Now call `create_webset` with the spec. The response contains a webset ID — capture it. Tell the user:

```
Webset submitted. ID: webset_abc123
Searching for ~35 companies, populating ~10 enrichment fields per company.
Estimated time: 5–10 minutes.

Monitor progress at: dashboard.exa.ai/websets/webset_abc123
I'll continue when it's idle.
```

Poll `get_webset` every 30 seconds until status is `idle`. (Tip: the first poll should wait at least 10 seconds after `create_webset` per Exa's recommendation.) If a Webset takes >15 minutes, surface to user and decide whether to abort and retry with looser criteria.

**6h — Pull and review items**

Once idle, call `list_webset_items` (paginate if needed for >25 results). Save the full response immediately:

```bash
# Save the full Webset response — items, enrichments, source URLs
# Tool output goes to webset-response.json for reproducibility
# (write the JSON the tool returned to this file)

git add webset-response.json
git commit -m "Phase 6h: Webset response captured (28 companies)"
```

Why save it: data.js population in Phase 7 reads from this file, not from re-calling `list_webset_items`. If you need to re-run Phase 7 (reword a `gtm_thesis`, fix a tier assignment), you don't have to re-fire the Webset.

Now post a summary to the user:

```
WEBSET COMPLETE:
- Companies returned: 28 (requested 35)
- Matching all criteria: 21
- Matching most criteria: 7 (review case-by-case)
- Enrichments fully populated: 23/28
- Sparse enrichment (>30% blank): 5 — flagging [Company X], [Y], [Z]
- Notable absences: [Lookalike from CONTEXT.md] not in results — should I rerun with adjusted query?

Vendor/peer false positives detected (will exclude from data.js): [list any]

Companies with strongest signal across enrichments: [list top 8]
Companies that look thin: [list]

Saved: webset-response.json (full enrichment data, committed to repo)

Proceed to curation, or adjust the company list / re-run with different criteria first?
```

**Specifically check for vendor/peer matches** that slipped past criteria — companies whose primary business is selling the founder's product category. Flag these and exclude them from data.js. If 30%+ of results are vendor/peer matches, the searchQuery's exclusion clause needs strengthening — rerun with a tighter spec.

Wait for confirmation.

**6i — Curate the final company list**

Before contact discovery and data.js population, you need to commit to a final list. This is the moment to drop weak matches, add specific founder-named companies that the Webset missed, and lock in segment assignments. Don't skip this — the dashboard's quality is bounded by which companies make the cut.

Three categories of edits to make explicit with the user:

**Drop:**
- Vendor/peer false positives (already flagged in 6h)
- Companies with sparse enrichments (>30% blank fields) where you can't find supplemental data via web search
- Companies that match criteria but don't fit the founder's actual GTM motion (e.g., wrong size, wrong region, named exclusion in CONTEXT.md)

**Add:**
- Lookalike anchors from CONTEXT.md that the Webset didn't surface (often the case when a company's web presence is thin even though they're a perfect ICP fit)
- Companies the founder named in transcripts/memos that didn't make it through Webset criteria
- Pipeline / design-partner companies (Pipeline segment of the dashboard — these go in regardless)

For added-back companies, you need to manually research them: pull data via `Exa:web_search_exa` and `Exa:web_fetch_exa` to populate the same enrichment fields the Webset produced for others. Or use `create_enrichment` on the existing Webset to add specific companies via the import flow if they're missing entirely.

**Segment assignment:**
- Pipeline = signed design partners + named active pipeline accounts (from CONTEXT.md)
- Mid-Market = Stage 1 ICP (the founder's "easier to convert" segment)
- Enterprise = Stage 2 ICP (founder's larger/longer-cycle targets)

Some companies will fit multiple segments. The default rule: if a company is in Pipeline, it goes in Pipeline regardless of size. If it's not, size + signal strength determines segment.

Post the final list to the user as a structured proposal:

```
FINAL COMPANY LIST PROPOSAL:

Pipeline ([N] companies):
- [Company A] — [signed design partner / active pipeline / named in pipeline]
- [Company B] — [...]
- ...

Mid-Market ([N] companies):
- [Company C] — Webset rank 4, all 4 axes ≥3
- [Company D] — Webset rank 7, strong residency signal
- ... 

Enterprise ([N] companies):
- [Company E] — Webset rank 1, scale + opportunity signal
- ...

DROPPED from Webset (with reasons):
- [Company X] — vendor/peer false positive
- [Company Y] — sparse enrichments, couldn't supplement
- [Company Z] — wrong-size/wrong-region for ICP

ADDED to fill gaps (need manual enrichment):
- [Lookalike anchor from CONTEXT.md not in Webset]
- [Founder-named company missed by Webset]

Total: [N] companies (target: 18-22).

Confirm or revise before I move to contact discovery + data.js population.
```

Wait for user confirmation. Course-correction here is much cheaper than fixing a populated dashboard.

**6j — Contact discovery via Lovelace**

Run `Lovelace:search_linkedin_profiles` per target company (max 10 results per call). Two queries per company is usually right: one for the primary buyer persona, one for the technical champion persona.

The persona phrasing must come from the founder's vertical and from CONTEXT.md's antagonist warnings — generic "engineering leader" returns garbage.

**Per-vertical persona template patterns:**

For **inference / AI infrastructure** founders (Valar-shaped):
- Buyer: `keywords: "platform engineering"` or `keywords: "ML infrastructure"` or `keywords: "AI platform"`
- Champion: `keywords: "site reliability"` or `keywords: "cloud infrastructure"` or `title: "VP Platform Engineering"`
- EXCLUDE: ML engineering / applied ML / data science roles unless they're the platform owner

For **healthcare workflow** founders (RCM / prior auth / claims):
- Buyer: `title: "VP Revenue Cycle"` or `title: "Director Revenue Cycle Management"`
- Champion: `keywords: "denial management"` or `keywords: "prior authorization"` or `title: "Director RCM Operations"`
- EXCLUDE: Pure clinical roles (MDs, RNs) unless they hold an admin operations title

For **fintech infrastructure** founders (payments / banking / compliance):
- Buyer: `title: "Head of Payments"` or `keywords: "payments engineering"`
- Champion: `keywords: "fraud engineering"` or `title: "VP Compliance Engineering"`
- EXCLUDE: Retail banking / branch management unless explicit infra mandate

For **cybersecurity** founders:
- Buyer: `title: "CISO"` or `keywords: "security engineering"` or `title: "VP Security Operations"`
- Champion: `keywords: "detection engineering"` or `keywords: "security platform"`
- EXCLUDE: Compliance/audit unless they own platform decisions

For **data / analytics infra** founders:
- Buyer: `keywords: "data platform"` or `title: "VP Data Engineering"`
- Champion: `keywords: "analytics engineering"` or `title: "Head of Data Infrastructure"`

If the founder's vertical isn't in the templates above, derive the patterns from CONTEXT.md the same way: who buys (decision-maker title), who champions internally (technical advocate keyword), who to exclude (founder's named antagonist personas).

**Tool signature:**

```javascript
search_linkedin_profiles({
  company: "<exact company name from Webset>",
  keywords: "<persona keyword from vertical template>",
  // optionally:
  title: "<seniority title if known>",
  location: "<filter to founder's GTM geo if relevant>",
  max_results: 10  // cap per Lovelace
})
```

**Warm-intro mapping:**

For each LinkedIn profile returned, cross-reference against `PRIMARY_TEAM` and the Primary network connections list in CONTEXT.md (if present). For each match:

1. Tag `primary_connection: "<teammate name>"` in the contact's CONTACT_MAP entry
2. In the contact's `note` field (if you add one), specify the connection: "Warm via [Teammate]: [shared context — same school, prior coworkers, mutual connection X]"

Warm intros are the highest-signal contact for the founder. Surface them in the dashboard's Network view and in `gtm_thesis` if a target company has multiple warm paths.

**Sequencing:**

For a 25-company Webset, that's potentially 50 Lovelace calls. Sequence intelligently — high-tier companies first, low-tier last. Stop early if you hit rate limits or low signal.

**Save to disk:**

Once contact discovery is done, save the aggregated results so Phase 7 can populate CONTACT_MAP from disk (not from re-calling Lovelace):

```bash
# Save contacts as lovelace-contacts.json keyed by company name
# {"BigPanda": [{name, title, linkedin, primary_connection}, ...], "Varonis": [...], ...}

git add lovelace-contacts.json
git commit -m "Phase 6j: Lovelace contacts gathered (47 profiles across 23 companies)"
```

**6k — Fallback ladder if Webset returns thin or off-target**

If <15 companies came back (thin):
1. Loosen one criterion (often the most specific)
2. Broaden the searchQuery (add "or similar" hints to the lookalike list)
3. Rerun with the same enrichments

If 30%+ of results are vendor/peer matches (off-target):
1. Strengthen the EXCLUDE clause in searchQuery with more specific vendor categories
2. Add an explicit exclusion criterion to searchCriteria
3. Rerun

If specific enrichments are blank for many companies (sparse data):
1. Reword the enrichment description (often a phrasing or sourcing issue)
2. Add a fallback enrichment via `create_enrichment` on the existing Webset

**Cost awareness**

A real test on May 4 with 5 companies × 3 enrichments returned in ~5 minutes for ~$0.50–1. Production Webset of 30 companies × 10 enrichments ≈ $5–8, ~5–10 minutes. Lovelace per-call. Total per FDI build: $7–15.

### Phase 7: Populate data.js

**Before writing any company entries, re-read `template/TEMPLATE_GUIDE.md` Section 9 ("Field-by-field craft patterns").** This section extracts concrete writing patterns from the V1 Valar dashboard — every example is real V1 copy. Following the schema is necessary but not sufficient; following the *patterns* is what separates a generic dashboard from one that lands.

```bash
# Refresh on the patterns before populating
sed -n '/^## 9\./,/^## 10\./p' template/TEMPLATE_GUIDE.md
```

**Working from disk:** read company data from `webset-response.json`, contacts from `lovelace-contacts.json`. Don't re-call MCP tools at this phase — the saved files are your source of truth.

**Build incrementally — one company at a time, not in batch.** Generic batch-population is the failure mode that produces "Valar with names changed" output. After each company, run a quick self-check before moving on. This is slower but produces dramatically higher quality.

**Start by copying the template's `data.js` as your base:**

```bash
cp template/data.js data.js
# Open data.js, identify the const declarations: ROW_SOURCES, SEGMENTS, CONTACT_MAP,
# COMPANY_SOURCES, JOB_LISTINGS, RESIDENCY_MAP, PRIMARY_TEAM
# You'll fully replace SEGMENTS, CONTACT_MAP, COMPANY_SOURCES, JOB_LISTINGS, RESIDENCY_MAP, ROW_SOURCES.
# Keep PRIMARY_TEAM array structure but populate with Primary's actual roster from CONTEXT.md.
```

**For each company from the curated list (Phase 6i):**

1. **Pick the right segment** — based on signal strength + ICP match (Pipeline if signed/active, Mid-Market for Stage 1 ICP, Enterprise for Stage 2).

2. **Set `tier`** — `'high'` if all axes scored 4+ AND the company is signed/in-pipeline OR has a strong warm intro; `'med'` for strong ICP fit with mixed axis scores; `'low'` for speculative pattern-matches without warm path. Aim for the V1 distribution across ~30 companies: 10 high, 12 med, 8 low. If everything is `'high'`, tiers carry no information.

3. **Write `subtitle` in V1 pattern** — *[what the company is] — [why-they-fit-the-founder phrase], [founder relationship status]*. One sentence, dense, signal-rich. See TEMPLATE_GUIDE.md Section 9.1.

4. **Write `overview` in V1 pattern** — 3-5 sentences that name the company's position in the founder's market story, specify the data sensitivity in concrete terms (not abstract), and connect the company to a category-level reference. See Section 9.2.

5. **Fill the 3 sections** (Profile / [renamed Opportunity] / GTM Strategy) directly from Webset enrichments:
   - Profile: ~6 rows. Last row is always founder-relationship status. Cite source materials inline. See Section 9.8.
   - Opportunity: ~4 rows. Estimated Spend always uses a range. Pain Points uses contractual/business language, not technical jargon.
   - GTM Strategy: ~5 rows. Urgency Level uses uppercase action verbs (EXECUTE / HIGH / WARM / MED / COLD / DEFER). Target Buyer splits Buyer + Champion when both are knowable. Messaging Angle includes a quoted opening line.

6. **Score the founder-specific axes 0–5** and write the trio for each: score + bullet signals + reasoning paragraph. See Section 9.5-9.7. The reasoning paragraph should be honest about challenges (V1 Mastercard's `opp_reason` flagged in-house expertise as a hurdle) — credibility carries.

7. **Write `gtm_thesis` in the V1 three-sentence pattern**: anchor sentence + motion sentence + buyer call-out. Splice verbatim founder quotes from `inputs/granola-*.txt` or CONTEXT.md if they fit. End with `**Buyer:** [persona]` (and `**Champion:**` if known) — use **NOT [persona]** when the founder has named antagonist personas. See Section 9.3.

8. **Write `tags` 3-5 chips with mixed colors** (`Valar`/`brand` for relationship, `stack` for technical/constraint, `hw` for hard constraint, `hiring` for hiring signal, `neutral` for factual). Tooltips required if the tag is non-obvious. Hiring tags prefixed with the role being hired (e.g., `'Hiring: ML Platform'` for inference founders, `'Hiring: RCM'` for healthcare workflow, `'Hiring: Payments'` for fintech). See Section 9.4.

9. **Populate `CONTACT_MAP`** with platform/infrastructure leadership keyed exactly to `SEGMENTS[].companies[].name` (character-for-character match including parentheses). Read from `lovelace-contacts.json`. Persona discipline reflects founder antagonist warnings — exclude personas the founder has flagged. Cross-reference Primary network for warm intros. See Section 9.10.

10. **Populate `COMPANY_SOURCES`, `JOB_LISTINGS`, `RESIDENCY_MAP`** from `webset-response.json`. Sources should be third-party (SEC, Bloomberg, Reuters, engineering blogs) — avoid the company's own marketing pages.

11. **Cite via `ROW_SOURCES`** for every numeric or specific claim. Webset returns sources inline within text fields (pattern: `fact text | URL / fact text | URL`); parse these into ROW_SOURCES entries. Empty entries are fine; wrong entries are worse than nothing. Source citations must include publication and approximate date. See Section 9.9.

**Self-check after each company entry, before moving on:**

- Strip the company name from the `gtm_thesis`. Could you swap any other company's name in and have it still make sense? If yes, rewrite.
- Are all numeric claims sourced in ROW_SOURCES? If not, add citations or remove the claim.
- Does the `subtitle` follow the *[what] — [why-they-fit], [relationship]* pattern? If not, rewrite.
- Does the entry feel generic? If you couldn't tell it apart from another company in the same segment, the patterns aren't landing.

If a company entry fails any check, fix before adding the next.

**Validate JS parses** after each batch of ~5 companies (or after each one if you're being careful):

```bash
node -e "require('./data.js')" 2>&1 || echo "PARSE ERROR — fix before continuing"
# Alternative if data.js doesn't have module.exports:
node --check data.js && echo "OK"
```

A broken data.js means the dashboard won't render. Catch errors early.

**Commit at the end of Phase 7:**

```bash
git add data.js
git commit -m "Phase 7: data.js populated (N companies across pipeline/mid-market/enterprise)"
```

### Phase 8: Customize index.html

The template HTML carries Valar's labels and structure as defaults — almost everything needs swapping for a non-Valar founder. Start by copying the template:

```bash
cp template/index.html index.html
```

Now edit `index.html` directly. Walk this checklist:

1. Replace `{{PRODUCT_NAME}}` with the founder's product name throughout (~5-7 occurrences depending on template version).
2. Replace `{{PRODUCT_SLUG}}` with a lowercase-dash slug for the Slack channel reference.
3. In `<script>`, find `buildScoreTips()` — it has hardcoded label strings ("Inference Pain", "Data Residency", "Buying Trigger", "Hiring"). The first two are Valar-specific; replace with your axis names from Phase 5. The latter two ("Buying Trigger" → conceptually "Opportunity Score" axis, and "Hiring") are mandatory axes — keep their function but rename labels if you want different display names.
4. In `dotsRow()` call sites (search for `dotsRow('Inference Pain'`, `dotsRow('Data Residency'`), update labels to match Phase 5 axis names. There are typically ~4 of these calls per render.
5. Update the `labels` object inside the detail-view tooltip code (search for `labels = {pain:'Inference Pain'...}` to update those references).
6. In `computeJobSignal()`, replace the hiring keyword regex per Phase 5 vertical-specific keywords.
7. Update tab labels in `<nav class="page-tabs">` if your segment IDs differ from default.
8. Sanity check `.tag.brand` CSS class is intact. All `c: 'brand'` values in your data.js render with founder accent.

**Validate after edits:**

```bash
# Check no leftover Valar-specific strings remain (should return 0 lines for non-Valar founders)
grep -E "Inference Pain|Data Residency|\\{\\{PRODUCT_NAME\\}\\}|\\{\\{PRODUCT_SLUG\\}\\}" index.html
# If any results, those are missed replacements — fix.

# Check the file is reasonably-sized HTML (not corrupted)
wc -l index.html
```

**Commit:**

```bash
git add index.html
git commit -m "Phase 8: index.html customized with [vertical] axis labels and branding"
```

**Manual verification:** open `index.html` in a browser. Walk through one company card end-to-end. Any string you see in the UI that says "Inference Pain" or "Data Residency" is a missed replacement.

### Phase 9: Write BUILD_NOTES.md

Following `template/BUILD_NOTES_TEMPLATE.md`, write `./BUILD_NOTES.md` at the repo root. Document:

- Signal axis labels chosen + justification
- Segments + reasoning
- Sections per company + reasoning
- Features kept/dropped + reasoning
- Notable copy decisions
- Companies featured & rationale (especially if Webset returned more than were used)
- **Webset details: ID, query, criteria, enrichment fields, completion time** (so user can refresh later)
- **Reproducibility note:** point to `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json` as the saved intermediates
- Data quality notes — strong / thin / missing
- Open questions for the user
- Recommended next steps before the demo

Be honest about uncertainty. Flag anything that needs human judgment.

**Commit:**

```bash
git add BUILD_NOTES.md
git commit -m "Phase 9: BUILD_NOTES.md documenting structural decisions and Webset metadata"
```

### Phase 10: Self-check + deliver

Run automated checks where possible:

```bash
# 1. data.js parses (catches unbalanced quotes/brackets)
node --check data.js && echo "✓ data.js parses" || echo "✗ data.js PARSE ERROR"

# 2. No leftover template placeholders or Valar-specific strings
grep -nE "\\{\\{[A-Z_]+\\}\\}" data.js index.html && echo "✗ Placeholders remaining" || echo "✓ No placeholders"
grep -n "Inference Pain\\|Data Residency" index.html | grep -v "^[[:space:]]*\\(//\\|\\*\\)" && echo "✗ Valar axis labels still in HTML" || echo "✓ Axis labels customized"
# (skip the second check if the founder genuinely is Valar)

# 3. Required output files all exist
for f in CONTEXT.md data.js index.html BUILD_NOTES.md; do
  test -f "$f" && echo "✓ $f" || echo "✗ MISSING: $f"
done

# 4. Saved intermediates exist (reproducibility)
for f in webset-spec.json webset-response.json lovelace-contacts.json; do
  test -f "$f" && echo "✓ $f saved" || echo "✗ MISSING intermediate: $f"
done

# 5. Git history is clean (per-phase commits visible)
git log --oneline | head -15
```

Then walk this manual checklist:

- [ ] Every company has a real `gtm_thesis`, not a placeholder
- [ ] Every section has specific numbers/quotes/sources, not generic strings or "[X]"
- [ ] Signal axis labels are vertical-appropriate (NOT "Inference Pain" unless this is Valar)
- [ ] At least one `tags` chip per company has `c: 'brand'`
- [ ] CONTACT_MAP keys exactly match `SEGMENTS[].companies[].name` (character-for-character including parentheses)
- [ ] Hiring keyword regex reflects this vertical
- [ ] `gtm_thesis` paragraphs sound like the founder, not like Claude (spot-check 3 random entries)
- [ ] At least one verbatim founder quote (from `inputs/`) appears in CONTEXT.md or `gtm_thesis` entries
- [ ] Webset ID and saved JSON files are documented in BUILD_NOTES.md

Fix any failures before delivery.

**Final commit if anything was fixed:**

```bash
git add -u
git commit -m "Phase 10: self-check fixes"
```

**Deliver — present the repo to the user:**

```bash
# Confirm repo state
pwd
ls -la
git log --oneline
```

Closing message to user:

> "Dashboard build complete in `~/fdi/<founder-slug>/`. Open `index.html` in your browser to preview locally — no server needed. Read `BUILD_NOTES.md` first; it documents the structural choices and any open questions.
>
> The 10% of work you still own: pick the 5 companies to lead with in the demo, fine-tune narrative for [founder]'s voice, and spot-check the [N flagged data points] from BUILD_NOTES.md.
>
> The repo is initialized with per-phase commits — `git log --oneline` shows the build history. The GitHub remote was set up in Phase 0; run `git push` to send your changes. From there, connect the repo to Vercel for a hosted preview URL. To refresh data later, the saved Webset ID is in BUILD_NOTES.md and `webset-spec.json` is committed for reproducibility."

---

## Tips

**Claude Code-specific:**

- **Stay grounded in `inputs/` throughout the build.** The skill repeats "re-read the raw docs" at multiple phases for a reason — CONTEXT.md is a compression. Phase 6a especially: when you go to draft the Webset spec, run `cat inputs/granola-*.txt` again. The texture lost in CONTEXT.md is what disambiguates the buyer profile.
- **Commit per phase.** Real version history matters. If Phase 7 produces a bad data.js, `git revert` is one command. If you batch-commit at the end, you've lost the ability to roll back to a clean intermediate state.
- **Use the saved JSON files as your source of truth in Phase 7+.** Don't re-call MCP tools when populating data.js — read from `webset-response.json` and `lovelace-contacts.json`. This makes Phase 7 reproducible: you can re-run it without re-firing the Webset.
- **Validate JS parses incrementally.** `node --check data.js` after every ~5 companies catches syntax errors while context is small. A malformed data.js means the dashboard won't render — and the error gets harder to find the more you've added.
- **Skip Granola exports if Granola MCP is connected.** Instead of asking the user to drop transcripts in `inputs/`, query Granola directly. Same data, less friction.

**Skill-wide:**

- **Resist asking too early.** Phase 1 (read everything first) is non-negotiable. If you skip ahead, you make the user repeat what's already in the docs.
- **Push hard on the CRITICAL questions** — lookalikes, exclusions, the wow signal, founder quotes. These are what separate a generic dashboard from one that lands.
- **Buyer ≠ founder.** The single most common Webset failure mode is searching for the founder's industry instead of the buyer's industry. For an inference fabric founder, the buyer is a bank or a healthcare company, not another AI infrastructure vendor. Phase 6a's two-paragraph buyer profile exists specifically to prevent this. A real test on May 4 with a well-intentioned generic spec returned 3 of 5 vendor/peer matches; the rewrite fixes this by demanding the buyer profile be written explicitly before any spec.
- **Long enrichment descriptions are not optional.** Every enrichment description should be a 50-100 word paragraph specifying signal, sources, format, and exclusions. "Documented pain points" returns generic. "Documented evidence that this company runs production AI inference at scale, citing engineering blogs or third-party press; if the company appears to BE an inference vendor rather than a consumer, mark NOT APPLICABLE" returns specific signal.
- **The pre-Webset checkpoint (6f) is the cheapest place to course-correct.** Always pause there for user confirmation before firing a Webset. Fixing a spec costs zero. Re-running a Webset costs $5-10 and 10 minutes.
- **Always include the EXCLUDE clause in searchQuery.** Webset agents respect explicit negative phrasing. "Companies similar to Capital One" is good; "Companies similar to Capital One. EXCLUDE: AI infrastructure vendors, AI cloud providers, GPU resellers" is much better.
- **Name actual competitors in EXCLUDE, don't just name categories.** Generic exclusions ("AI vendors") leak. Named exclusions ("Fireworks, Together, Baseten, Modal") hold. Pull the explicit competitor list from CONTEXT.md's Competitive Landscape table — every named player goes into EXCLUDE.
- **Don't fight the async.** If the Webset takes 8 minutes, take 8 minutes. Don't try to populate data.js with placeholders during the wait — you'll do double work.
- **Founder voice in `gtm_thesis` is the leverage point on copy.** Splice verbatim quotes from `inputs/` or CONTEXT.md.
- **The wow signal should be visible from any angle.** Surface it in `gtm_thesis`, in `signals[]`, and in tags.
- **The most common second failure mode is over-anchoring on Valar.** If your output sounds like Valar with names changed, you've under-delivered. Vertical-specific signal axes (beyond the mandatory Hiring + Opportunity) are required, not optional.
- **Source every numeric claim.** Empty `ROW_SOURCES` is fine; wrong `ROW_SOURCES` is worse than nothing. Webset returns source URLs inline within text fields (pattern: `fact text | URL / fact text | URL`); parse these out into ROW_SOURCES rather than discarding them.
- **Don't fabricate data the Webset didn't return.** Mark fields as "needs verification" in `BUILD_NOTES.md` instead of inventing numbers.
- **Watch for 402 errors mid-build.** If credits run out partway through, finish what you can, mark the rest as needing manual fill, and flag in BUILD_NOTES.md. Don't silently skip companies.
- **The 0–5 axis math is generic — leave it alone.** Only the labels and axis count change between projects.
- **Save the Webset ID and webset-spec.json.** Future iterations can refresh data without rebuilding the query, and Exa monitors can keep the list fresh on a schedule.
- **Reference the gold standard:** `github.com/alexg207/valar-fdi`. When in doubt about tone or density, that's the bar.
