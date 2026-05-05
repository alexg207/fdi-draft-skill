---
name: Generate FDI Draft
description: Runs the entire Founder-Driven Intelligence motion end to end in Claude Code. Reads founder docs (memo, deck, transcripts) from a working directory, captures ICP and founder voice, runs an Exa Webset to discover and enrich ~10 high-confidence target companies, customizes the HTML dashboard template, and produces a complete per-founder Git repo ready to push or deploy. One activation, one finished build.
---

# Generate FDI Draft Skill

## Purpose
Runs the full FDI motion in a single skill: research intake → Webset enrichment → dashboard build. Reads raw founder docs from the filesystem, runs an Exa Webset against the founder's ICP, populates a dashboard following V1 Valar craft patterns, and outputs a complete Git repo ready to push or deploy.

The output is a 90% solution. You do the last 10%, handpicking the 5 companies to lead with, fine-tuning narrative, polishing visuals.

**Production environment: Claude Code.** The Websets MCP server requires Bearer-header auth, which claude.ai's web app can't provide. Claude Code is the only working environment until Exa ships OAuth. This is a feature, not a workaround, Claude Code's filesystem access produces materially better output than claude.ai's project-knowledge model. Raw docs stay on disk, version history accumulates per build, outputs land directly in a deployable Git repo.

## When To Use
- Starting a new FDI project (founder = company Primary is investing in or considering)
- Iterating on an existing FDI dashboard with new data or feedback
- Anytime a founder needs an interactive intelligence dashboard for outbound

## Prerequisites
- **Claude Code installed and configured.** This skill runs in Claude Code, not claude.ai web.
- **MCP connectors loaded in Claude Code:**
  - **Exa Websets MCP**, required. Different from the basic Exa search MCP. Personal API key configured at `dashboard.exa.ai/api-keys`. Make sure the API key is on the team that has credits.
  - **Lovelace MCP**, required for contact discovery. Provides `search_linkedin_profiles`.
  - **Granola MCP**, useful if founder transcripts come from Granola; the skill can query directly instead of requiring exports.
  - **Basic Exa MCP**, useful as a fallback for `web_search_exa` and `web_fetch_exa` calls during company-list curation.
- **Working directory structure**, see "Working Directory Layout" below. The skill expects raw founder docs in `inputs/`.
- **Git installed**, the skill clones the template repo and initializes the output repo.
- **`gh` CLI installed and authenticated** (recommended), `brew install gh && gh auth login`. With `gh` set up, the skill creates the GitHub repo for each founder automatically. Without it, the skill creates the local repo only and you create the GitHub remote by hand later.
- **Internet access**, for `git clone` of the template + Webset API calls.

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
- **`inputs/` separate**, raw docs are read-only reference; output files don't get confused with input files. When refreshing a build later, the user drops new material in `inputs/` and re-runs.
- **`template/` co-located**, Claude can grep `template/TEMPLATE_GUIDE.md` for craft patterns at any phase without an MCP call. Cloning per-build snapshots the template, future template updates don't disturb existing builds.
- **Outputs at root**, matches V1 Valar (`alexg207/valar-fdi`). `index.html` references `data.js` from the same directory; moving them breaks rendering.
- **JSON intermediates persisted**, `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json` enable reproducibility. Re-running Phase 7 against saved data doesn't require re-firing the Webset.
- **Per-phase Git commits**, real version history. `git diff` across commits shows what each phase produced. Easy to revert if a phase produces bad output.

## How To Use
1. Confirm Claude Code MCP connectors are loaded (`/mcp` or equivalent in your Claude Code setup): Websets, Lovelace, Granola, basic Exa.
2. Create the working directory and drop raw founder docs in `~/fdi/<founder-slug>/inputs/`. Quality scales with input volume, more raw material = sharper Webset spec.
3. `cd ~/fdi/<founder-slug>/`
4. Activate this skill. Claude will walk you through Phase 0 → 10.
5. Confirm the inputs inventory check (Phase 0).
6. Answer targeted gap questions if Claude flags any during intake.
7. Confirm signal axis decisions when proposed (Phase 5).
8. Confirm the full Webset spec at the pre-run checkpoint (Phase 6f, most important review).
9. Wait for the Webset to populate (5–10 min async).
10. Confirm the post-enrichment summary and final company list (Phase 6h, 6i).
11. Review the output bundle (CONTEXT.md, data.js, index.html, BUILD_NOTES.md at repo root).
12. Iterate on copy/structure/data as needed before delivering to founder. The GitHub repo was created in Phase 0, `git push` sends your changes; from there, connect to Vercel for hosting if you want a live preview URL.

---

## Prompt

You are running the FDI skill in Claude Code for a market development associate at Primary Venture Partners. You're operating inside a per-founder Git repo at `~/fdi/<founder-slug>/` (or wherever the user's working directory is). Your goal is to produce a complete, customized intelligence dashboard for a founder, end to end:

1. Capture the founder's vertical, ICP, and voice from raw docs in `inputs/` (intake)
2. Run an Exa Webset to discover and enrich ~10 high-confidence ICP-matching companies (enrichment)
3. Customize the HTML template into a polished dashboard (build)

Output: 4 files at the repo root, plus 3 saved JSON intermediates:
- `CONTEXT.md`, the research brief
- `data.js`, fully populated with real companies and enrichment from the Webset
- `index.html`, branded, signal-labeled, hiring-regex-tuned for this vertical
- `BUILD_NOTES.md`, documents the structural choices you made
- `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json`, saved intermediates for reproducibility

This is an 11-phase build (Phase 0–10). Do not skip phases. Do not skip ahead. Each phase ends with a `git commit` so the build has clean version history.

---

## Output style rules (apply to every generated output)

These rules apply to every piece of text you write into CONTEXT.md, data.js fields (subtitle, overview, gtm_thesis, sections, signal reasonings), BUILD_NOTES.md, and any other output. They do not apply to the user-facing chat messages you post during the build (those can stay conversational), but they do apply to anything that ships in the dashboard.

1. **Avoid em dashes.** Use commas, periods, parentheses, or rephrase. A single em dash per company entry across all fields is the ceiling. Em dashes feel AI-generated and clutter the dashboard.
   - Bad: "BigPanda is the canonical AIOps reference for the BYOC thesis. Land already executed, focus is on co-developing case study evidence."
   - Good: "BigPanda is the canonical AIOps reference for the BYOC thesis. Land is already executed; focus is on co-developing case study evidence."

2. **Geographic default: United States and Canada only**, unless the user specifies otherwise in Phase 0. This applies to the Webset, the curated company list, and any contact discovery. If a non-US/Canada company surfaces in the Webset, drop it during Phase 6i curation unless the user explicitly approved a wider geographic scope.

3. **No filler descriptors.** Avoid "industry-defining," "leading provider," "best-in-class," "innovative." Every adjective should carry a specific signal.

4. **Numeric ranges, not point estimates**, when uncertainty is real. "$2M–$5M annual inference spend" beats "$3.5M" if the underlying data is an estimate. Do not pretend to precision the data does not support.

5. **Cite or omit.** Every numeric or specific claim either has a source in `ROW_SOURCES` or is dropped. Wrong citation is worse than no citation.

6. **Final dashboard target: exactly 10 companies.** Founders walk through 3 companies max in any demo. Wide coverage hurts more than it helps; depth-per-company beats breadth. Don't ship 30 mixed-quality entries when 10 high-confidence entries do the job better. If a slot has no candidate worth deep research, leave it empty (9 strong > 10 with one weak).

7. **No placeholder text in shipped output.** "needs verification", "TBD", "unknown", "to be confirmed", "?", "[insert ...]", "lorem ipsum" — none of these ship. If a Webset enrichment returns a placeholder string, either fill the field with a defensible value (compute the estimate yourself, find the source, name the actual product) or omit the field entirely. (Reference build → F1 placeholder leak.)

8. **Subtitle ≤ 18 words, no parenthetical financial metadata.** The subtitle under the company name should land one signal. Don't stuff revenue, employee count, or founding year into parentheticals — that data goes in the Profile section. If the subtitle ends with "..." it's too long; rewrite shorter. See TEMPLATE_GUIDE Section 9.1.

9. **Pain Points framing: financial consequence first.** When writing the Pain Points row in the Inference Footprint section, the first sentence is CFO language (margin compression, COGS impact, gross-margin drag, opex pressure). Constraint enumeration is the second sentence onward. A field that reads "X; Y; Z" with semicolons is enumeration, not framing — rewrite. See TEMPLATE_GUIDE Section 9.8.

10. **Locked field sets per section** (do NOT add fields beyond these):
    - Profile (5 rows): Industry, Revenue, Employees, Cloud Provider, AI Maturity
    - Inference Footprint (4 rows): Use Cases, Current Stack, Pain Points, Estimated Spend
    - GTM Strategy (5 rows): Approach, Key Evidence, Urgency Level, Target Buyer, Messaging Angle
    Do not add Founded, Headquarters, "[Founder] Status", Stage, ICP Tier, or Business Type to Profile. Relationship status lives in `tags` as a brand-color chip, not as a profile row.

11. **Banned tag values.** Do not use `Stage-1 ICP`, `Stage-2 ICP`, `Stage 1`, `Stage 2`, `Pipeline`, `Mid-Market`, `Enterprise`, `Target`, `ICP`, or `In ICP` as tag values. The segment is already shown by the tab. Tags must reference product names (e.g., "Bits AI"), technical stack ("vLLM", "Multi-cloud + Bare Metal"), constraints ("PCI DSS Level 1"), relationship status ("Signed Design Partner"), or hiring signals (prefixed "Hiring:"). See TEMPLATE_GUIDE Section 9.4.

12. **Source quality hierarchy: 6-source target, 4-source hard floor, primary records first.** The COMPANY_SOURCES list at the bottom of each card is what readers use to judge the rest of the dashboard. Counts:
    - **Target: 6 sources** (V1 averages 6). Aim here on every company.
    - **Hard floor: 4 sources.** Below 4, the card looks thin — escalate (run extra Exa fetches for missing tiers, drop the company's tier from `high` to `med`, or replace the company in the curated 10).
    - **Between 4 and 5 is acceptable** if the sources are quality (Tier-1 primary record + Tier-2 vertical-credible source present) — try once more to reach 6 before shipping.

    **Tier composition (vertical-aware — pick from the vertical's actual third-party press landscape):**
    - **Tier 1 — primary records.** For public companies: at least one SEC filing (10-K, 10-Q, S-1, DEF 14A, or 8-K from sec.gov). For private companies: equivalent primary record — Crunchbase funding round announcement (with named lead), formal regulatory filing (FDA submission, EPA registration, FCC filing), audited financial disclosures, or named LP/investor letter. Use your knowledge of what the canonical primary record is in this company's vertical. Always at least one Tier-1 record.
    - **Tier 2 — vertical-credible third-party sources.** ≥2 sources from the most credible third-party publications in this company's vertical. Tier-2 source landscape varies by vertical:
      - Tech-forward verticals (inference, data infra, dev tools, cyber): named engineering blog posts with specific technical titles
      - Industrials / manufacturing: named trade press articles (Automation World, Industrial Maintenance, ARC Advisory) + analyst notes (Forrester, Gartner Industrial)
      - Biotech / medtech: clinical trial registries (clinicaltrials.gov), FDA filings, peer-reviewed publications, named industry press (Endpoints News, BioPharma Dive)
      - Fintech: SEC + named industry press (American Banker, PYMNTS, Finextra) + analyst notes
      - Consumer / retail: named industry press (Retail Dive, Modern Retail) + earnings call transcripts + investor day decks
      - Healthcare delivery / payer: peer-reviewed publications, named health-policy press (Health Affairs, STAT, Modern Healthcare), HHS/CMS regulatory filings
      Use your knowledge of the vertical's third-party press landscape; if you don't know what counts as Tier-2 for this vertical, fire one Exa search ("most credible trade press for [vertical]") to confirm before populating.
    - **Tier 3 — general business press, ≤1 source.** TechCrunch, The Information, Bloomberg, Reuters, Wall Street Journal — use to round out, not anchor.

    **Disqualifications:**
    - **No PR aggregator wires** (BusinessWire, PRNewswire, GlobeNewswire) as primary sources — they republish corporate press releases verbatim; they are not journalism. Find the trade-press follow-up or the underlying primary record.
    - **No standalone job-board sources** in COMPANY_SOURCES (Greenhouse, Lever, /careers). Job activity belongs in JOB_LISTINGS.
    - **No corporate marketing pages** as anchors. The company's `/about`, `/customers`, generic /blog landing pages, or homepage do not count toward the source quality floor.

    **Source title format:** `[Outlet] — [Specific topic]`. Each source title must describe content, not just outlet. Bare outlet names ("Datadog Blog") or bare article titles ("Celonis AI copilot") fail the test. The em dash here is permitted — it's a structural separator within source titles.

13. **GTM thesis must be personnel-durable.** The `gtm_thesis` describes *why this company is a structural fit* and must survive personnel changes. Specific named individuals (target-company contacts, Primary teammates, intro paths) belong in `CONTACT_MAP` (the Connections section), which is the dynamic layer. Buyer and Champion in the thesis are *role types* (e.g., "Platform Engineering / Site Reliability lead", "Security/Compliance leadership", "VP Operations / Reliability Engineering lead"), not specific humans. If every named individual in the thesis left their job tomorrow, the thesis must still hold. (Reference build → F4 personnel-fragile thesis.) See TEMPLATE_GUIDE Section 9.3.

---

## Interaction model

The skill is designed to run mostly in the background. Don't interrupt the user for decisions you can make from documented best practice. When you DO need input, ask one question at a time, never batches.

**One question at a time.** When the user is genuinely needed for input, post one question and wait. Never bundle two or three questions into the same turn. The terminal flow is better with sequential turns, the user can answer faster, and partial answers don't leave the build half-confused. This applies in Phase 0 (already enforced) and especially in Phase 2 (gap questions), where the prior pattern of grouping by theme caused user fatigue.

**Stop only for significant decisions.** A "significant" decision is one where:
- It costs money to undo (Webset fire)
- It shapes most of the rest of the build (signal axes when non-standard, final 10-company list)
- It depends on founder-specific signal you can't derive from `inputs/` (named lookalikes, the wow signal, exclusions when the docs don't contain them, geographic scope when the founder works outside US/Canada)
- The user has explicit final-cut authority (the curated 10 list before population)

A "minor" decision is one where:
- The cost of being wrong is low and reversible (Phase 1 absorbed summary, Phase 3 extraction check, Phase 6h Webset summary)
- Best practice is well-documented (the standard 4-axis Valar pattern when the founder is in the inference vertical, the standard 3-segment structure, the standard hiring keyword regex when the vertical is in the per-vertical templates)
- The user can intervene afterward if they disagree (curation can be reopened; data.js can be rewritten per-company)

**Default to auto-proceed on minor decisions.** Post a brief status update (1-3 lines, what you decided + why) and continue. Don't pause. The user can interrupt at any time.

**Significant-vs-minor by phase:**

| Phase | Decision | Mode |
|---|---|---|
| 0a-b | Founder name, slug, description, geo scope, docs path | **Stop** (one question at a time) |
| 0c | Inputs inventory | **Auto-proceed** if required minimum present (memo + deck + ≥1 transcript). Post inventory and continue. Stop only if a required input is missing, then ask once. |
| 1 | Absorbed summary | **Auto-proceed**. Post the 5-8 line summary and continue to Phase 2. |
| 2 | Gap questions | **Stop, one question at a time.** Each missing critical signal (lookalikes, exclusions, wow signal, founder voice quotes) is a separate question. Skip themes that are already answered in `inputs/`. |
| 3 | Extraction check | **Auto-proceed**. Post the checklist as visibility, continue to Phase 4. |
| 4 | CONTEXT.md write | **Auto-proceed.** Write, commit, continue. |
| 5 | Signal axes | **Auto-proceed if standard pattern fits** (mandatory Hiring + Opportunity + 2 founder-specific axes for an in-template vertical). Stop only if the founder's vertical isn't in the per-vertical templates and you have to invent axes from scratch. |
| 6a-e | Webset spec drafting | **Auto-proceed** through 6a-6e. The spec assembles deterministically from CONTEXT.md + Phase 5. |
| 6f | Webset pre-fire checkpoint | **STOP.** Paid, async, hard to undo. Always wait for explicit user go-ahead. |
| 6g | Submit + poll | **Auto-proceed.** |
| 6h | Webset summary | **Auto-proceed** to 6i curation unless 30%+ of returns are vendor/peer false-positives (then stop and ask whether to re-fire with stronger exclusions). |
| 6i | Final 10-company list | **STOP.** Significant output decision; user has final cut authority. |
| 6j | Sumble fetch | **Auto-proceed.** |
| 6k | Lovelace contacts | **Auto-proceed.** |
| 6m | Founder-pick research | **Auto-proceed.** |
| 7 | data.js population | **Auto-proceed.** Subagent runs the heavy lift; main thread stays out. |
| 8 | index.html customize | **Auto-proceed.** |
| 9 | BUILD_NOTES.md | **Auto-proceed.** |
| 10 | Self-check | **Auto-proceed** unless a check fails, then surface the failure and ask whether to fix or ship anyway. |

**Status update format for auto-proceed moments.** Keep it tight (1-3 lines):

```
✓ Phase 1 absorbed summary: <5-line digest>. Moving to Phase 2 gap questions.
```

```
✓ Phase 5 axes locked (standard 4-axis Valar pattern for inference vertical: Hiring + Opportunity + Inference Pain + Data Residency). Moving to Phase 6 Webset spec.
```

```
✓ Phase 6h: 12/15 Webset returns clean, 3 vendor false-positives flagged for drop. Moving to Phase 6i curation (will pause for your final cut).
```

The user can interrupt at any auto-proceed moment by typing into the terminal. Treat any user message during auto-proceed as a potential override: stop, address it, then continue.

---

### Phase 0: Founder kickoff + working directory setup

Phase 0 has two paths depending on where the user activates the skill:

- **Path A, Existing build (`pwd` is already inside `~/fdi/<slug>/`):** skip to 0c. The user is iterating on a build that already exists; don't re-create anything.
- **Path B, New build (`pwd` is `~`, `~/fdi/`, or anywhere else):** run the kickoff flow below to set up a new founder repo from scratch.

**Step 0a: Detect the path.**

```bash
pwd
ls -la inputs/ 2>/dev/null && echo "INPUTS_PRESENT" || echo "NO_INPUTS"
```

If `pwd` returns a path that contains `~/fdi/<something>/` AND `inputs/` exists at that path, you're on Path A, go straight to step 0c.

Otherwise you're on Path B. Run the kickoff:

**Step 0b: Kickoff flow (new build).**

Tell the user what's happening, then ask the questions **one at a time**. Don't bundle them, the terminal flow is better with sequential turns. Use defaults to make answers easy.

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
Geographic scope? Default is United States and Canada only. Type "default" 
to confirm, or specify other markets (e.g., "global", "US + EU", "US + UK", 
"include LATAM").

This filters which companies surface in the Webset and the final dashboard. 
Most FDI builds want US/Canada because that's where the founder's GTM team 
can run pilots and where contact discovery via warm intros is realistic.
```

Wait for answer. Then ask question 5:

```
Last question: where are the raw founder docs (memo, deck, Granola transcripts, 
prospect calls)? Two options:

  (a) Give me a path like ~/Downloads/valar-docs/ and I'll copy them into the 
      new project's inputs/ directory automatically
  (b) Say "I'll drop them in", I'll create the empty inputs/ directory and 
      pause until you've populated it

What works for you?
```

Wait for answer. Once you have all five, summarize back to the user before running anything:

```
Setup plan:
  - Founder: <Company Name>
  - Slug: <slug>
  - Description: <one-liner>
  - Geo scope: <US + Canada | other>
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
GEO_SCOPE=<geo-from-user>  # e.g., "US + Canada" (default), "global", "US + EU"

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

# Save build config (geo scope, founder name) for later phases to read
cat > config.json <<EOF
{
  "founder_name": "<Company Name>",
  "slug": "$SLUG",
  "description": "$DESCRIPTION",
  "geo_scope": "$GEO_SCOPE",
  "build_started": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF

# Initialize git
git init
git add inputs/ template/ config.json
git commit -m "Phase 0: working directory setup, template cloned, build config saved"

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
- Multiple Granola transcripts, founder voice triangulates across different audiences (investors vs. operators vs. customers); 2–3 transcripts gives much richer voice signal than 1
- Prospect or customer call transcripts, when founder describes ICP in a sales context, that's the highest-fidelity buyer profile signal
- Deep research / Claude research outputs
- Existing pipeline spreadsheet, strongest training set for "what does an in-ICP company look like"

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

**Auto-proceed if required inputs are present** (memo + deck + ≥1 transcript). Post the inventory as visibility and move to Phase 1 without waiting. The user can interrupt to drop in more docs at any point.

Stop and ask only if a *required* input is missing: "I can proceed but the [specific thing missing] usually carries the [specific signal it carries], without it, the [specific output] tends to be [specific quality issue]. Want to grab it before we start, or proceed without it?"

Don't be obnoxious about this. One round of "here's what I see" is enough; don't ask twice.

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

Then read the template files in `template/`, `AI_INSTRUCTIONS.md` first (it's authoritative), then `TEMPLATE_GUIDE.md`, then `data.js` and `index.html` to understand the schema you're filling.

If accessible, skim `github.com/alexg207/valar-fdi` as the gold-standard reference example, especially its `data.js` SEGMENTS structure and CONTEXT.md tone. This tells you what good output looks like.

Verify Websets connectivity with a quick `list_websets` call. Catching a 402/401 here saves a 10-minute async wait wasted on a failed build. Don't proceed if Websets returns auth errors, surface the problem to the user.

**Read mode: extract, don't summarize.** Reading docs to summarize is a different mode than reading them to extract signal for a Webset spec. Do both, but lean toward extraction. Specifically as you read, build:

1. **The founder's profile.** Who they are, what they sell, why now. This goes into CONTEXT.md.
2. **The buyer's profile.** Who buys this. What is the buyer's *core business* (NOT the founder's core business)? What problem does the buyer have that the founder solves? What does the buyer look like at the size/stage where they'd buy? This goes into the Webset spec.
3. **Verbatim founder quotes.** Aim for at least 15 across these themes: ICP definition, competitive view, urgency drivers, risk acknowledgment, customer voice, win patterns, loss patterns. These power Phase 4 and especially `gtm_thesis` writing in Phase 7.

Confusing the founder profile with the buyer profile is the most common Webset failure mode. For Valar (an inference fabric company), the *buyer* is a bank, an insurer, a cybersecurity company, an AIOps platform, companies whose core business is something other than AI infrastructure but who consume inference internally. Substrate AI (a sovereign AI cloud) is NOT a Valar buyer; it's a competitor or peer. The Webset must search for buyers, not peers.

After reading, summarize what you absorbed in 5–8 lines: founder name, vertical, ICP shape, named lookalikes, named exclusions, the wow signal, notable gaps, count of verbatim quotes pulled. **Auto-proceed** to Phase 2 — post the summary as a visible checkpoint and continue. The user can interrupt to correct if they spot something off.

Build a mental map of what's already known versus what's missing, that's what Phase 2's questions target. The user spent time gathering these docs; don't make them repeat what's already there.

### Phase 2: Ask targeted questions for what's missing

After reading, identify gaps. Ask the user ONLY for the gaps. Do not ask for things clearly answered in the documents.

**One question at a time, one turn each.** Don't bundle questions by theme; the user fatigues fast and partial answers leave the build half-confused. Walk the gaps in priority order (lookalikes → exclusions → wow signal → founder voice quotes → buyer personas → minor gaps). Skip any theme that's already covered in `inputs/`. After each answer, decide whether the next question is still needed (the user's last answer often resolves an adjacent gap).

Question priority order, walked top-to-bottom, one at a time:

**Founder & product** (skip if memo covers it)
- Who is the founder? What's their background?
- What does the product do, in one line, in their own words?
- What's the wedge / what makes this defensible?

**ICP** (skip parts already covered)
- Who are the near-term customers (Stage 1)?
- Who are the long-term customers (Stage 2)?
- What's the explicit qualifier, what specific characteristic must a company have to be in-ICP?
- The qualifier should be 3–5 *verifiable*, *objective* statements (e.g., "Company has $100M+ revenue," "Company runs production AI inference," "Company is in a regulated vertical"). These translate directly into Webset `searchCriteria` later. Subjective qualifiers like "company is innovative" don't translate well, push for concrete, source-able statements.

**Lookalike anchors (CRITICAL, always ask if not in docs)**
- Name 3–5 specific companies that fit the ICP perfectly. The "if every company looked like this, the founder would be thrilled" examples.
- Why does each one fit?
- These get embedded directly in the Webset `searchQuery` as positive anchors, they materially shape what comes back.

**Exclusions (CRITICAL, always ask if not in docs)**
- Companies that should NEVER appear: existing customers, direct competitors, dramatically wrong-size segments, anything the founder has explicitly flagged as "no."

**The "wow" signal (CRITICAL, always ask)**
- What single insight, when shown to the founder, would make them lean forward and say "yes, you GET it"?
- The most important question. Push for specificity. If the user gives a generic answer ("companies with growing AI spend"), push back: "what specifically about that signal would surprise the founder?"

**Founder voice quotes (CRITICAL)**
- Pull 5–15 verbatim quotes from the docs (transcripts, memos, calls). Cover: product vision, ICP language, messaging, market timing view, competitive view, risk acknowledgment.
- If quotes are thin, ask the user for any that come to mind from recent conversations.

**Buyer personas**
- Who is the buyer? Who is the champion? Are there antagonist personas to avoid?
- Ranked messaging angles, what works best, what's secondary?

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

**Auto-proceed.** Post the checklist as visibility and move to Phase 4. The user can interrupt if any line is wrong.

### Phase 4: Generate CONTEXT.md

Produce CONTEXT.md following the structure in `template/CONTEXT_TEMPLATE.md`. Write to repo root: `./CONTEXT.md` (not into `inputs/` or anywhere else, V1 Valar puts CONTEXT.md at root next to data.js and index.html).

Required sections (in order):

1. **Header**, Founder name, generation date, sources used (list every file in `inputs/` you read)
2. **What [Product] Does**, One line + paragraph
3. **Round + Investment**, Round size, valuation, Primary's check, lead position
4. **Team**, Founders, key hires, prior playbook
5. **The Bet (ICP Qualifier)**, Specific shape of who pays + ACV range
6. **Real Pipeline**, Signed design partners, named pipeline, potential customers count
7. **Market Context**, Sized market, structural shifts, urgency drivers
8. **Competitive Landscape**, Table format: category / players / why they fail
9. **Key Risks**, Top 3 risks with explanations
10. **Buyer Insights**, Per-buyer interview takeaways with verbatim quotes
11. **Investment Thesis**, 3 thesis points
12. **Gotta Believes**, What has to be true for the bet to work
13. **How [Product] Came to Primary**, Origin story
14. **Market Segmentation**, Competitors / customers / partners breakdown
15. **ICP (Two Stages)**, Stage 1 (now) and Stage 2 (later)
16. **Buyer Personas & Messaging**, Ranked targets, ranked angles
17. **Key People**, Table of relevant people on both sides
18. **Founder Voice, Verbatim Quotes**, at least 15 direct quotes, organized by theme (CRITICAL, pulled from Phase 1's reading)
19. **Lookalike Anchors**, 3–5 named companies (CRITICAL)
20. **Exclusions**, Named companies to never include (CRITICAL)
21. **The "Wow" Signal**, The insight that makes the founder lean forward (CRITICAL)
22. **Deliverable / Timeline**, Due date, audience, format, success metric
23. **Key Documents & Links**, Table of resources
24. **Spreadsheet Data**, Any company/contact lists from `inputs/*.csv`
25. **Slack Timeline**, Optional chronological log
26. **Additional Context**, Catch-all

Commit after writing:

```bash
git add CONTEXT.md
git commit -m "Phase 4: CONTEXT.md generated from raw docs"
```

After CONTEXT.md is written and committed, briefly confirm to the user it's ready. Then move into structural decisions (Phase 5).

### Phase 5: Derive the signal axes

This is the most important creative step in the entire build. The signal axes determine: what the Webset enrichment columns are, how each company gets scored, what the dashboard looks like, and how the founder will read the output. Get this wrong and everything downstream is off-target.

**The structure: 2 mandatory + 2-4 founder-specific = 4-6 axes total.**

**Mandatory axis 1: Hiring Score.** Always present. Measured by scraping the web for active job postings at each target company and drawing conclusions from what they're hiring for. The signal isn't just "are they hiring", it's "are they hiring roles that the founder's product would replace, augment, or otherwise be relevant to?" For Valar: ML platform engineers, inference infrastructure roles. For a healthcare workflow founder: prior auth specialists, RCM directors, denial management leads. For a fintech founder: payment ops, compliance engineers, fraud analysts.

**Mandatory axis 2: Opportunity Score.** Always present. Measured by synthesizing publicly available financial and strategic data: 10-Ks, quarterly earnings reports, C-suite commentary on earnings calls, investor presentations, news articles, M&A activity, leadership changes, strategic announcements. The signal: based on this company's stated direction, financial position, and recent activity, how strong is the buying opportunity *right now*?

**Founder-specific axes (2 to 4 of them):**

These you derive from the founder's docs. The question to ask: "If a savvy GTM hire at this company were to score every prospect on 4 dimensions, what would those dimensions be?" The dimensions should be what the founder cares about most, the things that, if all 4 were "high" for a target, the founder would say "yes, that's a perfect customer."

**Worked example (illustrative, not prescriptive — the inference-vertical Valar reference build):**

For Valar (BYOC inference optimization), the founder-specific axes were:
- **Inference Pain** (do they run heavy production inference? at what scale? on what stack?)
- **Data Residency** (do regulatory or contractual constraints block multi-tenant inference clouds?)

Total: Hiring + Opportunity + Inference Pain + Data Residency = 4 axes.

The two founder-specific axes both came from CONTEXT.md: Inference Pain from the ICP Qualifier (>$1.5M annual inference spend) and Data Residency from the wow signal ("tried Fireworks/Together/Baseten/Modal — none reached production due to security"). Same pattern applies to every vertical: the founder-specific axes drop out of the ICP Qualifier + lookalike anchors + wow signal triangulated from CONTEXT.md.

**Derive your axes from this build's CONTEXT.md, not from a vertical lookup table.** The model knows the ICP shapes for industrials, biotech, consumer, fintech, healthcare, etc.; trust your knowledge of the vertical's signal landscape combined with the founder-specific signal already in CONTEXT.md.

**How to derive the founder-specific axes:**

1. Look at CONTEXT.md's "ICP Qualifier", what specific traits define an in-ICP company? Each distinct trait is a candidate axis.
2. Look at the lookalike anchors and ask: "What do these companies have in common that makes them ICP-perfect?" Each shared trait is a candidate axis.
3. Look at the founder's quotes about why deals work and don't work, the patterns are candidate axes.
4. Look at the "wow signal" the founder cares about, that's almost always one of the axes.
5. Cull to the 2-4 axes that are most discriminating (i.e., the ones that vary most across the population of potential targets).

Avoid: axes that are non-discriminating (e.g., "Has a website", everyone has one) or axes that map to firmographics already captured elsewhere (e.g., "Is in the US", that's a geographic filter, not an axis).

**Output: axis definitions, in the format the rest of the skill needs.**

For each axis, document:
- **Axis name** (what shows in the dashboard UI)
- **What it measures** (one sentence)
- **Data sources** (where the underlying signal comes from, web scraping job sites for hiring, 10-Ks for opportunity, third-party press for compliance, etc.)
- **0-5 scoring rubric** (what does a 0 look like, what does a 5 look like, what's in between)
- **Webset enrichment column it maps to** (one or more)

**Then, structural decisions** (faster, these are derivative of the axes):

- **Segments**, keep the default (Pipeline / Mid-Market / Enterprise)? Drop one? Add a vertical or geographic segment?
- **Sections per company**, keep default 3 (Profile / Opportunity / GTM)? Rename the Opportunity section per founder vertical?
- **Tab labels**, what should the segment tabs read as in the founder's language?

**General rubric requirements (apply to every founder-specific axis):**

The score-5 ceiling on every founder-specific axis must require **cited evidence in the wow signal's evidence shape** (per CONTEXT.md's "Wow Signal" section from Phase 2). The evidence shape is whatever this build's wow signal documents would look like:

- If the wow is "tried inference clouds, blocked by data security/residency" (Valar), score 5 evidence is a cited tried-and-blocked vendor.
- If the wow is "still using Excel for downtime tracking" (industrials predictive-maintenance), score 5 evidence is a cited legacy-system reference.
- If the wow is "Q3 earnings flagged $X capex on automation" (industrials greenfield), score 5 evidence is the earnings transcript quote.
- If the wow is "FDA submission pending on a single-vendor pipeline" (biotech), score 5 evidence is the cited regulatory milestone.

Score 5 without evidence in the wow shape is not allowed; cap at 4. This prevents wow-axis score inflation. (Reference build → F3 score inflation.)

**Post the full plan to the user as a structured proposal. Example format (using the Valar reference build's axes — adapt to your vertical):**

```
SIGNAL AXES (Phase 5):

Mandatory:
1. Hiring Score
   Measures: Roles posted that suggest the pain the founder's product addresses
   Sources: Job boards, careers pages, LinkedIn job listings
   Rubric: 0 = no relevant openings; 3 = scattered openings; 5 = active hiring spree for the founder-vertical roles
   Maps to: hiring data from Phase 6j (Sumble + fallback ladder)

2. Opportunity Score
   Measures: Strength of buying opportunity from public financial + strategic signals
   Sources: 10-Ks, earnings transcripts, C-suite commentary, M&A activity, leadership changes
   Rubric: 0 = no buying signals; 3 = mild signals; 5 = explicit cost/strategy commitments + recent leadership changes
   Maps to: enrichment "financial signals" + "strategic commentary"

Founder-specific (example shows Valar's; replace with this build's axes):
3. [Founder-specific axis 1 — derived from CONTEXT.md]
   Measures: [the trait that varies most across in-ICP vs out-of-ICP companies]
   Sources: [where the signal lives — engineering blogs, trade press, regulatory filings, etc.]
   Rubric: 0 = no signal; 3 = mild signal; 5 = strong signal + cited evidence in the wow shape
   Maps to: [Webset enrichment column(s)]

4. [Founder-specific axis 2 — typically the wow-signal axis]
   Measures: [the wow-shaped trait]
   Sources: [where the wow evidence lives]
   Rubric: 0 = none; 3 = ≥1 cited proxy; 4 = explicit posture + secondary evidence; **5 = score-4 evidence PLUS ≥1 cited wow-shape evidence** (whatever the wow shape is for this build)
   Maps to: [Webset enrichment column(s)]

STRUCTURAL DECISIONS:
- Segments: Pipeline / Mid-Market / Enterprise (default — adjust if founder's GTM has different segmentation)
- Section 2 rename: "Opportunity" → "[vertical-appropriate label, e.g., Inference Footprint, Workflow Footprint, Capex Footprint]"
- Tab labels: Pipeline / Mid-Market / Enterprise (default)

Confirm or push back before I move to Webset spec design.
```

**Auto-proceed if the standard pattern fits.** When the founder's vertical is in the per-vertical templates above (inference, healthcare workflow, fintech infra, cybersecurity, data infra), the 4-axis structure (Hiring + Opportunity + 2 founder-specific) and the standard 3-segment scaffold land cleanly. Post the axis plan as visibility and move to Phase 6.

**Stop and ask** only if the founder's vertical isn't in the per-vertical templates and you have to invent founder-specific axes from scratch — that's a significant decision worth a checkpoint.

### Phase 6: Build and run the Webset

This is the main enrichment phase. **The skill's success or failure depends on the quality of the Webset spec.** A great spec returns 25 ICP-perfect companies with rich enrichments. A mediocre spec returns 25 vendors-and-peers with thin data.

The spec is built up across 6a-6f as a multi-step subroutine. Don't shortcut it.

**6a — Re-extract the buyer profile from raw docs (NOT from CONTEXT.md)**

Before writing any spec text, re-read the raw docs in `inputs/`, specifically the founder transcripts, prospect call transcripts, and the relevant memo sections on ICP and buyers. Don't shortcut by re-reading CONTEXT.md; it's a compression and you need the texture.

```bash
# Re-read the raw docs that carry buyer profile texture
cat inputs/granola-*.txt
cat inputs/prospect-call-*.txt 2>/dev/null
pdftotext inputs/memo.pdf - | grep -iA 30 "ICP\|buyer\|customer\|target"  # quick orient
```

Now write two paragraphs, in your own words:

1. **What is the buyer's core business?** (Banking, healthcare claims, threat detection, AIOps, etc., the buyer's industry and the buyer's primary product or service. NOT the founder's industry. NOT the founder's product.) Pull verbatim phrases from how the founder describes their target buyer if available, those phrasings are higher-signal than synthesized descriptions.
2. **What problem does the buyer have that the founder solves?** (The pain, not the solution. E.g., "buyer runs heavy AI inference and faces strict data residency constraints", that's the pain. "buyer needs Valar's inference fabric", that's the solution. Use the pain phrasing.)

This is the disambiguation step. Confusing the buyer with the founder produces garbage Webset results. Common failure mode: searching for "AI inference companies" returns *AI inference vendors*, not *AI inference buyers*. The two paragraphs above force you to be specific about which side of the market you're targeting.

**Why raw docs, not CONTEXT.md?** CONTEXT.md's "ICP Qualifier" is a compression to 3-5 verifiable statements. The texture you need, offhand asides in transcripts, half-finished thoughts, analogies, the founder's attitude toward different competitors, only lives in raw founder talk. The Webset spec quality is bounded by how well-grounded these two paragraphs are.

**6b — Translate axes into enrichment columns**

For each axis you defined in Phase 5, write 1-3 Webset enrichment column descriptions. Each description should be a full paragraph (50-100 words) that specifies:

- **What signal to extract** (concrete, specific to the founder's vertical)
- **What sources to consult** (third-party preferred, SEC filings, analyst reports, engineering blogs, third-party press; NOT the company's own marketing site for credibility-sensitive fields)
- **How to phrase the answer** (specific phrasing requirements, "in USD," "with quoted excerpt," "with source URL inline")
- **What format** (`text` for narratives, `number` for monetary/count values, `options` for controlled vocabulary, `boolean` for binary)

Example, for Valar's "Inference Pain" axis, instead of:

> "Documented pain points or constraints" (BAD, too generic, gets generic data)

Use:

> "Documented evidence that this company runs production AI inference at scale, citing engineering blogs, conference talks, or third-party technical press (NOT the company's marketing pages). Include: estimated annualized inference spend if mentioned, the scale of inference workloads (requests/second, tokens/day), the current inference stack (GPU type, framework, deployment pattern), and any documented pain points (cost overruns, latency issues, capacity constraints). Format: 2-4 sentence narrative with at least one source URL inline. If the company appears to BE an inference vendor rather than an inference consumer, mark this field 'NOT APPLICABLE, vendor not buyer.'"

The "vendor not buyer" exclusion catches the failure mode from the Valar test where Substrate AI and Oxmaint came back as ICP matches when they're actually vendors.

Standard enrichments to include in every FDI Webset (in addition to axis-specific ones):

```javascript
[
  // Identity
  {description: "Industry classification, SIC or NAICS preferred", format: "text"},
  {description: "Most recent annual revenue in USD with reporting year", format: "text"},
  {description: "Headquarters city and state/country", format: "text"},

  // Hiring axis (mandatory) — DEFERRED to Sumble in Phase 6j
  // Do NOT include a hiring enrichment in the Webset request. Webset's job-board
  // enrichment is unreliable; we fetch from Sumble directly per company in Phase
  // 6j after curation. This frees up an enrichment slot for higher-leverage data.

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
- The lookalike anchors from CONTEXT.md (named explicitly as positive examples; pull every name from the Pipeline / Lookalike / Stretch ICP / Outbound spreadsheet sections)
- Explicit exclusions to prevent vendor/peer matches (pull every named competitor from CONTEXT.md's "Competitive Landscape" table; don't just name categories, name the actual companies)
- **Geographic constraint** from `config.json` (default: "United States and Canada only")

Read the geo scope before writing the query:

```bash
GEO_SCOPE=$(jq -r .geo_scope config.json)
echo "Geo scope: $GEO_SCOPE"
```

Template:

> "[Buyer's core business description, 1-2 sentences drawing from 6a paragraph 1.] Their core business is NOT [founder's industry/product category]; they CONSUME [founder's product type] internally as a capability for their own operations, rather than producing or selling it. [Pain description, 1-2 sentences drawing from 6a paragraph 2.] Strongest fit: [3-5 vertical descriptors from CONTEXT.md ICP], and companies similar to [3-5 lookalike anchors named explicitly]. **Geographic scope: [headquartered in the United States or Canada | global | US + EU | etc., based on config.json].** EXCLUDE: [vendor categories that should NOT match, AND specifically named competitors from CONTEXT.md's Competitive Landscape table; for Valar, 'AI infrastructure vendors, AI cloud providers, GPU resellers, AI training data companies, ML platform companies, sovereign AI cloud providers, including but not limited to: Fireworks, Together, Baseten, Modal, Groq, Cerebras, Crusoe, Coreweave, Nebius, Scale AI, Substrate AI']."

The "EXCLUDE" clause is doing real work. Webset agents respect negative phrasing when it's explicit and concrete. Be specific: name competitor categories AND, where possible, name the actual competitors from CONTEXT.md (for Valar: "Fireworks, Together, Baseten, Modal, Groq, Cerebras, Crusoe, Coreweave, Nebius, Scale AI, Substrate AI"). Generic phrases like "AI vendors" leak; named exclusions hold.

**Why an explicit geographic clause matters:** without it, Websets tends to over-return EU companies for any vertical with strong GDPR positioning (banks, insurance, healthcare). The default US/Canada scope filters this out at the search stage rather than having to drop them in Phase 6i curation.

**6d, searchCount**

Default: **15**. Final dashboard target is **10 companies** total. Oversampling 50% allows you to drop the 3-5 weakest Webset returns during Phase 6i curation without re-firing. Lower to 12 if criteria are tight and quality is high; raise to 20 if the founder vertical is broad and you want more options to curate from. Don't go below 12, a thin Webset means no curation room.

**Why 10, not 30:** founders walk through 3 companies max in any demo. Wide coverage hurts more than it helps. 10 companies of high-quality, deeply-researched data beats 30 companies of mixed quality. The skill is biased toward depth-per-company, not breadth.

**Adjusting for force-includes.** Subtract from this number for each company you'll force-include from CONTEXT.md (signed design partners, named pipeline accounts, founder-named ICP picks). Example: if the founder has 4 signed design partners that must be in the dashboard, set searchCount to 10 (you only need 6 more from the Webset to reach 10 total). For Tom's case in May 5 (with 18 named picks), searchCount could have been as low as 8 since most slots were already spoken for.

**6e — Write the searchCriteria**

Webset caps `searchCriteria` at 5 hard filters. Every company must pass all of them. Best practices:

- **Mandatory: include the geographic constraint** as a dedicated criterion (e.g., "Company is headquartered in the United States or Canada"). Reinforces the searchQuery's geo clause; without this, Websets often returns EU/UK matches that pass the other criteria.
- **Mandatory: at least one criterion is exclusion-shaped** (e.g., "Company's primary business is NOT in the [founder's product category]; companies that sell [founder's product type] are excluded"). Reinforces the EXCLUDE clause from the searchQuery; don't rely on the searchQuery alone.
- **Objective and verifiable on the open web.** "Company has $100M+ revenue OR is publicly traded" is verifiable via SEC filings or news. "Company is innovative" is subjective and worthless.
- **Pull from CONTEXT.md's "ICP Qualifier" section** for the 3-5 verifiable statements you wrote in Phase 4. If the founder gave you a hard threshold (e.g., "$1.5M+ annualized inference spend"), do NOT use that exact phrasing as a criterion: it's not externally verifiable. Translate it to a softer proxy ("publicly documented use of production AI/ML inference" or "$500M+ revenue OR publicly traded") and keep the dollar threshold in the searchQuery as qualitative anchor.
- **Don't over-specify.** Webset will return fewer results if criteria are too narrow. Aim for criteria that 30-40% of in-vertical companies would pass. Strict criteria (the kind that pass <5% of candidates) cause the search to fail or return thin results; if the first run shows criterion pass rates below 10%, soften and re-fire.

Recommended 5-criterion structure:
1. Vertical-specific positive criterion (operates in [data-sensitive vertical / regulated industry / etc.])
2. Exclusion-shaped (NOT in [founder's product category])
3. Geographic (headquartered in [US + Canada from config.json])
4. Scale/maturity (revenue threshold or public-trading status)
5. Vertical-specific signal (production use of [product capability])

**Source-type diversification (read this carefully — it determines what universe of companies surfaces):**

Every criterion implicitly reads against a particular part of the web. If all 5 criteria read against the same part, Webset converges on whichever companies are loudest in that part — and you get a one-note result. (Reference build → F9 source-type stacking, where 4 of 5 criteria read regulatory text and Webset returned a heavy European-bank skew.)

The fix is to tag each criterion with the part of the web it reads against, then check that the set spans at least 3 different parts.

Source types to tag against:

| Tag | Reads against | Example criterion phrasing |
|---|---|---|
| `[regulatory]` | SEC filings, compliance disclosures, regulatory pages | "Operates under data residency obligations" |
| `[earnings]` | Earnings call transcripts, IR commentary | "AI inference cited as cost or margin pressure in earnings" |
| `[engineering]` | Engineering blogs, technical docs, GitHub | "Has published engineering content on inference architecture or LLM serving" |
| `[product]` | Product announcements, news, press | "Has named AI product in production (not roadmap)" |
| `[founder-anchor]` | Imported founder CSV / named pipeline | "Listed in founder's outbound CSV" (uses Webset `scope` parameter, not criteria) |

**Discipline:** before submitting the spec, write each criterion followed by its tag. Then count distinct tags. If fewer than 3 distinct tags, revise — replace duplicate-tag criteria with criteria that read against an underrepresented part.

Worked example, Valar:

❌ **Same-tag stacking** (the May 5 failure mode):
1. "Operates under data residency obligations" `[regulatory]`
2. "Subject to data sovereignty regulations" `[regulatory]`
3. "Customer data restricted to specific jurisdictions" `[regulatory]`
4. "Regulatory framework requires data localization" `[regulatory]`
5. "Headquartered in US or Canada" `[geographic]`

→ 1 distinct content tag. Webset hunts in regulatory disclosures, finds European banks (loudest in that corpus), returns them.

✅ **Diversified across parts**:
1. "Operates under data residency or compliance obligations (PCI DSS, HIPAA, SOC 2, FedRAMP, GDPR)" `[regulatory]`
2. "AI inference cited as cost or margin pressure in earnings calls or engineering content" `[earnings + engineering]`
3. "Has named AI product in production with public technical write-up" `[product + engineering]`
4. "Headquartered in US or Canada" `[geographic]`
5. "NOT primarily an inference platform / model-serving vendor" `[exclusion]`

→ 4 distinct content tags. Webset hunts in regulatory text, earnings transcripts, AND engineering blogs. The same Valar ICP, but the universe of candidate companies now includes Datadog (loud in engineering blogs about Bits AI), Plaid (loud in CFPB compliance text AND fintech engineering posts), and CrowdStrike (loud in earnings call AI commentary AND security engineering content) — alongside the regulated-industry banks the first version found.

This is not vertical diversification (a banking-vertical FDI still gets banks). It's *evidence-source* diversification within whatever vertical the founder targets. A banking-vertical FDI gets banks via 3 different lines of evidence per company instead of 1.

**6f — Pre-Webset checkpoint**

Before firing the (paid, async) Webset, do two things:

**Pre-flight: `preview_webset` (free)**

Call `preview_webset` with your `searchQuery` first. This is a free Exa endpoint that returns Webset's interpretation of the query: detected `entityType`, generated `criteria`, and suggested `enrichments`. It's a free sanity check on whether your buyer-profile paragraph reads to Webset the way you intended.

Compare what comes back against what you wrote in 6a-6e:

- Does the detected `entityType` match yours (`company`)? If Webset flags it as something else (`person`, `custom`), the query is being read wrong.
- Do the generated `criteria` overlap with yours? If Webset's auto-generated criteria look meaningfully different, especially if it's missing your exclusion-shaped criterion, the searchQuery's EXCLUDE clause may not be landing. Strengthen it.
- Are Webset's suggested `enrichments` adjacent to yours, or wildly different? If wildly different, your enrichment phrasing in 6b may not match the founder's vertical the way you meant.

Don't blindly accept Webset's suggestions, your enrichments are tailored to the data.js schema, theirs are generic. But significant divergence is a yellow flag worth investigating before you fire.

**User checkpoint**

Post the COMPLETE spec to the user:

```
WEBSET SPEC PROPOSAL:

searchQuery (the buyer profile paragraph, full text):
> [paste here]

searchCriteria (3-5 hard filters, each tagged with the part of the web it reads against):
- [criterion 1]  [tag]
- [criterion 2]  [tag]
- ...

Source-type coverage: [N] distinct content tags across [M] criteria
(target: at least 3 distinct content tags — regulatory, earnings, engineering, product, exclusion, geographic — so Webset hunts in multiple parts of the web rather than converging on one)

searchCount: 15 (oversample by 50% — final dashboard 10 companies)
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

Total: ~15 companies × ~10 enrichments. Estimated cost ~$2-4. Estimated time 5-10 min.

Before I fire this, please confirm:
1. Does the searchQuery accurately describe [Founder]'s buyers (not peers/vendors)?
2. Are there exclusions I'm missing? (Specific vendor categories, geographies, etc.)
3. Are there axis-specific signals I should add or rephrase?
4. Does the lookalike list match what you'd want?
```

Wait for explicit go-ahead. **This is the single most important checkpoint in the entire skill.** It's also the cheapest place to course-correct, fixing the spec before firing is much cheaper than fixing results after.

**6g — Submit and wait**

After user confirmation, save the spec to disk first (for reproducibility), then submit:

```bash
# Save the spec as webset-spec.json so it lives with the build
cat > webset-spec.json <<'EOF'
{
  "searchQuery": "...",
  "searchCount": 15,
  "searchEntity": {"type": "company"},
  "searchCriteria": [...],
  "enrichments": [...]
}
EOF

git add webset-spec.json
git commit -m "Phase 6g: Webset spec saved before submission"
```

Now call `create_webset` with the spec. The response contains a webset ID, capture it. Tell the user:

```
Webset submitted. ID: webset_abc123
Searching for ~15 companies, populating ~10 enrichment fields per company.
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
- Companies returned: 13 (requested 15)
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

**Specifically check for vendor/peer matches** that slipped past criteria, companies whose primary business is selling the founder's product category. Flag these and exclude them from data.js. If 30%+ of results are vendor/peer matches, the searchQuery's exclusion clause needs strengthening, rerun with a tighter spec.

**Auto-proceed to 6i curation** unless 30%+ of returns are vendor/peer false-positives — in that case, stop and ask whether to re-fire with a tighter EXCLUDE clause (a re-fire costs ~$2-4 and 5-10 min; cheaper than curating around bad returns).

**6i — Curate the final company list**

Before contact discovery and data.js population, commit to a final list of **exactly 10 companies**. This is the moment to drop weak matches, add specific founder-named companies that the Webset missed, and lock in segment assignments. Don't skip this, the dashboard's quality is bounded by which companies make the cut.

**Prioritization rule (apply in order until you have 10):**

1. **Signed design partners** (Pipeline). Always include all of them. These are the credibility anchors.
2. **Named active pipeline accounts** (Pipeline). Include if there's room and the founder mentioned them by name in the memo or transcripts.
3. **Founder-named ICP picks** (Enterprise or Mid-Market). The companies the founder explicitly listed. Prioritize ones with strong public data signal so the dashboard entries can be deeply researched.
4. **Webset returns ranked by enrichment quality** (Mid-Market or Enterprise). The strongest signal across the 4 axes wins. Skip any company you can't get to a high-confidence entry on.

If the founder has more than 10 named picks (rare but possible, like Tom's 18 in May 5), pick the 10 with the strongest combination of (a) public data signal, (b) named in pipeline vs only in spreadsheet, (c) warm-intro paths from the Primary network. Document the cut list in BUILD_NOTES.md so the founder knows what didn't make it and why.

**If a company would land in the dashboard but you can't get to a deeply-researched entry on it, drop it.** Better to ship 9 strong entries than 10 with one weak. Pad the slot only if there's a clear-quality candidate to fill it.

**Segment assignment:**
- Pipeline = signed design partners + named active pipeline accounts (from CONTEXT.md). Typically 3-5 of the 10 slots.
- Mid-Market = Stage 1 ICP (the founder's "easier to convert" segment). Typically 2-4 slots.
- Enterprise = Stage 2 ICP (founder's larger/longer-cycle targets). Typically 2-3 slots.

Some companies will fit multiple segments. Default rule: if a company is in Pipeline, it goes in Pipeline regardless of size. If not, size + signal strength determines segment.

Post the final list to the user as a structured proposal:

```
FINAL 10-COMPANY LIST PROPOSAL:

Pipeline ([N] companies, signed/active):
- [Company A] — signed design partner
- [Company B] — named active pipeline (memo)
- ...

Mid-Market ([N] companies, Stage 1 ICP):
- [Company C] — founder-named pick + Webset rank 3, all 4 axes ≥3
- [Company D] — Webset rank 7, strong residency signal
- ... 

Enterprise ([N] companies, Stage 2 ICP):
- [Company E] — founder-named pick + Webset rank 1, scale + opportunity signal
- ...

DROPPED from Webset (with reasons):
- [Company X] — vendor/peer false positive
- [Company Y] — sparse enrichments, couldn't supplement
- [Company Z] — wrong-size/wrong-region for ICP

DOCUMENTED BUT NOT INCLUDED (founder-named, didn't make 10-slot cut):
- [Founder-named company]: weaker public signal, can't research to high confidence
- [Founder-named company]: more relevant for next iteration
(These go in BUILD_NOTES.md so the founder sees the cut list.)

Total: 10 companies.

Confirm or revise before I move to contact discovery + data.js population.
```

Wait for user confirmation. Course-correction here is much cheaper than fixing a populated dashboard.

**6j — Job discovery via Sumble**

Sumble (sumble.com) is the canonical source for tracking tech company hiring. Use it for the hiring axis instead of generic Webset enrichment or LinkedIn scraping. Generic job-board enrichment via Webset is thin and inaccurate; Sumble has structured, company-scoped role data with much higher signal-to-noise. For non-tech verticals where Sumble coverage is uneven, work the fallback ladder in Step 5.

Run AFTER Phase 6i curation so you only fetch jobs for the final dashboard list, not the full Webset return.

**Step 1: Check for a Sumble MCP first.**

```
tool_search query: "sumble jobs hiring"
```

If a Sumble MCP is available with a per-company jobs endpoint, use it directly. If not (more likely today), fall back to web fetch.

**Step 2: For each curated company, locate its Sumble page.**

Sumble's URL convention is `https://sumble.com/company/<slug>` (verify this once at the start of the build with one company before looping). If the conventional slug doesn't resolve:

```bash
# Use Exa to search Sumble for the company
# web_search_exa query: "sumble.com <Company Name> jobs"
# Take the first sumble.com result.
```

**Step 3: Fetch the company's job page and extract relevant roles.**

```bash
# Use Exa:web_fetch_exa with the Sumble URL
# Extract the active job postings.
```

Filter to roles relevant to the founder's pain. Use the hiring keyword regex you generated in Phase 8 step 6 (which derives the keywords from CONTEXT.md's pain language + Phase 5 axis names + antagonist warnings). Keep the regex narrow: each kept role should be a leading indicator that the company is staffing toward the pain the founder solves. Don't keep generic "Software Engineer" postings unless they specifically reference the vertical-relevant skill stack.

**Step 4: Save to disk.**

```bash
# sumble-jobs.json structure:
# {
#   "BigPanda": {
#     "sumble_url": "https://sumble.com/company/bigpanda",
#     "fetched_at": "2026-05-05T...",
#     "jobs": [
#       {"title": "Senior ML Platform Engineer", "team": "Platform", "location": "Mountain View, CA", "url": "...", "posted": "..."},
#       {"title": "Staff SRE", "team": "Infrastructure", "location": "Remote", "url": "...", "posted": "..."}
#     ]
#   },
#   "Varonis": { ... },
#   ...
# }

git add sumble-jobs.json
git commit -m "Phase 6j: Sumble job postings (N/M companies covered)"
```

**Step 5: Fallback ladder when Sumble is empty or company isn't tracked.**

Sumble is the primary source, but it doesn't cover every company. For non-tech enterprises (banks, insurers, retailers, manufacturers) Sumble coverage is uneven. Don't default to empty `jobs: []` — work the fallback ladder before giving up:

1. **Sumble company page** (primary). Try `https://sumble.com/company/<slug>` first. If hit, extract jobs and skip the rest.
2. **`careers.<domain>` direct fetch.** `Exa:web_fetch_exa` against `careers.<company>.com` or `<company>.com/careers`. Filter results by the Phase 5 hiring keyword regex. Capture title + URL + posting date if visible.
3. **LinkedIn job search by company.** `Lovelace:search_linkedin_profiles` won't help here, but `Exa:web_search_exa` with query `linkedin.com/jobs "<Company>" "<role keyword>"` (e.g. `"Capital One" "ML Platform"`) usually surfaces 1-3 specific reqs. Take only ones whose company field exactly matches (LinkedIn returns adjacent companies sometimes).
4. **Greenhouse / Lever / Ashby boards** if the company uses them. `Exa:web_search_exa` with `boards.greenhouse.io/<slug>` or `jobs.lever.co/<slug>`.
5. **Last resort: empty.** If steps 1-4 all return zero relevant roles, set `jobs: []` and note `fallback_attempted: ["sumble", "careers", "linkedin", "ats"]` in `sumble-jobs.json` so Phase 7 knows the gap is researched, not skipped.

The hard floor is **≥1 verified job per company in tier='high'**. If a tier='high' company has zero jobs after the full ladder, drop it to tier='med' before Phase 7. (Reference build → F7 JOB_LISTINGS empty, where 27 of 30 were empty because the build accepted Webset's NULL enrichment without working the fallback.)

**Why Sumble first:** generic job board enrichment via Webset returns inaccurate role descriptions and stale postings (validated in May 5 test build). Sumble tracks company-specific hiring with structured role/team/location data. Higher signal, lower noise. The fallbacks fire only when Sumble has no record.

**6k — Contact discovery via Lovelace**

Run `Lovelace:search_linkedin_profiles` per target company (max 10 results per call). Two queries per company is usually right: one for the primary buyer persona, one for the technical champion persona.

The persona phrasing must come from the founder's vertical and from CONTEXT.md's antagonist warnings, generic "engineering leader" returns garbage.

**Building the queries from CONTEXT.md, not from a vertical lookup:**

The buyer / champion / antagonist personas for this build are already in CONTEXT.md (Phase 4 → Buyer Personas & Messaging section, plus the antagonist warnings). Your job is to translate those personas into the right `title:` filter values. Default to `title:` parameter (validated higher-fidelity than `keywords:`); fall back to `keywords:` only if `title:` returns thin.

Process:
1. Read CONTEXT.md's "Buyer Personas & Messaging" section. The buyer is whoever the founder said is most receptive (often a platform / operations / domain leadership role). The champion is the technical advocate who'd own the day-to-day evaluation.
2. Read CONTEXT.md's antagonist warnings (per Phase 2). These are the personas the founder has explicitly flagged as either threatened by the founder's product (and therefore likely to block) or wrong-fit buyers. Antagonist personas must NOT be queried for as champions; if Lovelace returns one as a top match, drop them or demote to a non-champion `note`-only entry.
3. Translate the buyer/champion personas into specific senior titles for `title:` queries. For each company, run two queries: one buyer query + one champion query. Generic role labels ("engineering leader") return garbage; use specific senior titles.
4. Use your knowledge of the vertical to map the founder's persona description to the canonical senior titles in that vertical. If a founder's persona is "platform / infrastructure leader at a cloud-consuming enterprise," the canonical titles are VP/Head Platform Engineering, VP/Head ML Infrastructure, VP Cloud Infrastructure. If the founder's persona is "VP of Revenue Cycle at a hospital system," the canonical titles are VP/Director Revenue Cycle, Director RCM Operations. The model knows the vertical's title taxonomy; use it.
5. If you genuinely don't know the canonical senior titles for a vertical, run one Exa search ("[vertical] VP titles" or "[founder's persona description] LinkedIn") to discover them before firing Lovelace queries. Cheaper than firing a noisy Lovelace query.

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
2. In the contact's `note` field (if you add one), specify the connection: "Warm via [Teammate]: [shared context, same school, prior coworkers, mutual connection X]"

Warm intros are the highest-signal contact for the founder. Surface them in the dashboard's Network view and in `gtm_thesis` if a target company has multiple warm paths.

**Sequencing:**

For a 25-company Webset, that's potentially 50 Lovelace calls. Sequence intelligently, high-tier companies first, low-tier last. Stop early if you hit rate limits or low signal.

**Save to disk:**

Once contact discovery is done, save the aggregated results so Phase 7 can populate CONTACT_MAP from disk (not from re-calling Lovelace):

```bash
# Save contacts as lovelace-contacts.json keyed by company name
# {"BigPanda": [{name, title, linkedin, primary_connection}, ...], "Varonis": [...], ...}

git add lovelace-contacts.json
git commit -m "Phase 6j: Lovelace contacts gathered (47 profiles across 23 companies)"
```

**6l — Fallback ladder if Webset returns thin or off-target**

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

**6m — Targeted research for founder-named picks not surfaced by Webset**

Founder-named accounts (signed design partners, named pipeline, ICP picks from a CSV) often don't surface in Webset because Webset is a *discovery* tool — it finds new ICP-fit companies, it doesn't validate the founder's existing list. (Reference build → F6 founder-pick research gap, where 16 of 18 founder-named picks went into Phase 7 with thin sourcing because per-company directed research was skipped.)

**Don't skip this phase for founder-named accounts.** They are the highest-stakes entries in the dashboard — these are the companies the founder's pitch hinges on, and shallow entries here are the most-noticed quality gap.

**Step 1: Identify the gap.**

After Phase 6i curation, list every company in the curated 10 that did NOT come back from Webset. This is the founder-named-pick research backlog.

```bash
# Compare curated list against Webset returns
# Output: list of companies needing per-company directed research
```

**Step 2: Per company, run a 4-query directed research pass.**

For each founder-named pick not in Webset, run these 4 `Exa:web_search_exa` queries in parallel:

1. `"<Company> 10-K SEC EDGAR"` — gets the SEC filing for public companies. For private, swap to `"<Company> latest funding round" OR "<Company> annual revenue"`.
2. `"<Company> AI inference engineering blog"` — surfaces engineering content. Add `engineering.<company>.com` to the query if the company is known to host one.
3. `"<Company> Chief AI Officer" OR "<Company> VP Platform Engineering"` — leadership / champion personas.
4. `"<Company> earnings call AI infrastructure"` for public companies, OR `"<Company> AI strategy"` press for private.

For regulated-vertical companies (banks, insurers, healthcare), add a 5th query targeting the wow signal:

5. `"<Company> Fireworks OR Together OR Baseten OR Modal OR Anyscale failed OR blocked OR security OR compliance"` — surfaces tried-and-blocked evidence (the Data Residency axis 5 requirement).

**Step 3: Save to disk.**

```bash
# founder-pick-research.json structure:
# {
#   "Mastercard": {
#     "sec_filing": {"title": "Mastercard 2024 10-K", "url": "...", "snippet": "..."},
#     "engineering_content": [{"title": "...", "url": "...", "date": "..."}, ...],
#     "leadership": [{"name": "...", "title": "...", "linkedin": "..."}, ...],
#     "earnings_signals": [{"quote": "...", "source": "...", "date": "..."}, ...],
#     "tried_and_blocked": [{"vendor": "Fireworks", "blocker": "...", "source_url": "..."}] // empty if none found
#   },
#   "Walmart": { ... },
#   ...
# }

git add founder-pick-research.json
git commit -m "Phase 6m: targeted research for N founder-named picks not in Webset"
```

**Step 4: Phase 7 reads from this file.**

When populating data.js for any founder-named pick, Phase 7's source data is the union of `webset-response.json` (if the company is there, rare) AND `founder-pick-research.json` (where most founder-named picks live). Phase 7 must reach the same source-quality floor (≥4 sources, ≥1 SEC filing for public, ≥2 engineering blog posts for tech-forward) for founder-named picks as for Webset-discovered ones.

**Why this matters:** the dashboard's credibility scales with its weakest entry. If 8 of 10 entries have 6 sources and 2 of 10 (the founder's named picks) have 2 generic sources, the dashboard reads as inconsistent. Founder-named picks are exactly the entries where the founder will look hardest at the sources, so the source quality has to match or beat the discovered companies.

**Cost awareness**

A real test on May 4 with 5 companies × 3 enrichments returned in ~5 minutes for ~$0.50–1. Production Webset of 15 companies × 10 enrichments ≈ $2–4, ~5–10 minutes. Lovelace per-call. Sumble fetches minimal. Phase 6m founder-pick research adds ~4-5 Exa search calls per founder-named-not-in-Webset company (so 10 picks × 5 queries × $0.05 ≈ $2.50 worst case). Total per FDI build: $5–12.

### Phase 7: Populate data.js

**Before writing any company entries, re-read `template/TEMPLATE_GUIDE.md` Section 9 ("Field-by-field craft patterns").** This section extracts concrete writing patterns from the V1 Valar dashboard, every example is real V1 copy. Following the schema is necessary but not sufficient; following the *patterns* is what separates a generic dashboard from one that lands.

```bash
# Refresh on the patterns before populating
sed -n '/^## 9\./,/^## 10\./p' template/TEMPLATE_GUIDE.md
```

**Working from disk:** read company data from `webset-response.json`, contacts from `lovelace-contacts.json`. Don't re-call MCP tools at this phase, the saved files are your source of truth.

**Build incrementally, one company at a time, not in batch.** Generic batch-population is the failure mode that produces "Valar with names changed" output. After each company, run a quick self-check before moving on. This is slower but produces dramatically higher quality.

**Start by copying the template's `data.js` as your base:**

```bash
cp template/data.js data.js
# Open data.js, identify the const declarations: ROW_SOURCES, SEGMENTS, CONTACT_MAP,
# COMPANY_SOURCES, JOB_LISTINGS, RESIDENCY_MAP, PRIMARY_TEAM
# You'll fully replace SEGMENTS, CONTACT_MAP, COMPANY_SOURCES, JOB_LISTINGS, RESIDENCY_MAP, ROW_SOURCES.
# Keep PRIMARY_TEAM array structure but populate with Primary's actual roster from CONTEXT.md.
```

**For each company from the curated list (Phase 6i):**

1. **Pick the right segment**, based on signal strength + ICP match (Pipeline if signed/active, Mid-Market for Stage 1 ICP, Enterprise for Stage 2).

2. **Set `tier`**, `'high'` if all axes scored 4+ AND the company is signed/in-pipeline OR has a strong warm intro; `'med'` for strong ICP fit with mixed axis scores; `'low'` for speculative pattern-matches without warm path. With the 10-company target, aim for roughly **5 high / 4 med / 1 low**. If you have no `'low'` candidates worth including, drop the slot — better 9 entries you'd demo than 10 with one weak. If everything is `'high'`, tiers carry no information; force at least 2-3 into `'med'` based on which had less verifiable enrichment data.

3. **Write `subtitle` in V1 pattern**, *[what the company is], [why-they-fit-the-founder phrase], [founder relationship status]*. One sentence, dense, signal-rich. See TEMPLATE_GUIDE.md Section 9.1.

4. **Write `overview` in V1 pattern**, 3-5 sentences that name the company's position in the founder's market story, specify the data sensitivity in concrete terms (not abstract), and connect the company to a category-level reference. See Section 9.2.

5. **Fill the 3 sections** (Profile / Inference Footprint / GTM Strategy) directly from Webset enrichments. Locked field sets — do NOT add fields beyond these:
   - **Profile (5 rows exactly)**: Industry, Revenue, Employees, Cloud Provider, AI Maturity. Do not add Founded, Headquarters, "Valar Status" (or any "[Founder] Status"), Stage, ICP Tier, or Business Type — those duplicate information shown elsewhere. The relationship status lives in the `tags` array as a brand-color chip, not as a profile row. See Section 9.8.
   - **Inference Footprint (4 rows exactly)**: Use Cases, Current Stack, Pain Points, Estimated Spend.
     - Pain Points must lead with the financial consequence (margin compression, COGS impact, gross-margin drag), then list constraints. CFO language, not engineer language. See Section 9.8.
     - Estimated Spend always uses a range AND shows the estimation method in parens. NEVER write "needs verification", "TBD", "unknown", or any placeholder — either compute the range with a defensible method, or omit the row entirely.
   - **GTM Strategy (5 rows exactly)**: Approach, Key Evidence, Urgency Level, Target Buyer, Messaging Angle. Urgency Level uses uppercase action verbs (EXECUTE / HIGH / WARM / MED / COLD / DEFER). Target Buyer splits Buyer + Champion when both are knowable. Messaging Angle includes a quoted opening line.

6. **Score the founder-specific axes 0–5** and write the trio for each: score + bullet signals + reasoning paragraph. See Section 9.5-9.7. The reasoning paragraph should be honest about challenges (V1 Mastercard's `opp_reason` flagged in-house expertise as a hurdle), credibility carries.

7. **Write `gtm_thesis` in the V1 three-sentence pattern**: anchor sentence + motion sentence + buyer call-out. Splice verbatim founder quotes from `inputs/granola-*.txt` or CONTEXT.md if they fit. End with `**Buyer:** [persona]` (and `**Champion:**` if known), use **NOT [persona]** when the founder has named antagonist personas. See Section 9.3.

   **Durability constraint:** the thesis must survive personnel changes. Buyer and Champion in the gtm_thesis are *role types* / personas (V1: "Platform Engineering / Site Reliability lead", "Security/Compliance leadership"), NOT specific named individuals. Specific names belong in `CONTACT_MAP` (the Connections section), which is the dynamic layer.

   Forbidden in the gtm_thesis:
   - Specific named individuals at the target company ("John Morgan, Managing VP Product")
   - Specific named individuals at Primary who can intro ("Vivek Gupta, warm via Alex")
   - Comparative warmth claims that hinge on personnel ("highest-warmth account in the FDI")
   - Specific warm-intro paths ("via Alex", "lead with Vivek")

   Allowed in the gtm_thesis:
   - Persona/role-type buyer + champion (always)
   - Antagonist exclusions as roles ("NOT ML engineering function broadly")
   - Verbatim founder quotes from CONTEXT.md (strategic anchors, not personnel facts)
   - Aggregate warm-contact counts as descriptive attributes ("Primary has multiple warm contacts here") — but framed as attribute, not strategy
   - Founder relationship status if it's structural ("signed design partner", "named pipeline pick")

   (Reference build → F4 personnel-fragile thesis. V1's Capital One thesis described the same company in durable terms — culture, technology fit, business need, contact-count-as-attribute — and would survive personnel rotation.)

8. **Write `tags` 3-5 chips with mixed colors** (`Valar`/`brand` for relationship, `stack` for technical/constraint, `hw` for hard constraint, `hiring` for hiring signal, `neutral` for factual). Tooltips required if the tag is non-obvious. Hiring tags prefixed with the role being hired (e.g., `'Hiring: ML Platform'` for inference founders, `'Hiring: RCM'` for healthcare workflow, `'Hiring: Payments'` for fintech). **Banned tag values** (do not use): `Stage-1 ICP`, `Stage-2 ICP`, `Stage 1`, `Stage 2`, `Pipeline`, `Mid-Market`, `Enterprise`, `Target`, `ICP`, `In ICP`, or any other segment-classification meta-tag. Tags must reference product names, technical stack, constraints, relationship status, or hiring signals — never segment classification (the segment is already shown by the tab). See Section 9.4.

9. **Populate `CONTACT_MAP`** with platform/infrastructure leadership keyed exactly to `SEGMENTS[].companies[].name` (character-for-character match including parentheses). Read from `lovelace-contacts.json`. Persona discipline reflects founder antagonist warnings, exclude personas the founder has flagged. Cross-reference Primary network for warm intros. See Section 9.10.

10. **Populate `JOB_LISTINGS`** from `sumble-jobs.json` (Phase 6j). Each company's `jobs[]` array maps to JOB_LISTINGS entries with title, team, location, URL. If a company has empty jobs in Sumble data, leave its JOB_LISTINGS entry empty rather than backfilling with weaker data.

11. **Populate `COMPANY_SOURCES`** from `webset-response.json`, `founder-pick-research.json` (Phase 6m results for founder-named picks), AND targeted web fetches. Source list quality is what readers use to judge the rest of the dashboard. Counts and rules:
    - **Target: 6 sources per company** (V1 averages 6; reference build → F2 source thinness).
    - **Hard floor: 4 sources.** Below 4 is unacceptable. Either fetch the missing tier(s) yourself (SEC filing for public companies, engineering blog posts for tech-forward companies), drop the company's tier from `high` to `med`, or replace the company in the curated 10.
    - **4-5 sources is acceptable but try once more to reach 6** before moving on, especially if Tier 1 or Tier 2 sources are missing.
    - **For public companies: ≥1 SEC filing.** 10-K, 10-Q, S-1, DEF 14A, or 8-K from sec.gov. If the company is publicly traded and your Webset enrichment didn't surface a 10-K, fetch one yourself: `Exa:web_search_exa` with query `"<company> 10-K SEC EDGAR"` and add the top result.
    - **For tech-forward companies: ≥2 named engineering blog posts.** Posts with specific technical titles ("LLMs for Postmortems", "State of AI Engineering"), not generic landing pages. If Webset didn't surface them, fetch from the company's `/blog` or `engineering.<company>.com`.
    - **≤1 trade press source** (TechCrunch, The Information, etc.). Use trade press to round out, not anchor.
    - **No PR aggregators alone.** Don't ship BusinessWire / PRNewswire / GlobeNewswire press releases as primary sources. Find the trade-press follow-up or the underlying primary record (FedRAMP Marketplace listing, the actual product page, the actual blog post). PR wires republish the company's own claims; they're distribution, not journalism.
    - **No standalone job board sources** (Greenhouse, Lever, /careers). Job activity belongs in `JOB_LISTINGS`, not `COMPANY_SOURCES`.
    - **Source titles describe content, not just outlet.** Format: `[Source/Outlet] — [Specific topic or filing]`. Good: `"Datadog Engineering — LLMs for Postmortems (Bits AI)"`. Bad: `"Datadog Blog"` (which post?), `"Celonis AI copilot"` (no outlet attribution). The em dash here is permitted — it's a structural separator within source titles.
    - **Budget guardrail.** Each company should require ≤3 extra Exa fetches beyond the Webset baseline. If a company would need 5+ extra fetches to reach 6 quality sources, that's a signal the research case is thin — drop its tier or replace it.
    - See TEMPLATE_GUIDE Section 9.12 for the full source quality hierarchy.

12. **Populate `RESIDENCY_MAP`** (or whatever the founder-specific axis-2 field is named, per Phase 5) from `webset-response.json` and `founder-pick-research.json`. Each entry pairs a company with a one-sentence reason that quotes the underlying source language where possible.

    **Score-5 evidence requirement (per Phase 5 rubric).** Before assigning a 5 on any founder-specific axis, the reason field must include cited evidence in **the wow signal's evidence shape** (per CONTEXT.md's "Wow Signal" section). The evidence shape is defined by what the wow signal documents — for inference founders this is a tried-and-blocked vendor citation; for industrials predictive-maintenance founders this is a documented legacy-system reference (e.g., "10-K cites $X annual unplanned-downtime cost from end-of-life DCS"); for biotech founders this is a cited regulatory milestone (e.g., "FDA submission pending on a single-vendor pipeline with named alternative pending validation"); for fintech founders this might be a CFO earnings-call quote about a named compliance gap. Use CONTEXT.md to determine the right shape for this build.

    If no wow-shape evidence is in `webset-response.json` or `founder-pick-research.json`, cap the score at 4 even if the underlying posture would justify a 5. The wow axis only earns its weight when the top score requires the wow evidence. (Reference build → F3 score inflation.)

13. **Cite via `ROW_SOURCES`** for every numeric or specific claim. Webset returns sources inline within text fields (pattern: `fact text | URL / fact text | URL`); when reading any enrichment text into a `sections` row, scan for URLs (regex `https?://[^\s\)]+`), extract them into ROW_SOURCES entries, and use the cleaned text (without inline URLs) as the row value. Density target: every Profile or [vertical-named] Footprint row containing a number, named product, regulatory standard, or other verifiable specific should have a `ROW_SOURCES` entry. V1 averages 5 of ~10 cited rows per company; aim there. (Reference build → F8 citation density gap.) Empty entries are fine; wrong entries are worse than nothing. See Section 9.9.

**Self-check after each company entry, before moving on:**

- **Subtitle ≤ 18 words.** Count them. If higher, rewrite shorter. No parenthetical financial metadata ($X revenue, X employees, founded YYYY) — that belongs in the Profile section, not the subtitle. No trailing "...". See Section 9.1.
- **No placeholder text anywhere.** Search the entry for "needs verification", "TBD", "unknown", "to be confirmed", "?", "[insert", "lorem". Any match means either fill the field with a defensible value (compute the estimate, find the source, name the product) or remove the row entirely. Never ship the placeholder. This is the single biggest credibility killer; the V2 Celonis Estimated Spend showing `$3-8M annual inference (needs verification)` is the failure mode to avoid.
- **Profile is exactly 5 rows.** Industry, Revenue, Employees, Cloud Provider, AI Maturity. No Founded, no Headquarters, no Valar Status, no other fields. If the saved Webset enrichment supplied other fields, drop them — they don't earn their space.
- **Inference Footprint is exactly 4 rows.** Use Cases, Current Stack, Pain Points, Estimated Spend.
- **Pain Points leads with financial consequence.** First sentence of Pain Points should be CFO language (margin, cost, COGS, opex, cash). Constraint enumeration is the second sentence onward, not the first. If the field reads as "X; Y; Z; W" with semicolons, rewrite — that's enumeration, not framing.
- **No banned tag values.** Search the `tags[]` array for "Stage-1 ICP", "Stage-2 ICP", "Pipeline", "Target", "ICP". If any are present, replace with product names, technical stack, constraints, or relationship status.
- **GTM thesis swap test.** Strip the company name from the `gtm_thesis`. Could you swap any other company's name in and have it still make sense? If yes, rewrite — the thesis isn't specific enough.
- **GTM thesis durability test.** Read the `gtm_thesis` and ask: if every named individual in this thesis left their job tomorrow, would the thesis still hold? Specifically scan for: named buyers ("John Morgan"), named champions ("Vivek Gupta"), named warm-intro paths ("via Alex"), comparative claims tied to personnel ("highest-warmth account"), specific role+name combinations ("EVP Chief Scientist Prem Natarajan"). If any are present, move them to CONTACT_MAP and replace with role types in the thesis. Buyer/Champion in the thesis are personas ("Platform Engineering leadership"), not humans.
- **Antagonist persona consistency check.** If the gtm_thesis ends with `**NOT [persona]**` (e.g., **NOT** ML engineering, **NOT** security/governance, **NOT** controls engineering, **NOT** clinical operations — whatever persona this build's antagonist warning names), grep the company's CONTACT_MAP entries for matching persona keywords. The titles flagged in NOT must NOT appear as recommended champions. Worked example: gtm_thesis says **NOT** Technology Governance team → CONTACT_MAP cannot list a contact whose title contains "Technology Governance" as `type: 'business'` champion. If a match exists, drop the contact, demote them to a non-champion `note`-only entry, or rewrite the antagonist callout. (Reference build → F5 antagonist contradiction.)
- **Em dashes ≤ 1 per entry.** Count em dashes across the entry (subtitle, overview, gtm_thesis, all section values). Target: zero or one. If higher, rephrase using commas, periods, or parentheses.
- **Source citation density.** Count cited rows in Profile + Inference Footprint. V1 averages 5 of ~10. If your entry has fewer than 4 of 9 cited, you missed URL extraction in step 13 — go back and parse the Webset enrichment text more carefully.
- **COMPANY_SOURCES count.** Open the entry's source list. Count it. Target is 6 (V1 average). Hard floor is 4 — under 4 means escalate (extra Exa fetches for missing tier, or drop the company's tier, or swap company). 4-5 is acceptable if Tier 1 / Tier 2 quality is present, but try once more to reach 6 first. (Reference build → F2 source thinness.)
- **No PR-aggregator-only sourcing.** If COMPANY_SOURCES is anchored on BusinessWire / PRNewswire / GlobeNewswire press releases without a Tier 1 (SEC) or Tier 2 (engineering blog) source alongside, the source list is too weak. Find the trade-press follow-up or the underlying primary record.
- **Source titles describe content.** Each entry in COMPANY_SOURCES should follow `[Outlet] — [Specific topic]` format. Bare outlet names ("Datadog Blog") or bare article titles ("Celonis AI copilot") fail the test. See TEMPLATE_GUIDE Section 9.12.
- **Generic test.** If you couldn't tell the entry apart from another company in the same segment, the patterns aren't landing.

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

The template HTML carries Valar's labels and structure as defaults, almost everything needs swapping for a non-Valar founder. Start by copying the template:

```bash
cp template/index.html index.html
```

Now edit `index.html` directly. Walk this checklist:

1. Replace `{{PRODUCT_NAME}}` with the founder's product name throughout (~5-7 occurrences depending on template version).
2. Replace `{{PRODUCT_SLUG}}` with a lowercase-dash slug for the Slack channel reference.
3. In `<script>`, find `buildScoreTips()`, it has hardcoded label strings ("Inference Pain", "Data Residency", "Buying Trigger", "Hiring"). The first two are Valar-specific; replace with your axis names from Phase 5. The latter two ("Buying Trigger" → conceptually "Opportunity Score" axis, and "Hiring") are mandatory axes, keep their function but rename labels if you want different display names.
4. In `dotsRow()` call sites (search for `dotsRow('Inference Pain'`, `dotsRow('Data Residency'`), update labels to match Phase 5 axis names. There are typically ~4 of these calls per render.
5. Update the `labels` object inside the detail-view tooltip code (search for `labels = {pain:'Inference Pain'...}` to update those references).
6. In `computeJobSignal()`, replace the hiring keyword regex with vertical-appropriate keywords. Generate the regex from: (a) CONTEXT.md's ICP Qualifier and pain language, (b) Phase 5 axis names, (c) the antagonist warnings (exclude antagonist-role keywords; for an inference founder don't include "data scientist" because Wiggin flagged ML eng as antagonist; for an industrials founder don't include "controls engineer" if the founder named them as threatened by external automation vendors). The regex should be the leading-indicator job titles + skills that signal a company is staffing toward the pain the founder solves. Test on 3 known-prospect careers pages before locking it in — the regex should match at the right companies and miss at obviously-wrong-fit companies.

   Examples by vertical (use as inspiration, not template):
   - Inference / AI infra: `/inference|llm |triton|tensorrt|sglang|vllm|model serving|ml platform|gen[ ]?ai platform|kernel/i`
   - Industrials predictive-maintenance: `/predictive maintenance|condition monitoring|industrial ai|reliability engineering|asset performance|opc-ua|industrial iot/i`
   - Healthcare RCM: `/revenue cycle|denial management|prior authorization|claims operations|coding|RCM/i`
   - Fintech infra: `/payments engineering|fraud engineering|compliance engineering|banking platform|payment rail/i`
   - Cybersecurity: `/detection engineering|security platform|security operations|threat detection|SIEM|SOC/i`
   - Biotech / medtech: `/clinical operations|regulatory affairs|bioinformatics|process development|GMP|computational biology/i`
   - Consumer / retail: `/personalization|merchandising|recommendation systems|search relevance|consumer data platform/i`
   The model knows the title taxonomy for any given vertical — derive the regex rather than reaching for a template.
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
- Data quality notes, strong / thin / missing
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

**Deliver, present the repo to the user:**

```bash
# Confirm repo state
pwd
ls -la
git log --oneline
```

Closing message to user:

> "Dashboard build complete in `~/fdi/<founder-slug>/`. Open `index.html` in your browser to preview locally, no server needed. Read `BUILD_NOTES.md` first; it documents the structural choices and any open questions.
>
> The 10% of work you still own: pick the 5 companies to lead with in the demo, fine-tune narrative for [founder]'s voice, and spot-check the [N flagged data points] from BUILD_NOTES.md.
>
> The repo is initialized with per-phase commits, `git log --oneline` shows the build history. The GitHub remote was set up in Phase 0; run `git push` to send your changes. From there, connect the repo to Vercel for a hosted preview URL. To refresh data later, the saved Webset ID is in BUILD_NOTES.md and `webset-spec.json` is committed for reproducibility."

---

## Tips

**Claude Code-specific:**

- **Stay grounded in `inputs/` throughout the build.** The skill repeats "re-read the raw docs" at multiple phases for a reason, CONTEXT.md is a compression. Phase 6a especially: when you go to draft the Webset spec, run `cat inputs/granola-*.txt` again. The texture lost in CONTEXT.md is what disambiguates the buyer profile.
- **Commit per phase.** Real version history matters. If Phase 7 produces a bad data.js, `git revert` is one command. If you batch-commit at the end, you've lost the ability to roll back to a clean intermediate state.
- **Use the saved JSON files as your source of truth in Phase 7+.** Don't re-call MCP tools when populating data.js, read from `webset-response.json` and `lovelace-contacts.json`. This makes Phase 7 reproducible: you can re-run it without re-firing the Webset.
- **Validate JS parses incrementally.** `node --check data.js` after every ~5 companies catches syntax errors while context is small. A malformed data.js means the dashboard won't render, and the error gets harder to find the more you've added.
- **Skip Granola exports if Granola MCP is connected.** Instead of asking the user to drop transcripts in `inputs/`, query Granola directly. Same data, less friction.

**Skill-wide:**

- **Resist asking too early.** Phase 1 (read everything first) is non-negotiable. If you skip ahead, you make the user repeat what's already in the docs.
- **Push hard on the CRITICAL questions**, lookalikes, exclusions, the wow signal, founder quotes. These are what separate a generic dashboard from one that lands.
- **Buyer ≠ founder.** The single most common Webset failure mode is searching for the founder's industry instead of the buyer's industry. For an inference fabric founder, the buyer is a bank or a healthcare company, not another AI infrastructure vendor. Phase 6a's two-paragraph buyer profile exists specifically to prevent this. A real test on May 4 with a well-intentioned generic spec returned 3 of 5 vendor/peer matches; the rewrite fixes this by demanding the buyer profile be written explicitly before any spec.
- **Long enrichment descriptions are not optional.** Every enrichment description should be a 50-100 word paragraph specifying signal, sources, format, and exclusions. "Documented pain points" returns generic. "Documented evidence that this company runs production AI inference at scale, citing engineering blogs or third-party press; if the company appears to BE an inference vendor rather than a consumer, mark NOT APPLICABLE" returns specific signal.
- **The pre-Webset checkpoint (6f) is the cheapest place to course-correct.** Always pause there for user confirmation before firing a Webset. Fixing a spec costs zero. Re-running a Webset costs $5-10 and 10 minutes.
- **Always include the EXCLUDE clause in searchQuery.** Webset agents respect explicit negative phrasing. "Companies similar to Capital One" is good; "Companies similar to Capital One. EXCLUDE: AI infrastructure vendors, AI cloud providers, GPU resellers" is much better.
- **Name actual competitors in EXCLUDE, don't just name categories.** Generic exclusions ("AI vendors") leak. Named exclusions ("Fireworks, Together, Baseten, Modal") hold. Pull the explicit competitor list from CONTEXT.md's Competitive Landscape table, every named player goes into EXCLUDE.
- **Don't fight the async.** If the Webset takes 8 minutes, take 8 minutes. Don't try to populate data.js with placeholders during the wait, you'll do double work.
- **Founder voice in `gtm_thesis` is the leverage point on copy.** Splice verbatim quotes from `inputs/` or CONTEXT.md.
- **The wow signal should be visible from any angle.** Surface it in `gtm_thesis`, in `signals[]`, and in tags.
- **The most common second failure mode is over-anchoring on the reference build.** The skill carries Valar/inference examples to anchor patterns; do not let those examples constrain your axes, persona templates, or wow shape. If the output sounds like the reference build with names changed, you've under-delivered. Vertical-specific signal axes (beyond the mandatory Hiring + Opportunity) are required, not optional, and they should come from THIS founder's CONTEXT.md, not from the reference build.
- **Source every numeric claim.** Empty `ROW_SOURCES` is fine; wrong `ROW_SOURCES` is worse than nothing. Webset returns source URLs inline within text fields (pattern: `fact text | URL / fact text | URL`); parse these out into ROW_SOURCES rather than discarding them.
- **Don't fabricate data the Webset didn't return.** If a row's value isn't externally derivable, omit the row entirely or compute a defensible range with a stated method. Never ship "needs verification" / "TBD" / "?" as a row value (see Output Style Rules #7). Flag any thin areas in `BUILD_NOTES.md` so the founder knows what to follow up on.
- **Watch for 402 errors mid-build.** If credits run out partway through, finish what you can, mark the rest as needing manual fill, and flag in BUILD_NOTES.md. Don't silently skip companies.
- **The 0–5 axis math is generic, leave it alone.** Only the labels and axis count change between projects.
- **Save the Webset ID and webset-spec.json.** Future iterations can refresh data without rebuilding the query, and Exa monitors can keep the list fresh on a schedule.
- **Reference the gold standard:** `github.com/alexg207/valar-fdi` for tone/density/specificity. The Valar V1 build is the inference-vertical reference. For non-inference verticals, V1's tone and rigor still apply; only the vertical-specific patterns change.

---

## Reference build: Valar V2 (May 5 2026)

The skill body cites failure modes from a single end-to-end test build (Valar, inference vertical, May 5 2026 — `github.com/alexg207/valar-v2-test-fdi`). Those examples are pedagogical; **none of them describe a constraint on the build you are running now**. They exist so you can recognize the failure mode when it shows up in your own output, regardless of vertical.

### Failure modes catalogued

**F1 — Placeholder leak.** May 5 V2 shipped `Estimated Spend: "$3-8M annual inference (needs verification)"` as a row value. The placeholder reads as junior research. Output Style Rule #7 prevents recurrence.

**F2 — Source thinness.** May 5 V2 Celonis card shipped 3 sources (BusinessWire press release + TechCrunch + Greenhouse job board) — under the 4-source hard floor and anchored on a PR aggregator. Output Style Rule #12 prevents recurrence.

**F3 — Wow-axis score inflation.** May 5 V2 assigned residency scores of 4 or 5 to nearly every regulated-vertical company based on regulation alone. UHG (the actual wow exemplar with cited "tried Fireworks/Together/Baseten/Modal — none reached production due to security") got drowned in a pool of 25 indistinguishable 4s and 5s. Phase 5 axis-4 rubric and Phase 7 step 12 prevent recurrence by requiring score-5 evidence in the wow shape.

**F4 — Personnel-fragile gtm_thesis.** May 5 V2 Capital One thesis named John Morgan, Vivek Gupta, Prem Natarajan, and Alex by name and called Capital One "the highest-warmth Stage-2 account." Every claim breaks if any of those four rotate roles. Output Style Rule #13 (personnel-durable thesis) and Phase 7 self-check (durability test) prevent recurrence.

**F5 — Antagonist contradiction.** May 5 V2 Mastercard entry listed Kiran Jayant (VP Technology Governance) as a champion AND said `**NOT** Technology Governance` in the gtm_thesis. Phase 7 self-check (antagonist consistency lint) prevents recurrence.

**F6 — Founder-pick research gap.** May 5 V2 had 18 founder-named picks; Webset surfaced only 2 (Capital One, Bank of America). The other 16 went into Phase 7 with thin sourcing because the build skipped per-company directed research for accounts not in Webset. Phase 6m (founder-pick research path) prevents recurrence.

**F7 — JOB_LISTINGS empty.** May 5 V2 had 27 of 30 JOB_LISTINGS empty because Webset's job-board enrichment returned NULL and the build stopped there instead of working a fallback ladder. Phase 6j Step 5 (Sumble + careers + LinkedIn + ATS fallback) prevents recurrence.

**F8 — ROW_SOURCES citation density gap.** May 5 V2 had `src` tags on only 2 of ~12 fields per company; V1 averages 5 of ~10. Phase 7 step 13 (URL extraction discipline) and Phase 7 self-check (citation density count) prevent recurrence.

**F9 — Source-type stacking.** May 5 V2 Webset criteria had 4 of 5 reading against compliance/regulatory text (data residency, sovereignty, GDPR-style framing). Webset hunted in regulatory disclosures and brought back European banks loudest in that corpus. Phase 6e source-type tagging (require ≥3 distinct content tags across criteria) prevents recurrence.

**F10 — Estimated Spend not externally derivable.** May 5 V2 estimated annualized inference spend across 30 companies, all flagged "needs verification" because no public source cites this directly per company. The enrichment ask was structurally wrong. Pattern: when an enrichment field's evidence isn't externally available for the population you're scoring, drop the field or replace with a defensible 4-bucket enum, not "needs verification" filler.

### What the reference build got right

- The 4-axis structure (Hiring + Opportunity + Inference Pain + Data Residency) was correct for the inference vertical and surfaced the wow exemplar (UHG) — the failure was rubric inflation, not axis selection.
- The Phase 6f checkpoint caught a 3.3% pass-rate criterion before the Webset fired — saved $5 and 10 minutes by re-firing with softened criteria.
- Founder voice splicing in gtm_thesis (verbatim Tom Amsterdam quotes from CONTEXT.md) materially improved the dashboard's tone over V1's hand-build.
- The interaction model and per-phase Git commits made the build auditable.

### Pattern: "Valar with names changed"

If your output for a non-inference founder reads like Valar with names changed — the same axis labels, the same persona descriptions, the same wow-evidence shape — you've under-delivered. The reference-build patterns are scaffolding for the discipline, not the content.
