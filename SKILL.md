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
  - **Basic Exa MCP**, required as a fallback for `web_search_exa` and `web_fetch_exa` calls during company-list curation, Sumble fetching (Phase 6j), and founder-pick research (Phase 6m).
- **CRITICAL: Websets MCP and basic Exa MCP are on SEPARATE credit pools.** They share the dashboard at `dashboard.exa.ai/api-keys` but bill independently. Websets credits cover the `mcp__websets__*` calls (search + enrichment); basic-Exa credits cover `web_search_exa` + `web_fetch_exa`. Either pool can hit a 402 while the other still works. **Both must be funded before the build starts.** Phase 0 will preflight both — see Phase 0 step "Verify both credit pools" below.
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
├── config.json                          # Saved Phase 0 — founder name, slug, geo scope, build start
├── webset-spec.json                     # Saved Phase 6g — input to create_webset (assembled in 6a-6f)
├── webset-response.json                 # Saved Phase 6h — full items + enrichments
├── sumble-jobs.json                     # Saved Phase 6j — Sumble + fallback job postings per company
├── lovelace-contacts.json               # Saved Phase 6k — LinkedIn results per company
├── founder-pick-research.json           # Saved Phase 6m — directed research for founder-named picks
├── CONTEXT.md                           # Generated Phase 4
├── data.js                              # Generated Phase 7 (overwrites template's). company name+domain also feed the Network/Contacts tabs
├── network-data.js / network-data.json  # ENGINE-generated AFTER Phase 10 (fetch-affinity-network.mjs, real Affinity). NEVER create/copy/fabricate it — absent during every skill phase; the Network+Contacts tabs degrade to a designed empty state without it
├── index.html                           # Phase 8c — the scroll walkthrough (entry page), copied VERBATIM from template/build.html (never edited per founder)
├── dashboard.html                       # The dashboard app (Phase 8; dark default + light toggle + All tab + score color-coding + NETWORK_DATA-driven Network & Contacts tabs)
├── middleware.js, vercel.json           # From template — edge Basic-Auth deploy gate + static config; keep as-is (engine sets FDI_DASHBOARD_PASSWORD)
├── build-data.js                        # Phase 8c — the cinematic's only founder-specific input (schema: template/build-data-template.js)
├── competitors.html / competitors-data.js # Phase 8d (OPTIONAL, only when config.json.modules includes "competitors") — standalone competitor teardown page + its window.COMPETITORS_DATA; absent on every default build
├── assets/                              # Phase 8c — FULL copy of template/assets (logos/ + primary-lockup.svg — used by the WALKTHROUGH; the dashboard topbar uses an inline mark)
├── assets/dashboard-preview.png         # Phase 8c — screenshot of THIS build's dashboard for the cinematic hero
├── build-summary.json                   # Phase 10 — modules map (ok|skipped|failed) the hub reads for per-module checkmarks
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
- `dashboard.html`, the dashboard — branded, signal-labeled, hiring-regex-tuned for this vertical, and shipping the v3 defaults (dark mode default + light toggle, score-quality color-coding, "All" tab as the default view, and the two data-driven relationship tabs — Network + Contacts — that ship empty-but-graceful and light up when the engine writes `network-data.js` post-build). You customize it as `index.html` in Phase 8, then Phase 8c renames it to `dashboard.html` and the walkthrough takes `index.html`. See "Dashboard visual defaults (v3)" before Phase 8.
- `BUILD_NOTES.md`, documents the structural choices you made
- `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json`, saved intermediates for reproducibility
- NOTE: `network-data.js` is NOT one of your outputs — the engine generates it from Affinity after your build finishes. Never create, copy (the template ships a fictional fixture), or fabricate it.
- the scroll walkthrough (Phase 8c, STANDARD) — `template/build.html` copied verbatim as the build's **`index.html`** + a generated `build-data.js` + `assets/`. Final flow every build ships: **index.html (walkthrough, two-beat founder opener) → dashboard.html**, all in the Ember design system. A separate landing page (Phase 8b) is optional and off by default.

This is an 11-phase build (Phase 0–10), plus the standard landing (Phase 8b) and cinematic (Phase 8c). Do not skip phases. Do not skip ahead. Each phase ends with a `git commit` so the build has clean version history.

---

## Modules (registry-driven)

A build is composed of **modules** — one per feature. The catalog is
`template/modules-registry.json` (single source; the hub menu, this skill, and
the engine all read it). Each module declares a `key`, `tier`, `default`, the
`artifacts` it produces, and its `generator` (`skill:<phase>` or
`engine:<script>`).

**Which modules to generate — read `config.json.modules`:**

- **`config.json` has NO `modules` field (or it's absent/empty) → generate every
  `default:true` module.** This is the DEFAULT and it is EXACTLY today's behavior.
  Backward-compat is sacred: an unset selection must never change what ships.
- **`config.json.modules` is a non-empty array of module keys → generate the
  `core` modules ALWAYS, plus only the `optional` modules whose key is in the
  array.** (`core` is never gated off — walkthrough + dashboard + their data ARE
  the product; a selection that omits them is ignored, they still generate.)

Only modules with a `skill:` generator are yours to gate here. Today that's:
`landing` (`skill:8b`, optional, `default:false`) and `competitors` (`skill:8d`,
optional, `default:false`) — both opt-in. Modules with an `engine:` generator
(e.g. `network` → `fetch-affinity-network.mjs`) are gated by the engine AFTER
your build, not by you; never generate their artifacts.

**Degrade, don't fail.** If an `optional` module is not selected, SKIP its phase
cleanly — the dashboard already renders the designed empty state for an absent
optional artifact (this is exactly how the Network/Contacts tabs behave when
`network-data.js` is absent). Never hard-fail a build because an optional module
was skipped. Only `core` modules hard-fail.

**Record what shipped.** At the end of Phase 10, after the self-check, write
`build-summary.json` at the repo root with a `modules` map of every registry
module you were responsible for → `"ok"` (generated), `"skipped"` (optional, not
selected), or `"failed"` (selected but its generation could not complete — the
module shipped its designed empty state instead of blocking the build). The
engine merges its own modules (e.g. `network`) into the same map post-build; the
hub renders per-module checkmarks from it. Example:
`{"modules": {"walkthrough": "ok", "dashboard": "ok", "landing": "skipped", "competitors": "ok"}}`.
(Leave a module out of the map entirely if it isn't yours — don't claim
`network`, the engine owns that one.)

Adding a NEW feature later = a `template/` folder or a new skill phase + one line
in `modules-registry.json`. Nothing else in this skill changes.

---

## Output style rules (apply to every generated output)

These rules apply to every piece of text you write into CONTEXT.md, data.js fields (subtitle, overview, gtm_thesis, sections, signal reasonings), BUILD_NOTES.md, and any other output. They do not apply to the user-facing chat messages you post during the build (those can stay conversational), but they do apply to anything that ships in the dashboard.

1. **Avoid em dashes — hard cap, not advisory.** The body-text em-dash count for any single company entry must NOT exceed **3** across subtitle + overview + section row values + tag tooltips + axis reasonings + signals[] bullets combined. Source titles in COMPANY_SOURCES are exempt (the `[Outlet] — [Specific topic]` separator is structural; see Rule #12). Use commas, periods, semicolons, parentheses, or rephrase. Em dashes feel AI-generated and read as written-by-essayist. Phase 7 self-check counts em dashes per company and fails the entry if it exceeds 3 — the entry is rewritten before moving on.
   - Bad (the Plural-build regression pattern, 3 em dashes in one Pain Points sentence): "Operating EKS at decision-engine scale across multiple regions compresses gross margin as Alloy scales the perpetual-KYC AI feature set — every additional model evaluation amplifies platform-engineering load on a still-mid-market headcount."
   - Better: "Operating EKS at decision-engine scale across multiple regions compresses gross margin as Alloy scales the perpetual-KYC AI feature set; every additional model evaluation amplifies platform-engineering load on a still-mid-market headcount."
   - Good (V1 BigPanda baseline): "BigPanda is the canonical AIOps reference for the BYOC thesis. Land is already executed; focus is on co-developing case study evidence."
   - (Reference build → Plural FDI 2026-05-06 audit found 28.7 em dashes/co body text vs Valar V1's 16.6/co — a 73% bloat regression that traced to Rule #1 being treated as advisory rather than enforced.)

2. **Geographic default: United States and Canada only**, unless the user specifies otherwise in Phase 0. This applies to the Webset, the curated company list, and any contact discovery. If a non-US/Canada company surfaces in the Webset, drop it during Phase 6i curation unless the user explicitly approved a wider geographic scope.

3. **No filler descriptors.** Avoid "industry-defining," "leading provider," "best-in-class," "innovative." Every adjective should carry a specific signal.

4. **Numeric ranges, not point estimates**, when uncertainty is real. "$2M–$5M annual inference spend" beats "$3.5M" if the underlying data is an estimate. Do not pretend to precision the data does not support.

5. **Cite or omit.** Every numeric or specific claim either has a source in `ROW_SOURCES` or is dropped. Wrong citation is worse than no citation.

6. **Final dashboard target: 9 or 10 companies (10 preferred; 9 if no 10th candidate clears the source-quality floor or self-check).** Founders walk through 3 companies max in any demo. Wide coverage hurts more than it helps; depth-per-company beats breadth. Don't ship 30 mixed-quality entries when 10 high-confidence entries do the job better. **Precedence:** if Phase 7 reaches the 10th slot and the candidate fails the 4-source floor, the antagonist consistency check, the durability test, or any self-check item that can't be repaired in one round of editing, drop to 9. Document the dropped slot in BUILD_NOTES.md (which company, why dropped, whether to revisit). Shipping 9 strong entries is always better than 10 with one weak.

7. **No placeholder text in shipped output.** "needs verification", "TBD", "unknown", "to be confirmed", "?", "[insert ...]", "lorem ipsum" — none of these ship. If a Webset enrichment returns a placeholder string, either fill the field with a defensible value (compute the estimate yourself, find the source, name the actual product) or omit the field entirely. (Reference build → F1 placeholder leak.)

8. **Subtitle ≤ 18 words, no parenthetical financial metadata.** The subtitle under the company name should land one signal. Don't stuff revenue, employee count, or founding year into parentheticals — that data goes in the Profile section. If the subtitle ends with "..." it's too long; rewrite shorter. See TEMPLATE_GUIDE Section 9.1.

9. **Pain Points framing: financial consequence first.** When writing the Pain Points row in Section 2 (whatever the vertical-named section is — `[SECTION_2_LABEL]`, e.g. "Inference Footprint" for inference, "Workflow Footprint" for workflow automation, "Capex Footprint" for industrials), the first sentence is CFO language (margin compression, COGS impact, gross-margin drag, opex pressure). Constraint enumeration is the second sentence onward. A field that reads "X; Y; Z" with semicolons is enumeration, not framing — rewrite. See TEMPLATE_GUIDE Section 9.8.

10. **Locked field sets per section** (do NOT add fields beyond these):
    - Profile (5 rows): Industry, Revenue, Employees, Cloud Provider, AI Maturity
    - `[SECTION_2_LABEL]` (4 rows): Use Cases, Current Stack, Pain Points, Estimated Spend. The label is chosen in Phase 5 per the founder's vertical ("Inference Footprint" for inference, "Workflow Footprint" for workflow automation, "Capex Footprint" for industrials greenfield, "Compliance Footprint" for fintech, etc.). The 4-row shape is fixed; only the label varies.
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

14. **Reference build is scaffolding, not content.** The skill carries Valar/inference examples (the reference build) to anchor patterns. Do not let those examples constrain your axes, persona templates, source choices, or wow-evidence shape for a different vertical. If the output sounds like Valar with names changed — same axis labels, same persona descriptions, same wow shape — you've under-delivered. The mandatory axes (Hiring, Opportunity) are reusable across all verticals; everything else (Section 2 label, founder-specific axes, hiring keyword regex, persona titles, wow-evidence shape, Tier-2 source mix) must derive from THIS founder's CONTEXT.md, not from the reference build. The single most reliable signal that a build went wrong is when the output reads as inference-vertical for a non-inference founder.

15. **Body-prose word caps — hard, enforced by Phase 7 self-check.** Long-form prose fields have hard word/sentence ceilings. Subagent-generated entries that exceed are rewritten before moving on. The caps reflect the Valar V1 hand-built baseline; bloat past these numbers reads as AI-essayist register, not founder-voiced.

    | Field | Word cap | Sentence cap | V1 baseline | Self-check |
    |---|---|---|---|---|
    | `subtitle` | 18 | 1 | 12-17 (avg ~15) | Already in Rule #8 |
    | `overview` | **80** | **4** | 64-90 (avg ~68) | Phase 7 self-check counts |
    | `gtm_thesis` | **75** | **3** | 17-63 (avg ~40) | Phase 7 self-check counts |
    | `opp_reason` | 50 | 2 | 22-30 | Phase 7 self-check counts |
    | `distress_reason` | 60 | 2-3 | 20-40 | Phase 7 self-check counts |
    | `residency_reason` | 90 | 3 (extra room for wow-evidence trace) | 25-50 | Phase 7 self-check counts |

    (Reference build → Plural FDI 2026-05-06 audit found gtm_thesis at 132w avg vs Valar V1's 40w — 3.3× bloat regression. Overview at 115w vs 68w. Length-cap enforcement was missing as an explicit rule.)

16. **No `(a)/(b)/(c)` enumeration in prose contexts.** This style tic is reserved for **`residency_reason` only**, where it traces the score-5 wow-evidence rubric (the `(a) named exec + (b) cited scale + (c) governance friction` shape). Do NOT use `(a)/(b)/(c)` enumeration in subtitle, overview, gtm_thesis, opp_reason, distress_reason, signals[] bullets, or section row values. If the build wants to enumerate three points in one of those fields, use a list (markdown-style hyphens or sentences) rather than the enumeration tic. Phase 7 self-check greps for `(a)` outside `residency_reason` and fails the entry. (Reference build → Plural FDI overview field had `(a)/(b)/(c)` templating in 6 of 10 entries — should have stayed in `residency_reason`.)

17. **`opp_reason` must acknowledge a hurdle when `signal_score` ≤ 4.** Honest weakness signals credibility. The Valar V1 Mastercard pattern is the model: `opp_reason` flags "Mastercard has deep in-house expertise, so Valar needs to demonstrate clear value beyond what their team has built" alongside the opportunity. When `signal_score` is 4 or below, the `opp_reason` text MUST contain at least one challenge-acknowledgment phrase: a hurdle, an in-house competing capability, a procurement obstacle, a competitor relationship, or a timing risk. Phase 7 self-check fails the entry if `signal_score ≤ 4` and `opp_reason` reads uniformly bullish. (Reference build → Plural FDI's `opp_reason` fields were uniformly positive across all 10 companies; honesty leaked into `distress_reason` instead of where it belonged.)

18. **Warmth copy tone — confident partnership, never self-deprecating.** The three founder-facing warmth strings (`narration.introWarmth`, `narration.finaleWarmth`, and the dashboard's `{{FEEDBACK_CARD_BODY}}`) frame the deliverable as a capable partner's strong first pass and the start of a working relationship — never an apology. **Banned:** "we gave it our best shot", "hopefully we got it right", "we tried our best", "sorry if we missed", or any phrasing that undersells the research. The feedback ask is explicit and two-sided (what they love AND where we can improve), and closes on collaboration ("make this even stronger together" spirit). The three strings play **distinct rhetorical roles** and must not repeat each other: `introWarmth` = "this is our deep dive into your world, excited to keep exploring it together" (opens the walkthrough); `finaleWarmth` = "this is the start of a conversation, looking forward to continuing the research together" (closes the walkthrough cinematic); `{{FEEDBACK_CARD_BODY}}` = the explicit feedback ask (the dashboard outro, where the founder has just seen the real intelligence). (Reference build → Frost Security, per Jason Gelman's 2026-07 feedback.) Verify `{{FEEDBACK_CARD_BODY}}` is stamped in the FINAL `dashboard.html` (post-rename), not just `index.html` (F15). Use the founder's FULL brand name across all three strings (F16).

19. **No investor-internal framing in shipped output.** The dashboard, walkthrough (`build-data.js`), and competitors page are FOUNDER-FACING. Never expose Primary's internal investment process in any shipped string: **banned** — "investment memo", "IC" / "investment committee", "dealflow", "term sheet", "cap table", "fundraise", and "diligence" *as OUR process* (a target company's own M&A "due diligence" quoted in cited news is fine — that is public evidence about the account). Frame from the founder's side: cite "your materials", "the docs you shared", or "your memo" — never "the Primary investment memo". Evidence-feed labels in `build-data.js` especially: a founder-provided doc is a "Founder brief" or "Company materials", not a "Primary investment memo". The D1 deploy-content gate (`sensitive-policy.json`) blocks a deploy on these, but keep the framing out at authorship — do not rely on the gate. (Reference → Lantern-2 2026-07 shipped "Primary investment memo" + "IC" into the walkthrough evidence feed and "We read the memo" as the ICP narration; both blocked the deploy.)

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
- Best practice is well-documented (the standard 4-axis Hiring + Opportunity + 2 founder-specific structure, the standard 3-segment Pipeline / Mid-Market / Enterprise scaffold, the standard locked field sets per Section, the source quality hierarchy)
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
| 5 | Signal axes | **Auto-proceed if the standard 4-axis pattern fits** (mandatory Hiring + Opportunity + 2 founder-specific axes derived from CONTEXT.md). Stop only if you need to deviate from the 4-axis structure (e.g., 5 or 6 axes, or fewer than 4) or the founder-specific axes can't be cleanly derived from CONTEXT.md. |
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
| 8b | Landing page | **Skip by default** (walkthrough is the entry). Build only on explicit ask. |
| 8c | Scroll walkthrough (entry) | **Auto-proceed.** template/build.html copied verbatim as index.html; build-data.js synthesized by subagent. No questions. |
| 9 | BUILD_NOTES.md | **Auto-proceed.** |
| 10 | Self-check | **Auto-proceed** unless a check fails, then surface the failure and ask whether to fix or ship anyway. |

**Target account count (scope/cost lever).** Every build carries a target curated-account count, default **10** (valid 3-30). Sources, in priority order: explicit dispatch input (`target_accounts` from the GTM Hub form / Slack), the user's ask at Phase 0, else default. Record it in `config.json` as `target_accounts` at Phase 0. It scales the paid scope:

- Phase 6d/6e: size `searchCount` so the Webset returns ~**1.5-2x the target** in strong matches (e.g. target 10 → pull ~15; target 25 → pull ~35-40). Enrichment cost scales linearly - keep the enrichment set identical, scale only the count.
- Phase 6i: curate to **exactly the target** (one fewer is fine if the last candidate is weak - N-1 strong beats N with a passenger).
- Phase 7/8c: `scan.curated`, hero stats, and shortlist copy all reflect the actual curated count - never hardcode 10.

**Theme color (accent).** Every build carries an accent theme, default **ember** (amber). Source: the `THEME_COLOR` env/shell var set by the engine workflow (decoded from the hub form's dropdown) — else `ember`. Valid keys are exactly those in `template/theme-presets.json` (`ember gold coral rose magenta violet indigo blue cyan teal emerald lime`); any unknown/absent value falls back to `ember`. Recorded in `config.json` as `theme_color` at Phase 0. Consumed in Phase 8 (dashboard `{{THEME_ACCENT_*}}`) and Phase 8c (walkthrough `founder.themeAccent`), both resolved from `theme-presets.json`. Only the accent recolors — the `--green*` ramp plus `--q-med` (the MED tier, which IS the accent, historically amber). `--q-high` (green, high tier), `--q-low` (red, low tier), `--purple` (links), and `--teal` (priority star) stay FIXED — the tier legend + info hierarchy must read the same across every theme.

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

**Step 0 — preflight: verify both Exa credit pools BEFORE Phase 0a.** Websets and basic Exa share the same dashboard but bill on separate pools (see Prerequisites). A build that runs out of basic-Exa credits mid-flight cannot complete Phase 6j (Sumble fetches) or Phase 6m (founder-pick research) and will produce uniform-1 Hiring scores (the F11 cascade). Catch this before kickoff.

```bash
# Probe Websets MCP — a single list call confirms credits + auth
# Use mcp__websets__list_websets with limit=1; expect a successful response with .data[].

# Probe basic Exa MCP — a single trivial search confirms credits + auth
# Use mcp__claude_ai_Exa__web_search_exa with query="hello world", numResults=1.
# A 402 indicates credits exhausted. A 401 indicates auth. Either fails the preflight.
```

If either probe returns 402: halt the build, surface the failure to the user with the link `dashboard.exa.ai/api-keys` and instructions to top up the offending pool. Do not proceed to 0a until both probes pass. (Reference build → F11 Hiring axis flatline traced to skipped preflight: basic-Exa 402 surfaced 30 minutes into a 6-hour build.)

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

# Save build config (geo scope, founder name) for later phases to read.
# founder_name = the FULL brand name as the founder writes it ("Frost Security",
# not "Frost"). It flows verbatim into {{PRODUCT_NAME}}, the walkthrough, and
# Slack/hub reporting. If the dispatch input is a short form, expand it from the
# docs in inputs/ before writing it here (F16).
#
# modules = which modules to build (see "Modules (registry-driven)"). FDI_MODULES
# is an optional csv of module keys from the hub menu / engine dispatch. Empty or
# unset → `[]` → every default:true module generates = today's exact behavior.
if [ -n "${FDI_MODULES:-}" ]; then
  MODULES_JSON=$(printf '%s' "$FDI_MODULES" | jq -R -c 'split(",") | map(gsub("^\\s+|\\s+$";"")) | map(select(length>0))')
else
  MODULES_JSON="[]"
fi
cat > config.json <<EOF
{
  "founder_name": "<Full Brand Name>",
  "slug": "$SLUG",
  "description": "$DESCRIPTION",
  "geo_scope": "$GEO_SCOPE",
  "theme_color": "${THEME_COLOR:-ember}",
  "modules": $MODULES_JSON,
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

# Save build config (geo scope, founder name) — Phase 6c will read this
if [ -n "${FDI_MODULES:-}" ]; then
  MODULES_JSON=$(printf '%s' "$FDI_MODULES" | jq -R -c 'split(",") | map(gsub("^\\s+|\\s+$";"")) | map(select(length>0))')
else
  MODULES_JSON="[]"
fi
cat > config.json <<EOF
{
  "founder_name": "<Company Name>",
  "slug": "$SLUG",
  "description": "$DESCRIPTION",
  "geo_scope": "$GEO_SCOPE",
  "theme_color": "${THEME_COLOR:-ember}",
  "modules": $MODULES_JSON,
  "build_started": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF

git init
git add inputs/ template/ config.json
git commit -m "Phase 0: working directory setup, template cloned, build config saved"
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

**Output: axis definitions, plus three downstream artifacts the rest of the build will reference.**

For each axis, document:
- **Axis name** (what shows in the dashboard UI)
- **What it measures** (one sentence)
- **Data sources** (where the underlying signal comes from, web scraping job sites for hiring, 10-Ks for opportunity, third-party press for compliance, etc.)
- **0-5 scoring rubric** (what does a 0 look like, what does a 5 look like, what's in between)
- **Webset enrichment column it maps to** (one or more)

**Three downstream artifacts (produce these now, they're consumed by Phase 6j and Phase 8):**

1. **`SECTION_2_LABEL`** — the vertical-named label for Section 2 of each company card. The 4-row shape (Use Cases, Current Stack, Pain Points, Estimated Spend) is fixed; only the label changes by vertical. Examples: "Inference Footprint" for inference, "Workflow Footprint" for workflow automation, "Capex Footprint" for industrials greenfield, "Compliance Footprint" for fintech regtech, "Trial Footprint" for biotech. Pick one and use it consistently from this point forward.

2. **`HIRING_KEYWORD_REGEX`** — JavaScript-compatible regex for the Hiring axis. Generated from CONTEXT.md's pain language + Phase 5 axis names + antagonist warnings (exclude antagonist-role keywords; for an inference founder don't include "data scientist" because Wiggin flagged ML eng as antagonist; for an industrials founder don't include "controls engineer" if the founder named them as threatened by external automation vendors). The regex should match leading-indicator job titles + skills that signal a company is staffing toward the pain the founder solves. Test the regex against 3 known-prospect careers pages mentally before locking it in. Phase 6j (Sumble fetch) and Phase 8 (index.html update) both consume this regex; produce it once here.

   Examples by vertical (use as inspiration, not template):
   - Inference / AI infra: `/inference|llm |triton|tensorrt|sglang|vllm|model serving|ml platform|gen[ ]?ai platform|kernel/i`
   - Industrials predictive-maintenance: `/predictive maintenance|condition monitoring|industrial ai|reliability engineering|asset performance|opc-ua|industrial iot/i`
   - Healthcare RCM: `/revenue cycle|denial management|prior authorization|claims operations|coding|RCM/i`
   - Fintech infra: `/payments engineering|fraud engineering|compliance engineering|banking platform|payment rail/i`
   - Cybersecurity: `/detection engineering|security platform|security operations|threat detection|SIEM|SOC/i`
   - Biotech / medtech: `/clinical operations|regulatory affairs|bioinformatics|process development|GMP|computational biology/i`
   - Consumer / retail: `/personalization|merchandising|recommendation systems|search relevance|consumer data platform/i`

3. **`WOW_EVIDENCE_SHAPE`** — a one-line description of what cited evidence in the wow signal's shape would look like (per CONTEXT.md's "Wow Signal" section from Phase 2). This is consumed by Phase 7 step 12 (the score-5 evidence requirement). Examples: "cited tried-and-blocked vendor (Fireworks/Together/Baseten/Modal evaluated, none reached production due to security)" for Valar; "cited legacy-system reference (10-K mentions DCS end-of-life cost or specific platform name)" for industrials; "cited regulatory milestone (FDA submission with named alternative pending validation)" for biotech; "cited CFO earnings-call quote about a named compliance gap" for fintech.

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

**Auto-proceed if the standard 4-axis pattern fits cleanly.** Default structure is Hiring + Opportunity + 2 founder-specific axes (derived from CONTEXT.md), with the standard 3-segment scaffold (Pipeline / Mid-Market / Enterprise). When the 2 founder-specific axes drop out of CONTEXT.md's ICP Qualifier + lookalike anchors + wow signal without ambiguity, post the axis plan as visibility and move to Phase 6.

**Stop and ask** if you need to deviate from the 4-axis structure (e.g., 5 or 6 axes for unusually multi-dimensional ICPs), or if the founder-specific axes can't be cleanly derived from CONTEXT.md and require user judgment — that's a significant decision worth a checkpoint.

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
| `[trade-press]` | Vertical-specific industry press | "Featured in Industrial Maintenance / Endpoints News / Retail Dive coverage of the founder's pain area" |
| `[analyst]` | Forrester / Gartner / IDC / IDC / vertical analyst notes | "Cited in Gartner Industrial / Forrester Healthcare / IDC Financial Services research" |
| `[founder-anchor]` | Imported founder CSV / named pipeline | "Listed in founder's outbound CSV" (uses Webset `scope` parameter, not criteria) |

**Choose tags by vertical's actual source landscape.** Tech-forward verticals (inference, data infra, dev tools, cyber): `[engineering]` is rich. Industrials / consumer / healthcare delivery / non-tech enterprise: `[engineering]` is thin; `[trade-press]` and `[analyst]` carry the weight. Don't force `[engineering]` if the vertical doesn't publish engineering content.

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

Hiring axis: NOT a Webset enrichment — fetched from Sumble (+ fallback ladder) in Phase 6j. Do not include in this Webset spec.

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

Poll `get_webset` every 30 seconds (or use `ScheduleWakeup` with delaySeconds=270 to stay in cache window) until status is `idle`. The first poll should wait at least 10 seconds after `create_webset` per Exa's recommendation.

**Early-cancel triggers — don't wait it out.** Webset has good criterion-pass-rate telemetry; use it. The skill's failure mode is letting a slow search drain a too-restrictive criterion until the user is 30+ minutes deep with thin returns. Cancel and re-fire with softened criteria when:

- **Single-criterion pass rate < 10% within first 50-100 analyzed items.** This means the criterion is too narrow; the Webset is searching the wrong corpus or rejecting too aggressively. Cancel the search (`cancel_search`), soften the offending criterion (typically: drop a strict numeric threshold like "50+ clusters" to a softer proxy like "Kubernetes in production at meaningful scale"), keep everything else, and re-fire.
- **Pass rate is *decreasing* across consecutive polls.** A criterion at 12% → 8% → 6.5% as analyzed_count climbs from 100 → 200 → 250 means the search has drained the obvious matches and is now scraping bottom. Cancel; the remaining returns will be lower-quality than what's already found.
- **timeLeft estimate is increasing** (Webset's own ETA is growing, not shrinking). The search is hitting analysis-difficult corpora; what's already returned is the high-water mark.
- **Found count stalls for two consecutive polls** (no new items in 5+ minutes despite ongoing analysis). Cancel and accept partial returns.

In any of these cases, **cancel + accept partial returns** is usually better than re-fire-with-softened. The partial returns are quality matches; the saved findings + intersection-mining of any prior persona Websets in the workspace + Phase 6m founder-pick research often yields enough triangulation for the curated 10 list. (Reference build → Plural FDI 2026-05-06: K8s-50+-clusters criterion dropped from 11.9% → 8.0% → 6.5% pass rate over 16 minutes; canceling at 41% complete with 4 returns turned out to be net-positive — the 4 returned companies were all axis-4 score-5 quality, and the curated list filled out from intersection-mining of prior persona Websets.)

If a Webset reaches `idle` cleanly but takes >15 minutes anyway, that's a slow-but-successful run; keep it. The early-cancel rule is for *thin returns + degrading pass rate* specifically.

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

**Step 0: Mine the Webset's vertical-specific role-evidence enrichment FIRST.** Before the external fallback ladder (Sumble → careers → LinkedIn → ATS), parse the Webset's already-paid-for role-evidence enrichment for named role-bearers in vertical-relevant titles. Every FDI Webset spec includes a "Pain Evidence" or "Operational Posture" enrichment that surfaces named existing employees in target roles (the per-vertical phrasing was set in Phase 6b). For each curated company:

1. Read the relevant enrichment field from `webset-response.json`.
2. Extract names + titles + LinkedIn URLs of role-bearers cited there.
3. Filter each title against the per-build `HIRING_KEYWORD_REGEX` (Phase 5 artifact #2). Keep only matches.
4. Write each match to `JOB_LISTINGS[<company>]` as: `{title: <exact title>, team: "Verified Role", url: <linkedin>, date: "Verified Active <year>"}`. Mark `team: "Verified Role"` (not "Posting") so downstream code distinguishes verified-headcount evidence from active open reqs.

Step 0 produces hiring evidence that doesn't depend on basic-Exa MCP credits — it uses data the Webset already paid to surface. Steps 1-5 (the original external fallback ladder) still run after Step 0 to layer in active open-req data. The two sources stack: a company can have both Step-0 verified roles AND open reqs, with the regex-match count summing to the score.

**Why Step 0 first:** mitigates the F11 cascade (basic-Exa 402 → empty fallback ladder → uniform-1 Hiring scores). Step 0's data is always available because Websets is on a separate credit pool from basic Exa.

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

Filter to roles relevant to the founder's pain. Use the `HIRING_KEYWORD_REGEX` you produced in Phase 5 (artifact #2). Keep the regex narrow: each kept role should be a leading indicator that the company is staffing toward the pain the founder solves. Don't keep generic "Software Engineer" postings unless they specifically reference the vertical-relevant skill stack.

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
2. **`careers.<domain>` direct fetch.** `Exa:web_fetch_exa` against `careers.<company>.com` or `<company>.com/careers`. Filter results by the `HIRING_KEYWORD_REGEX` from Phase 5 (artifact #2). Capture title + URL + posting date if visible.
3. **LinkedIn job search by company.** `Lovelace:search_linkedin_profiles` won't help here, but `Exa:web_search_exa` with query `linkedin.com/jobs "<Company>" "<role keyword>"` (e.g. `"Capital One" "ML Platform"`) usually surfaces 1-3 specific reqs. Take only ones whose company field exactly matches (LinkedIn returns adjacent companies sometimes).
4. **Greenhouse / Lever / Ashby boards** if the company uses them. `Exa:web_search_exa` with `boards.greenhouse.io/<slug>` or `jobs.lever.co/<slug>`.
5. **Last resort: empty.** If steps 1-4 all return zero relevant roles, set `jobs: []` and note `fallback_attempted: ["sumble", "careers", "linkedin", "ats"]` in `sumble-jobs.json` so Phase 7 knows the gap is researched, not skipped.

The hard floor is **≥1 verified job per company in tier='high'**. If a tier='high' company has zero jobs after the full ladder, drop it to tier='med' before Phase 7. (Reference build → F7 JOB_LISTINGS empty, where 27 of 30 were empty because the build accepted Webset's NULL enrichment without working the fallback.)

**URL specificity rule (per-job).** Each job posting's `url` field MUST be the **specific job-req URL**, not the company's careers landing page. Valar V1 baseline: `https://www.capitalonecareers.com/job/mclean/sr-distinguished-machine-learning-engineer-remote-eligible/1732/93650794080` (specific req ID). Bad pattern: `https://www.finastra.com/about/careers` (generic landing). When a Sumble fetch returns a job with its detail-page URL, keep it; when the fallback ladder yields a careers index URL, return to step 3 (LinkedIn job search by exact company name) and capture the specific posting URL there. Phase 7 self-check fails any JOB_LISTINGS entry where `url` matches `/careers/?$` or `/about/careers/?$`. (Reference build → Plural FDI 2026-05-06: 4 of 10 companies had generic careers-page URLs — a credibility leak readers can spot in one click.)

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

For each LinkedIn profile returned, cross-reference against the Primary network connections list in CONTEXT.md (if present). For each match:

1. Tag `primary_connection: "<teammate name>"` in the contact's CONTACT_MAP entry (research signal only — see below)
2. In the contact's `note` field (if you add one), specify the connection: "Warm via [Teammate]: [shared context, same school, prior coworkers, mutual connection X]"

Warm intros are high-signal, but the dashboard's **Network + Contacts tabs no longer render CONTACT_MAP/Lovelace-derived connections** — they show ONLY real Affinity warm paths, which the engine writes to `network-data.js` after the build. So surface warm-intro texture in **prose** (`gtm_thesis` when a company has multiple warm paths; CONTACT_MAP `note` fields), NEVER by inventing connection edges. Keep `CONTACT_MAP[].connections` empty (no synthesized relationships).

**Sequencing:**

For a 10-company curated list, that's potentially 20 Lovelace calls (2 per company: buyer + champion). Sequence intelligently, high-tier companies first, low-tier last. Stop early if you hit rate limits or low signal.

**Save to disk:**

Once contact discovery is done, save the aggregated results so Phase 7 can populate CONTACT_MAP from disk (not from re-calling Lovelace):

```bash
# Save contacts as lovelace-contacts.json keyed by company name
# {"BigPanda": [{name, title, linkedin, primary_connection}, ...], "Varonis": [...], ...}

git add lovelace-contacts.json
git commit -m "Phase 6k: Lovelace contacts gathered (N profiles across M companies)"
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

**Step 2: Per company, run a directed research pass. Query budget scales with backlog size.**

Per-company query budget (default vs. scale-down):
- **Default — 4-5 queries per company**, when ≤5 founder-named picks need research. This is the canonical depth and produces 4+ Tier-2 sources reliably.
- **Scale-down — 2-3 highest-leverage queries per company**, when **>5** founder-named picks need research. The full 4-5 query pattern × 6+ companies blows the Exa budget (~30 queries) and burns context for marginal gains. When scaled-down, prioritize queries 1, 3, and 4 below (primary record + leadership + earnings/strategy press); skip queries 2 and 5 unless the company is a tech-forward vertical (engineering blog query) or a regulated vertical with explicit wow-shape evidence to hunt for (tried-and-blocked query). Phase 7 supplementary fetches can fill in any gap during data.js population. (Reference build → Plural FDI 2026-05-06 had 6 founder-named picks needing research; full 4-5 query pattern × 6 = 24-30 queries was reduced to 2-3 leveraged queries × 6 = ~15 queries with no measurable quality loss.)

The 5 canonical queries (run in parallel):

1. `"<Company> 10-K SEC EDGAR"` — gets the SEC filing for public companies. For private, swap to `"<Company> latest funding round" OR "<Company> annual revenue"`.
2. `"<Company> AI inference engineering blog"` — surfaces engineering content. Add `engineering.<company>.com` to the query if the company is known to host one. (Skip when scaled-down for non-tech-forward verticals.)
3. `"<Company> Chief AI Officer" OR "<Company> VP Platform Engineering"` — leadership / champion personas.
4. `"<Company> earnings call AI infrastructure"` for public companies, OR `"<Company> AI strategy"` press for private.

For regulated-vertical companies (banks, insurers, healthcare), add a 5th query targeting the wow signal:

5. `"<Company> Fireworks OR Together OR Baseten OR Modal OR Anyscale failed OR blocked OR security OR compliance"` — surfaces tried-and-blocked evidence (the Data Residency axis 5 requirement). Adapt vendor list to the founder's vertical when the wow shape is non-inference (e.g., for K8s-management vertical: "Tanzu Broadcom failed OR migrated OR EOL OR Rancher OR OpenShift cost OR sprawl"). (Skip when scaled-down for non-regulated verticals.)

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

**Delegate this phase to a Task subagent.** Phase 7 is the heaviest phase by far (10 companies × ~150 lines of data.js per entry × the 15 self-check items). The 1.5MB `webset-response.json` plus `founder-pick-research.json` plus `lovelace-contacts.json` plus `sumble-jobs.json` plus CONTEXT.md plus the template files don't all fit in the main thread's context efficiently. Spawn a `general-purpose` Task subagent and pass it:

- Working directory path (`~/fdi/<slug>/`)
- The 5 Phase 5 artifacts (`SECTION_2_LABEL`, `HIRING_KEYWORD_REGEX`, `WOW_EVIDENCE_SHAPE`, the 4 axis definitions, the segment structure)
- The curated 10-company list from Phase 6i
- The list of all 13 Phase 7 self-check items it must pass
- The instruction to commit `data.js` when done (not push)

The subagent reads `webset-response.json`, `founder-pick-research.json`, `lovelace-contacts.json`, `sumble-jobs.json`, CONTEXT.md, `template/TEMPLATE_GUIDE.md`, and `template/data.js` itself. It builds entries one company at a time, runs the 15 self-checks per entry, and writes the final data.js to disk. The main thread doesn't load any of those files; it just receives the subagent's status report at the end.

Failure recovery: if the subagent reports issues (e.g., 2 companies couldn't reach the 4-source floor even after the fallback ladder), the main thread decides whether to drop those companies, accept the lower tier, or pause for user input.

The rest of this phase (steps 1-13 below) is the **prompt to the subagent**, not instructions to the main thread.

---

**Subagent prompt begins here.**

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

2. **Break composite ties on the shortlist.** Whole-point 0-5 axes tie easily (17/20 = 85 five times on a real build - looks fabricated to a founder). After computing composites, if 2+ companies tie, add a `composite: <int>` override per tied company (the dashboard scorer honors it, clamped 0-100). Rank the tied group by a total order so it's reproducible: (1) strongest cited immaturity/wow artifact highest (primary/first-party citation beats inferred), (2) larger cited scale, (3) more total sources, (4) alphabetical as the final fallback. Overrides must be **collision-free and order-preserving across the whole board**: pick distinct integers strictly between the nearest distinct composites above and below the tied cluster so none equals another account's score and none leapfrogs a higher account or drops under a lower one (stay inside the tier, ~+/-3; if the integer gap is too tight for the count, widen it by also overriding the bracketing non-tied accounts). Preserve axis scores unchanged, carry the same numbers into `build-data.js` `companies[].score` in Phase 8c, and note the tiebreak rationale in BUILD_NOTES. Use this for genuine ties only - never to rescore a non-tied account.

3. **Set `tier` to the value the dashboard will compute: `'high'` if the company's signal is >= 75, else `'med'`.** `tier` is DERIVED, not editorial. The dashboard overwrites it on load (`co.tier = co._signal>=75?'high':'med'` — two-tone by design, so `'low'` never renders), which means a hand-picked tier is silently discarded and only survives as a pre-init fallback. Storing a tier that disagrees with the signal is a real defect: it fails the F2 consistency gate, and the data file then contradicts what the site shows. (This bit the NewCo golden build — 4 companies computing to exactly 75 were stored `med`/`low` while the dashboard rendered them High.)
   Compute the signal the same way the dashboard does, using **this build's** axis weights from Phase 5 (they are NOT always equal 0.25) and honoring any `composite` override, then set `tier` from it.
   **Shape the high/med mix through the axis scores and curation, never by relabeling `tier`.** With the 10-company target aim for roughly **6 high / 4 med** — if everything lands `high`, the tier carries no information, so revisit the axis scoring for the accounts with the least verifiable enrichment rather than forcing the label. The same applies wherever this doc says to "drop the company's tier from `high` to `med`" as a source-floor escalation: dropping the label alone has no effect at runtime — lower the axis scores that earned the signal, or drop the company.

4. **Write `subtitle` in V1 pattern**, *[what the company is], [why-they-fit-the-founder phrase], [founder relationship status]*. One sentence, dense, signal-rich. See TEMPLATE_GUIDE.md Section 9.1.

5. **Write `overview` in V1 pattern**, 3-5 sentences that name the company's position in the founder's market story, specify the data sensitivity in concrete terms (not abstract), and connect the company to a category-level reference. See Section 9.2.

5. **Fill the 3 sections** (Profile / `[SECTION_2_LABEL]` / GTM Strategy) directly from Webset enrichments. The `[SECTION_2_LABEL]` was chosen in Phase 5 per the founder's vertical. Locked field sets — do NOT add fields beyond these:
   - **Profile (5 rows exactly)**: Industry, Revenue, Employees, Cloud Provider, AI Maturity. Do not add Founded, Headquarters, "Valar Status" (or any "[Founder] Status"), Stage, ICP Tier, or Business Type — those duplicate information shown elsewhere. The relationship status lives in the `tags` array as a brand-color chip, not as a profile row. See Section 9.8.
   - **`[SECTION_2_LABEL]` (4 rows exactly)**: Use Cases, Current Stack, Pain Points, Estimated Spend. Label per Phase 5.
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

9. **Populate `CONTACT_MAP`** with platform/infrastructure leadership keyed exactly to `SEGMENTS[].companies[].name` (character-for-character match including parentheses). Read from `lovelace-contacts.json`. Persona discipline reflects founder antagonist warnings, exclude personas the founder has flagged. **Keep `CONTACT_MAP[].connections` EMPTY** — the UI no longer synthesizes warm intros from `PRIMARY_TEAM`, and the Network/Contacts tabs render only real Affinity paths (engine-written `network-data.js`). Never invent connection edges. See Section 9.10.

   **9a. `domain` is a join key, not just a favicon.** Every `SEGMENTS[].companies[].domain` must be the company's canonical registrable domain (no `www`, no path, no product/marketing subdomain — e.g. `acme.com`, not `www.acme.com/product` or `app.acme.com`). The engine's Affinity network step (`fetch-affinity-network.mjs`) resolves each company by normalized domain; a wrong/missing domain silently drops that account from BOTH the Network and Contacts tabs. Also keep `SEGMENTS[].companies[].name` stable — the tabs join it EXACTLY (character-for-character) to attach display metadata (subtitle/category/favicon).

10. **Populate `JOB_LISTINGS`** from `sumble-jobs.json` (Phase 6j). Each company's `jobs[]` array maps to JOB_LISTINGS entries with title, team, location, URL. If a company has empty jobs in Sumble data, leave its JOB_LISTINGS entry empty rather than backfilling with weaker data.

11. **Populate `COMPANY_SOURCES`** from `webset-response.json`, `founder-pick-research.json` (Phase 6m results for founder-named picks), AND targeted web fetches. Source list quality is what readers use to judge the rest of the dashboard. Counts and rules:
    - **Target: 6 sources per company** (V1 averages 6; reference build → F2 source thinness).
    - **Hard floor: 4 sources.** Below 4 is unacceptable. Escalation order: (1) fetch the missing tier(s) yourself — at most 3 extra Exa fetches before giving up; (2) drop the company's tier from `high` to `med` if it still keeps the entry meaningful; (3) **drop the company entirely and ship 9 entries** per Output Style Rule #6 — document in BUILD_NOTES.md. Do NOT reopen Phase 6i mid-Phase-7 to swap in a fresh candidate; that path leads to incoherent curation.
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
- **`[SECTION_2_LABEL]` is exactly 4 rows.** Use Cases, Current Stack, Pain Points, Estimated Spend. The label was chosen in Phase 5 per vertical (e.g., "Inference Footprint", "Workflow Footprint", "Capex Footprint"). Verify the label is consistent across all 10 entries — don't mix labels.
- **Pain Points leads with financial consequence.** First sentence of Pain Points should be CFO language (margin, cost, COGS, opex, cash). Constraint enumeration is the second sentence onward, not the first. If the field reads as "X; Y; Z; W" with semicolons, rewrite — that's enumeration, not framing.
- **No banned tag values.** Search the `tags[]` array for "Stage-1 ICP", "Stage-2 ICP", "Pipeline", "Target", "ICP". If any are present, replace with product names, technical stack, constraints, or relationship status.
- **GTM thesis swap test.** Strip the company name from the `gtm_thesis`. Could you swap any other company's name in and have it still make sense? If yes, rewrite — the thesis isn't specific enough.
- **GTM thesis durability test.** Read the `gtm_thesis` and ask: if every named individual in this thesis left their job tomorrow, would the thesis still hold? Specifically scan for: named buyers ("John Morgan"), named champions ("Vivek Gupta"), named warm-intro paths ("via Alex"), comparative claims tied to personnel ("highest-warmth account"), specific role+name combinations ("EVP Chief Scientist Prem Natarajan"). If any are present, move them to CONTACT_MAP and replace with role types in the thesis. Buyer/Champion in the thesis are personas ("Platform Engineering leadership"), not humans.
- **Durability regex check (target-company names).** Run a regex over `gtm_thesis` matching `\b[A-Z][a-z]+ [A-Z][a-z]+\b` (capitalized two-word names). For every match, verify whether the matched string is in `PRIMARY_TEAM` (allowed reference, but should not appear in thesis), is a firm/fund name (e.g., "Stephens Group", "Snow Phipps" — allowed), or is a target-company individual (FAIL). Any name that is neither a Primary teammate nor a recognized firm/fund triggers entry rewrite. The Lantern May 6 build slipped "Brian Schlise" (incoming President at APR Supply) and "Marco Schooley" (new EVP at Kele) through the prior subjective check; this regex catches both. Replacement pattern: target-company exec names → role descriptor ("incoming President", "new EVP Strategy & Transformation"), then move the name to `CONTACT_MAP`. (Reference build → F4 personnel-fragile thesis recurrence: Lantern May 6 audit found 2 of 10 entries violated despite the prior check being in the skill.)
- **Antagonist persona consistency check.** If the gtm_thesis ends with `**NOT [persona]**` (e.g., **NOT** ML engineering, **NOT** security/governance, **NOT** controls engineering, **NOT** clinical operations — whatever persona this build's antagonist warning names), grep the company's CONTACT_MAP entries for matching persona keywords. The titles flagged in NOT must NOT appear as recommended champions. Worked example: gtm_thesis says **NOT** Technology Governance team → CONTACT_MAP cannot list a contact whose title contains "Technology Governance" as `type: 'business'` champion. If a match exists, drop the contact, demote them to a non-champion `note`-only entry, or rewrite the antagonist callout. (Reference build → F5 antagonist contradiction.)
- **Em-dash count ≤ 3 per entry body text** (per Rule #1, hardened). Count em dashes across subtitle + overview + gtm_thesis + tag tooltips + all section row values + axis reasonings + signals[] bullets. EXCLUDES COMPANY_SOURCES titles (`[Outlet] — [Specific topic]` separators are structural and exempt). Implementation: split the entry into the body fields, run a single regex count, fail at >3. If higher, rephrase using commas, periods, semicolons, or parentheses BEFORE writing the next company.
- **Word caps per Rule #15.** Count words for each of these fields: `subtitle` ≤18, `overview` ≤80, `gtm_thesis` ≤75, `opp_reason` ≤50, `distress_reason` ≤60, `residency_reason` ≤90. If any field exceeds, rewrite to the cap before moving on. The Plural-build regression had gtm_thesis at 132w avg (3.3× the V1 baseline) — this self-check is the enforcement that prevents recurrence.
- **`(a)/(b)/(c)` enumeration only allowed in `residency_reason`** (per Rule #16). Grep the entry's overview, subtitle, gtm_thesis, opp_reason, distress_reason, signals[] and section row values for `(a)`. If found in any field other than `residency_reason`, rewrite as a list or sentence sequence. The enumeration tic is reserved for the wow-evidence rubric trace, where it earns its space.
- **`opp_reason` challenge acknowledgment when `signal_score ≤ 4`** (per Rule #17). Read the entry's `opp_reason` value. If `signal_score` is 4 or below, the text MUST contain at least one challenge phrase: a hurdle, in-house competing capability, procurement obstacle, competitor relationship, timing risk, or honest gap. If `opp_reason` reads uniformly bullish without any caveat, append a 1-sentence challenge phrase grounded in the actual research. (Valar V1 Mastercard pattern: "Mastercard has deep in-house expertise, so [Founder] needs to demonstrate clear value beyond what their team has built.")
- **Source citation density.** Count cited rows in Profile + `[SECTION_2_LABEL]`. V1 averages 5 of ~10. If your entry has fewer than 4 of 9 cited, you missed URL extraction in step 13 — go back and parse the Webset enrichment text more carefully.
- **COMPANY_SOURCES count.** Open the entry's source list. Count it. Target is 6 (V1 average). Hard floor is 4 — under 4 means escalate (extra Exa fetches for missing tier, or drop the company's tier, or swap company). 4-5 is acceptable if Tier 1 / Tier 2 quality is present, but try once more to reach 6 first. (Reference build → F2 source thinness.)
- **No PR-aggregator-only sourcing.** If COMPANY_SOURCES is anchored on BusinessWire / PRNewswire / GlobeNewswire press releases without a Tier 1 (SEC) or Tier 2 (engineering blog) source alongside, the source list is too weak. Find the trade-press follow-up or the underlying primary record.
- **Source titles describe content.** Each entry in COMPANY_SOURCES should follow `[Outlet] — [Specific topic]` format. Bare outlet names ("Datadog Blog") or bare article titles ("Celonis AI copilot") fail the test. See TEMPLATE_GUIDE Section 9.12.
- **Generic test.** If you couldn't tell the entry apart from another company in the same segment, the patterns aren't landing.

If a company entry fails any check, fix before adding the next.

**After all 10 entries are written, before commit — global tier-distribution check:**

- Count `tier` values across all 10 companies. The skill targets approximately **5 high / 4 med / 1 low** for honest signal discrimination. If everything is `'high'`, tiers carry no information. **The check fails if the build ships zero `tier='low'` companies.** When this happens, demote the weakest-evidence entry to `'low'` (typically the company where K8s scale is inferred rather than primary-cited, or where the founder-specific axis caps at 3 rather than 4-5). Document the demoted slot in BUILD_NOTES.md § 9 score-distribution. Distribution-hard floor: at most 7 of 10 may be `'high'`. (Reference build → Plural FDI shipped 7 high / 3 med / 0 low — over-graded; subagent ignored the soft-target.)

**After all 10 entries are written, before commit — axis-uniformity check:**

- For each axis (`signal_score`, `competitive_distress`, `data_residency`, plus the runtime-computed Hiring sub-score from `JOB_LISTINGS`), count the most-frequent value across the 10 entries. If any single axis has an identical score in **8 or more (≥80%) of the 10 companies**, the axis is flatlined and carries zero discriminating signal. **The check fails the build.** The subagent must investigate: either the data source for that axis is missing/empty (e.g., F11 — JOB_LISTINGS empty cascade for Hiring), or the rubric is being misapplied uniformly. Fix the underlying issue and rescore before commit. Surface the offending axis + value in the self-check JSON's `global.axis_uniformity` block. (Reference build → F11 Hiring axis flatline.)

**After all 10 entries are written — emit per-company self-check JSON:**

The Phase 7 subagent must emit a single JSON object capturing the self-check status for each company, so the main thread can verify late entries (companies 7-10) didn't get the rule-decay treatment that early entries (1-3) caught. Format:

```json
{
  "self_checks": [
    {
      "company": "JPMorgan Chase",
      "subtitle_words": 16,
      "overview_words": 78,
      "gtm_thesis_words": 73,
      "opp_reason_words": 47,
      "em_dash_body_count": 2,
      "abc_enumeration_outside_residency": false,
      "opp_reason_has_challenge_ack": true,
      "row_sources_cited": 9,
      "company_sources_count": 7,
      "tier": "high",
      "all_passed": true,
      "issues": []
    },
    ...
  ],
  "global": {
    "tier_distribution": {"high": 5, "med": 4, "low": 1},
    "tier_check_passed": true,
    "axis_uniformity": {
      "signal_score": {"max_frequency": 4, "passed": true},
      "competitive_distress": {"max_frequency": 3, "passed": true},
      "data_residency": {"max_frequency": 5, "passed": true},
      "hiring_sub_score": {"max_frequency": 5, "passed": true},
      "axis_uniformity_check_passed": true
    }
  }
}
```

`axis_uniformity_check_passed` is `false` if any single axis has `max_frequency >= 8` (≥80% of the 10 companies share the same score). Reject the build when false; fix the underlying cause before commit.

If `all_passed: false` for any entry, the subagent must fix the entry before commit. The main thread will pretty-print this JSON in BUILD_NOTES.md § 9 and reject the build if `tier_check_passed: false`. (Reference build → OPEN_QUESTIONS #2 "subagent rule decay" — Plural build's gtm_thesis bloat in companies 1-10 was undetected because no per-company self-check artifact was emitted.)

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

### Dashboard visual defaults (v3 — baked into the template; do not regress)

These are no longer optional polish, and they are **already baked into `alexg207/fdi-template` `index.html`.** Your Phase-8 job is to NOT regress them, not to port them in — and any fix belongs upstream in the template, never per build. (Do NOT copy from `~/fdi/lantern-auto/dashboard.html` — it predates the Network/Contacts tabs + the font overhaul and copying it would regress a build.) If a cloned `template/` somehow predates v3 (missing the Network/Contacts tabs or still using JetBrains Mono in the dashboard), re-clone the template rather than hand-porting.

**1. Dark mode default + light/dark toggle.** Ship dark by default with a working light toggle.
- Tokenize ALL color as CSS custom properties on `:root` (dark values) plus a `:root[data-theme="light"]{...}` override block. Never hard-code a hex/hsl outside the token set.
- Warm palette, not blue-gray. Dark: `--bg:hsl(28 14% 7%)`, `--surface:hsl(30 13% 11%)`, `--text:hsl(40 30% 95%)`; single founder accent (Lantern used amber `hsl(36 92% 50%)` — derive the accent from the founder's brand).
- No-FOUC head script (runs before paint) + global toggle, verbatim-portable (replace `<slug>` with the founder slug, e.g. `{{PRODUCT_SLUG}}`):
  ```html
  <script>(function(){try{if(localStorage.getItem("<slug>-theme")==="light")document.documentElement.setAttribute("data-theme","light");}catch(e){}})();
  function toggleTheme(){var h=document.documentElement;if(h.getAttribute("data-theme")==="light"){h.removeAttribute("data-theme");try{localStorage.setItem("<slug>-theme","dark")}catch(e){}}else{h.setAttribute("data-theme","light");try{localStorage.setItem("<slug>-theme","light")}catch(e){}}}</script>
  ```
- Sun/moon toggle button in the topbar (`onclick="toggleTheme()"`, `aria-label="Toggle light or dark mode"`).
- Both themes must pass WCAG AA contrast on every text/background pair. Light theme is a full re-map, not an inversion: flip `-dark`/`-deep` accent text variants to LIGHTER shades in dark mode so they stay legible on dark surfaces, and disable dark-only glow / `body::before` effects under `[data-theme="light"]`.

**2. Score-quality color coding (NO RED by default).** Color the signal bar, signal number, tier chip, and tier bar by score bucket so chip color, bar fill, and the number always agree.
- Two-tone: green = best, amber = below. Tokens: `--q-high:hsl(150 58% 52%)` / `--q-med:hsl(38 95% 56%)` plus `-d` darker text variants for the number.
- Derive the tier FROM the score so they can never disagree: `co.tier = co._signal>=75?'high':'med';`
- **NO RED.** Every curated target is a good signal; a red "bad" tier on a hand-picked account reads wrong. A `--q-low` (red) token may sit in the stylesheet for a future 3-tier mode, but the default is two-tone green/amber. Only add a low/red tier if the founder explicitly wants to flag weak accounts.

**3. "All" tab is the default view.** Entering the dashboard must show every company in one signal-ranked scrollable grid, not 3 segment cards (the "enter and see 3 cards" flatness Lantern fixed).
- `state.tab` defaults to `'all'`. Add an "All" tab first in the nav with a live count (`tab-count-all`).
- `getCurrentSeg(){if(state.tab==='all')return {id:'all',companies:getAllCompaniesFlat()};return SEGMENTS.find(function(s){return s.id===state.tab;});}`
- Segment tabs (pipeline/midmarket/enterprise) remain for filtering; "All" is the landing tab and the title-map needs an `'all'` entry.

**4. Typography — split by page (2026-07-16 overhaul).**
- **DASHBOARD (`index.html`/`dashboard.html`):** **Space Grotesk** `--font-display` for the wordmark/product name + `.context-h1` headings (do NOT use Fraunces — broken display-size "f"); **Inter** for all UI/body, and `--font-mono` is intentionally **repointed to Inter**; **Newsreader** serif `--font-num` for prominent figures ONLY (signal numbers, warmth scores, summary-stat figures). **NO actual monospace anywhere in the dashboard** — never re-add JetBrains Mono to the dashboard font `<link>` or any `font-family`. (Lyric Network Map match.)
- **WALKTHROUGH (`build.html`) + LANDING (`landing.html`):** unchanged Ember system — Space Grotesk display, Inter UI/body, **JetBrains Mono for NUMERIC DATA ONLY** (never on labels/eyebrows/headings — mono-on-labels is an AI-slop tell). This is the only place the mono rule still applies.

**5. Scoring consistency (stored composite == live weights).** The /100 signal shown in the dashboard MUST use the same weights as the tier/phase-7 composite. Lantern: `Math.round((0.35*pain + 0.25*trig + 0.25*res + 0.15*hir) * 20)` (Warranty 35 / Opportunity 25 / Tooling 25 / Hiring 15 — re-derive the weights per founder, and show them in a visible axis legend). Do NOT let `index.html` compute an equal-weight `(a+b+c+d)*5` total while tiers come from a weighted composite; they will disagree on the card. One weighting, applied in both places. **Corollary — ties:** whole-point axes make composite collisions routine. When two or more accounts land on the same composite, break the tie with an explicit `composite: <n>` override per company in data.js (evidence strength decides the order — see the Phase 7 composite tie-break step); `computeSignal()` returns the override first, so card, tooltip, and tier stay consistent.

**6. Motion + mobile.** `@media (prefers-reduced-motion: reduce)` disables fade-ups / animations. No horizontal scroll on mobile — when you add an `overflow-x:auto` rule for the tab nav, target the actual element's CLASS (Lantern bug: the rule targeted `.tab-nav` but the element's class was `.page-tabs`; `tab-nav` was only the id, so the rule did nothing).

**7. Network + Contacts tabs (data-driven, engine-owned).** Two tabs beyond the 3 segment tabs + "All", driven solely by `window.NETWORK_DATA`. Never build, edit, or fabricate them per founder; never remove `<script src="network-data.js">`. See "The Network + Contacts tabs" section below.

**8. Balanced headers (no orphan lines).** `text-wrap:balance` on `.context-h1` and `.context-sub` so a heading/subheading never wraps to a lone one-word last line (symmetry).

**9. Uniform card heights.** `.card-grid{align-items:stretch;grid-auto-rows:1fr}` + `.card{height:100%}` so every account card in a grid is the same height regardless of tag-row count.

**10. Tab bar.** Tabs render at 17px/600 in near-white; the per-tab count pills are hidden by CSS (`.tab-count{display:none}`). Do not "fix" the pills back on.

**11. Partner mark + polish.** The dashboard topbar co-brand is an inline Primary icon SVG + "Primary" TEXT sized/baselined to the product name — NOT `<img src="assets/primary-lockup.svg">` (that asset is for the walkthrough only). Also template-owned: summary stat cells vertically centered, dropdown menus max-height-capped + scrollable, trackpad zoom proportional + capped. All of these live in the template — fixes go upstream, never per build.

**12. Deploy gate.** The template ships `middleware.js` (edge Basic-Auth, fail-closed) + `vercel.json`. Keep both; the engine deploys ONLY the dashboard files (allowlist) and sets `FDI_DASHBOARD_PASSWORD`. Never deploy the raw build root (it holds `inputs/`, `CONTEXT.md`, research JSON).

**Two gotchas that break the render silently — check both after any copy edit:**
- **Apostrophes in single-quoted JS strings.** Inserting copy containing `'` (e.g. "group's", "Lantern's") into a single-quoted JS string literal breaks the script with NO console error — the page just hangs on "Loading...". Reword to avoid the apostrophe (or escape it), then run `node --check` on the extracted inline `<script>`.
- **`computeJobSignal` `/careers` filter.** If the hiring axis is computed from `JOB_LISTINGS` and the filter drops generic `/careers` URLs, a company whose listings all end in a bare `/careers` scores hiring=1 (false flatline — bit West Herr + Go Auto). Use deep careers URLs (the actual posting) and seed enough verified entries to reflect the tier. (Related: F11 axis-uniformity self-check.)

### The Network + Contacts tabs (engine-owned data, skill-owned shell)

Added 2026-07-16. The dashboard ships two relationship tabs that are **wholly data-driven and NOT built or customized per founder.**

**The data contract.** Both tabs read `window.NETWORK_DATA` from `network-data.js`, which is written by the **FDI engine** (`fdi-engine/scripts/fdi/fetch-affinity-network.mjs`) as a workflow step **AFTER your build finishes** (post-Phase-10, pre-deploy), from real Affinity relationship data. It does not exist during any skill phase. The engine script is the schema authority — this is the MINIMAL UI contract (the engine emits more per-path fields: shown_paths, best_score, connector_id/email, contact_id/email, has_interaction_data, enrichment_status). Shape (v1):
- `schema_version: 1`, `build_status: "ok" | "partial" | "unavailable"`
- `summary: { companies_total, companies_with_path, coverage_pct, total_relationships, primary_connectors[] }` — plus, after the roster filter runs, `excluded_connectors[]` + `roster_filter: { status: "applied"|"unavailable", source, roster_size, dropped_paths }` (additive; audit trail for the former-employee strip)
- `connectors: { "<Full Name>": { id, title, org, linkedin, location } }` — powers the hub-click connector card + "Show all connections →"
- `companies: [ { company, domain, status: "resolved"|"no_relationships"|"not_found"|"error", total_paths, paths: [ { connector, contact_name, contact_title, seniority, type: "interaction"|"linkedin", score (0-100 | null for LinkedIn-only), at_target_company, linkedin_url, last_contact, history{ first/last email+meeting, next_meeting, with_connector } } ] } ]`

**What the UI does with it** (so you don't try to rebuild it): a radial map per account (target center → Primary connector hubs → contact dots), warmth ramp fixed green/amber/**red** for faint <40 (independent of the founder accent), animated flow dots, boxed legend with a Map/List toggle, an **in-map filter bar** (top-left: seniority / warmth / last-contacted / connector — filters the graph AND the ranked list together), a **visual dot key** (ringed dot = works at the company, plain = elsewhere, size = seniority), a summary strip (coverage + clickable stats incl. Strong-75+ / C-level-reachable that deep-link into Contacts pre-filtered), a connector-centric view, and a ranked list with a **Sort-only** toolbar; the Contacts tab flattens paths into a filterable rows table (search + warmth/seniority/connector/account/activity), tier-grouped, row-expand (why-this-score + history timeline), multiselect → bulk bar → saved lists (`fdi_lists_<slug>`, localStorage, device-local) + CSV export.

**Contact de-duplication (render-side).** Each contact appears **once per company** (`groupPaths`/`contactKey` in the template), carrying all their connector paths on one row; the company map still shows a contact under each *distinct* connector who can intro them. Missing `contact_title` is **omitted** (no placeholder/em-dash). This is template behavior — you neither implement nor defeat it (don't "fix" data by inventing titles or splitting a person into fake rows).

**Former-employee filtering (engine-side).** The engine strips warm paths whose connector is no longer on the live primary.vc/people roster when it generates `network-data.js` (`fetch-affinity-network.mjs` → `lib/primary-roster.mjs`; audit trail in `summary.roster_filter` / `summary.excluded_connectors`; **fails open** if the roster can't be fetched). The template also carries an `EXCLUDED_CONNECTORS = []` array at the top of the network IIFE — a **manual per-build override knob** that ships empty and normally stays empty. Only populate it for a one-off correction the engine missed; names must match `paths[].connector` exactly.

**Hard rules:**
- NEVER write, copy (the template's `network-data.js` is a fictional fixture), or fabricate `network-data.js`. If it's absent, the tabs show a designed empty state — that is correct, not a defect.
- NEVER edit the Network/Contacts markup or JS per founder, and never remove `<script src="network-data.js">`. Leave `EXCLUDED_CONNECTORS` empty unless you have a specific, documented reason.
- The ONLY skill-side inputs that affect these tabs: `data.js` `SEGMENTS[].companies[].name` (exact-match join for display metadata), `SEGMENTS[].companies[].domain` (the Affinity join key), and `{{PRODUCT_SLUG}}` (the saved-lists localStorage namespace).

### Phase 8: Customize index.html

The template HTML uses `{{...}}` placeholders for everything that varies per build. Start by copying the template:

```bash
cp template/index.html index.html
```

Now substitute every `{{...}}` placeholder. The full list (14 dashboard placeholders, all enumerable via `grep -oE "\{\{[A-Z_0-9]+\}\}" template/index.html | sort -u`):

| Placeholder | Source / Value |
|---|---|
| `{{PRODUCT_NAME}}` | Founder's product name (Phase 0). **Use the FULL brand name** ("Frost Security", never "Frost") — it renders in the topbar brand, the partner mark, the axis-legend prose, and the feedback card; a short name reads wrong in all four. Occurs 5×. (Frost build shipped "Frost" — F16.) |
| `{{PRODUCT_SLUG}}` | Founder's slug (Phase 0). Powers the theme localStorage key (`<slug>-theme`) AND the Contacts tab's saved-lists key (`fdi_lists_<slug>`). Occurs 5× incl. inside the Contacts IIFE — replace globally. |
| `{{AXIS1_LABEL}}` | Phase 5 founder-specific axis 1 name (e.g., "Inference Pain", "Workflow Pain", "Process Pain") |
| `{{AXIS1_DESCRIPTION}}` | One-sentence "what it measures" for axis 1, ending in a period — e.g., "heterogeneous-accelerator routing, gross-margin, or latency pain. Sourced from earnings calls, blog posts, and product docs." |
| `{{AXIS2_LABEL}}` | Phase 5 founder-specific axis 2 name (the wow-axis) |
| `{{AXIS2_DESCRIPTION}}` | One-sentence "what it measures" for axis 2 |
| `{{HIRING_AXIS_DESCRIPTION}}` | One-sentence description of what the Hiring sub-score measures for this vertical (e.g., "active job listings for ML / inference platform roles. Specific tech in JDs (Triton, vLLM, etc.) raises the score.") |
| `{{HIRING_FALLBACK_TEXT}}` | Text shown in the tooltip when a company has zero hiring signal (e.g., "No specific inference/ML hiring detected.") |
| `{{HIRING_KEYWORD_REGEX}}` | Phase 5 artifact #2 — the literal regex (e.g., `/inference\|llm \|triton/i`) |
| `{{SEGMENT_MIDMARKET_SUBTITLE}}` | One-sentence subtitle for the Mid-Market tab landing page (e.g., "Data-sensitive mid-market accounts with near-term BYOC inference need.") |
| `{{PRODUCT_LOGO_SVG}}` | The founder's mark — a simple ~24×24 line-icon `<svg>`. The template ships a neutral spark as the default; replace it with something on-brand (the Lantern build used a coach-lamp lantern). Lives in `.logo-mark`; the landing reuses it. |
| `{{FEEDBACK_CARD_BODY}}` | 2-3 sentence body for the dashboard's bottom "How'd we do?" outro card, per founder. Per Output Style Rule #18: acknowledge we're early in the founder's world, ask explicitly what they love and where we can improve, close on making it stronger together. Distinct copy from the walkthrough's `introWarmth`/`finaleWarmth` — do not repeat those lines. |
| `{{THEME_ACCENT_DARK}}` | The dark-mode accent ramp, from `theme-presets.json` preset `config.theme_color` (fallback `ember`). Emit these 9 lines using the preset's `dashDark.{green,greenDark,greenDeep}` (G = the `green` triple): `--green-h: <green>;` (raw triple, no hsl() — powers the custom-alpha hero glow inner + med-chip) `--green-h2: <the preset's walkthrough.accDeep triple>;` (raw — the darker outer hero-glow stop; only emit in the DARK block, light inherits it) `--green: hsl(<green>);` `--green-dark: hsl(<greenDark>);` `--green-deep: hsl(<greenDeep>);` `--green-tint: hsl(<G> / .15);` `--green-tint2: hsl(<G> / .08);` `--green-ring: hsl(<G> / .4);` `--green-glow: 0 0 0 1px hsl(<G> / .45), 0 0 22px hsl(<G> / .18);` |
| `{{THEME_ACCENT_LIGHT}}` | Same 8 declarations for light mode, from the preset's `dashLight.{green,greenDark,greenDeep}` (Gl = the light `green` triple): `--green-h: <Gl>;` then the ramp with light alpha stops: tint `/ .12`, tint2 `/ .07`, ring `/ .3`, glow `0 0 0 1px hsl(<Gl> / .35),0 0 24px hsl(<Gl> / .15)` (note light glow blur is 24px, matching the original). (`--q-med`/`--q-med-d` already reference `var(--green)`/`var(--green-dark)` in the template, so they follow automatically — do not emit them.) |

Substitute via `sed` or directly — all should be replaced before validation. Several tokens occur multiple times (incl. `{{FEEDBACK_CARD_BODY}}` twice — one inside a CSS comment; `{{PRODUCT_NAME}}`/`{{PRODUCT_SLUG}}` 5× each), so substitution must be GLOBAL. The zero-match grep (digit-safe regex, below) is the only acceptance test.

Other Phase 8 work:

1. Update tab labels in `<nav class="page-tabs">` if your segment IDs differ from default (default is pipeline / midmarket / enterprise). There are **6 tabs total** — only the 3 segment tabs get renamed; **"All", "Network", and "Contacts" labels are fixed** — do not rename or remove them.
2. Sanity check `.tag.brand` CSS class is intact. All `c: 'brand'` values in your data.js render with founder accent.
3. The data.js `{{SECTION_2_LABEL}}` placeholder is consumed by index.html's section rendering — verify Phase 7 set it consistently across all entries.
4. **Do NOT touch the Network or Contacts tabs.** They are wholly `window.NETWORK_DATA`-driven with no per-founder customization. Never remove the `<script src="network-data.js">` tag (it 404s locally by design; the tabs show a designed empty state), and never copy `template/network-data.js` (a fictional fixture) into the build root.

**Validate after edits:**

```bash
# Self-validating: any remaining {{...}} placeholder = a missed substitution.
# NOTE the digit-safe class [A-Z0-9_]+ — a bare [A-Z_]+ MISSES {{AXIS1_*}}/{{AXIS2_*}}.
grep -nE "\\{\\{[A-Z0-9_]+\\}\\}" index.html
# If any results, those are missed replacements — fix.

# Check the file is reasonably-sized HTML (not corrupted)
wc -l index.html
```

This grep runs AGAIN on the FINAL `dashboard.html` in Phase 8c (after the rename) and in Phase 10 — passing here does not clear the build. (Frost shipped a live `{{FEEDBACK_CARD_BODY}}` because validation never re-ran after `index.html` → `dashboard.html`. See F15.)

**Commit:**

```bash
git add index.html
git commit -m "Phase 8: index.html customized with [vertical] axis labels and branding"
```

**Manual verification:** open `index.html` in a browser. Walk through one company card end-to-end. Any string you see in the UI that says "Inference Pain" or "Data Residency" is a missed replacement.

### Phase 8b (OPTIONAL, default OFF): Landing / cover page

A branded cover page. **Optional and off by default since v3.1** — the walkthrough (Phase 8c) opens on the founder itself, so a separate cover is redundant. Build it only on an explicit ask. The template ships a ready scaffold: **`template/landing.html`** (a dark cover page with the proven structure). Don't build from scratch — copy it and fill its placeholders. Reference build: `alexg207/fdi-builder-v2` (Lantern, Ember design).

**File convention when explicitly requested:** the landing takes `/landing.html` (the walkthrough owns `index.html`); link its CTAs to `./` and `./dashboard.html`.

**Landing-only placeholders** (`grep -oE "\{\{[A-Z_0-9]+\}\}" template/landing.html | sort -u`): `{{PRODUCT_NAME}}`, `{{PRODUCT_SLUG}}`, `{{PRODUCT_LOGO_SVG}}`, `{{POSITIONING_EYEBROW}}` (Q3), `{{PRODUCT_HEADLINE}}` (Q4), `{{STAT_1_VALUE}}`–`{{STAT_4_VALUE}}` + `{{STAT_1_LABEL}}`–`{{STAT_4_LABEL}}` (Q5), `{{PRODUCT_TRANSITION_LEAD}}` (Q7), `{{ICP_DESCRIPTION}}`, `{{MARKET_SIZE_PHRASE}}`, `{{BUYER_ROLE}}`, plus the shared `{{AXIS1_LABEL}}` / `{{AXIS2_LABEL}}` (reuse the dashboard's values). The landing is dark-only by design (the cover, not the app) — no light toggle needed.

**Clarifying questions the builder must answer before building (ask these first; headless: skip the landing entirely unless the dispatch asked for it):**
1. Build the landing page at all? (default: NO — the walkthrough is the entry)
2. Founder accent color + any brand font / logo asset? (default: keep the Ember system — cool ink canvas + ember accent + Space Grotesk — and design a simple themed line-icon like Lantern's coach-lamp lantern. Only override the accent when the founder's brand demands it, via the token values in the landing and `founder.themeAccent` in build-data.js so all three pages match)
3. Founder positioning eyebrow — one line of what the founder IS (pull from CONTEXT.md; Lantern's was "The intelligence layer for blue-collar work")
4. Product headline — what the founder's product does, founder-voiced (pull from CONTEXT.md)
5. 3–4 market proof stats for the small stat cards (the founder's market, each citable — confirm wording; keep ambition stage-appropriate, no over-claimed ARR / scale)
6. Co-brand lockup wording: "[Founder] × Primary" — confirm
7. The transition line that pivots from the founder's product to the deliverable — confirm exact wording, e.g. "Primary built [Founder] a **custom sales-intelligence dashboard**" (bold that phrase)
8. Audience: founders only (internal) vs shareable? (sets how much to explain and the jargon level)
9. Theme: dark cover by default (regardless of the dashboard's last-used theme)?
10. What does the dashboard UNLOCK for the founder's GTM? (drives the "What it unlocks" section — describe capabilities, not a prescribed sales playbook)

**Structure (Lantern, proven):**
- **Topbar:** founder mark + wordmark (serif) on the left; persistent "Enter the Dashboard" accent CTA + "[Founder] × Primary" on the right. The persistent CTA lets readers skip ahead without scrolling.
- **Hero:** glow mark, serif wordmark, positioning eyebrow, product headline, 3–4 SMALL stat cards under the headline, a transition paragraph that sells the **custom sales-intelligence dashboard** (bold that phrase), a primary CTA + a "Skip intro" link + an animated `.scroll-cue` ("How it works ↓"). Keep the hero breathable — do NOT cram the full pitch above the fold; readers scroll, and the topbar CTA covers skip-ahead.
- **Sections (each with a display-serif accent eyebrow, never mono):**
  - "Your thesis, turned into a ranked target list" — what this is.
  - "How the map is built" — numbered timeline: Define the signals → Scan the universe → Score 0 to 100 (every number cited) → Curate the shortlist.
  - "What it unlocks" — stacked cards: Focus the team / Warm entry / Conviction / Built to scale (scalable beyond the first 10). Describe GTM leverage.
  - "Inside the dashboard" — preview card + final Enter CTA.
  - Footer.

**Copy principles (learned the hard way on this build):**
- Sell the DASHBOARD and what it unlocks. Don't just regurgitate facts about the founder's business, and don't tell the founder their own GTM strategy or how to win a trial.
- Two visually + structurally distinct section layouts — "how it's built" (numbered timeline) and "what it unlocks" (stacked cards) must not overlap or read the same.
- Section eyebrows AND headings use the display serif (NOT mono). The accent eyebrow is the same font family as the white heading; the founder/product heading can be slightly larger.
- No em dashes (commas / periods / "..."). Human voice, not VC-deck filler. Keep scale ambition stage-appropriate (no "out of 18,000 dealerships" over-drama).
- The entry URL always opens the landing first, with the dashboard one click away.

**Validate:** `node --check` the inline `<script>`; confirm BOTH dark and light render; confirm no `{{...}}` left; open in a browser and walk hero → sections → Enter → back. Commit: `git commit -m "Phase 8b: landing / cover page"`.

### Phase 8c (STANDARD, auto — no human stop): The scroll walkthrough — the ENTRY page

Every build ships two pages: **index.html (the walkthrough) → dashboard.html.** The walkthrough opens on the founder (two-beat hero: founder-first intro with their logoSvg floating in 3D behind, then the dashboard pivot), so no separate landing is needed. Full contract + copy rules: **`template/TEMPLATE_GUIDE.md` Section 16**. This phase is fully automatic — every decision derives from artifacts already produced; do not ask questions.

**Step 1 — copy the generic files verbatim:**

```bash
mv index.html dashboard.html                 # the dashboard moves off the root
cp template/build.html ./index.html          # the walkthrough IS the entry — NEVER edit per founder
mkdir -p assets && cp -R template/assets/. assets/   # EVERYTHING — logos AND primary-lockup.svg
cp template/middleware.js template/vercel.json template/package.json template/.vercelignore .   # deploy gate + ESM flag + allowlist MUST ship at root
test -f middleware.js && test -f package.json && test -f .vercelignore || echo "BUILD ERROR: middleware.js / package.json / .vercelignore missing at root — deploy would be unprotected or leak the workdir"
git add middleware.js vercel.json package.json .vercelignore   # new files — git add -u would miss them
test -f assets/primary-lockup.svg || echo "BUILD ERROR: primary-lockup.svg missing — the WALKTHROUGH's topbar/intro/network act render broken images without it (the dashboard topbar does NOT use it — its partner mark is an inline SVG)"
# MANDATORY: re-validate placeholders on the file that actually deploys (post-rename).
grep -nE "\\{\\{[A-Z0-9_]+\\}\\}" dashboard.html && echo "✗ BUILD ERROR: placeholders in the SHIPPED dashboard.html" || echo "✓ dashboard.html clean"
```

Copy the WHOLE `template/assets/` tree, never cherry-pick: two real builds shipped with broken Primary lockups because only `logos/` was copied.

If you feel the urge to edit the walkthrough HTML for this founder, stop: the thing you want is a `narration` key or a template-repo fix, not a per-build fork.

**Step 2 — generate `build-data.js`** (spawn a subagent with this task): synthesize from `config.json` (founder block), `CONTEXT.md` (voice, wow reasoning, market stats), `webset-spec.json` (process stages, axes, scan query, enrichments), and the scored companies in `data.js` / `webset-response.json`. Schema + per-key generation notes: `template/build-data-template.js`. Rules the subagent must follow:

- **Voice:** confident analyst briefing the founder — never vendor pitch. Hyphens only, NEVER em dashes. Narrations ≤2 sentences each.
- **`narration.introHeadline`** = the founder's product headline for beat 1 (accent spans allowed), derived from CONTEXT.md positioning; falls back to `founder.oneLine`.
- **Hero frames account QUALITY, not count.** Pattern: line 1 = scale in ("18,000 rooftops in."), line 2 = quality out ("The readiest buyers out."). Never "N accounts out."
- **Each heroTitle line must fit on ONE rendered line: 28 characters max including spaces.** Longer lines wrap and orphan a word ("out." alone on its own line) - a real build shipped that way. Count the characters.
- **Every number and citation is REAL.** `heroStats` are the build's actual counts; `evidenceFeed` = 8-9 citations lifted from the scored companies' own `sources` (tier 1/2 mix). Fabricating either is a build-failing offense.
- **Exactly one axis carries `wowNote`** — the founder-specific WOW signal. Its `body` is the "why this signal wins" argument from CONTEXT.md, 3-4 sentences, `<b>` allowed on the one load-bearing phrase.
- **Keep the honest hedges:** the WALKTHROUGH's act-8 network fan stays `illustrative: true` with role-based connectors (Primary Partner / vertical Advisor / Founder / Operator Network — 3-4 roles, each owning a clean partition of the shortlist; secondary paths in `alsoReaches`). This governs ONLY the `build-data.js` walkthrough fan — it is unrelated to the DASHBOARD's Network tab, which is real Affinity data the engine writes to `network-data.js` after the build. Never mark the dashboard side illustrative, and never fabricate NETWORK_DATA to make the walkthrough and dashboard "agree."
- **Theme:** set `founder.themeAccent` from `config.theme_color`'s preset in `template/theme-presets.json` — copy that preset's 6 `walkthrough` triples verbatim (`acc/accSoft/accDeep/acc2/bgh/nh`). For `ember` (the default), omit `themeAccent` entirely (build.html's `:root` is already Ember). Unknown/absent `theme_color` → treat as `ember` (omit). build.html applies these at runtime, so the whole walkthrough recolors. Same key must drive the dashboard's `{{THEME_ACCENT_*}}` (Phase 8) so both pages match. A hand-set `themeAccent` overrides the preset.
- **Apostrophes inside JS strings are curly (`’`)** — the known straight-quote silent-render gotcha applies to this file too.
- **Warmth fields (required, per Output Style Rule #18):** fill `narration.introWarmth` (one sentence under the intro headline — our deep dive into the founder's world, excited to keep exploring together) and `narration.finaleWarmth` (one sentence between the finale CTA and replay, `<b>` on the closing phrase — the start of a conversation, looking forward to continuing the research together). Both render via `innerHTML`, so `<b>`/accent spans are allowed; both hide gracefully when absent. The dashboard's `{{FEEDBACK_CARD_BODY}}` is the third warmth string, filled in Phase 8 — keep all three distinct.
- **TAM discipline:** `scan.universe` is the NARROW-ICP estimate that matches the scan query (the population that would actually pass all the ICP criteria), NOT the broad market TAM. `scan.funnel.universe` label = `"est. in [Founder]'s ICP"`. `heroStats` = FIVE stats leading with the ICP-TAM estimate (TAM → companies analyzed → custom signals → accounts curated → named contacts); `finaleSub` restates the same TAM number in prose. `scan.methodNote` (optional; replaces the generic scan-note line when present) tells the full funnel story in plain language, ≤~40 words: TAM estimate → surfaced and analyzed one by one → strongest fits → curated to the final N. Ground the TAM estimate in a real bottom-up method (name it in BUILD_NOTES); never invent a round number.
- **Tagline anti-duplication:** `founder.tagline` renders directly above `introHeadline` in beat 1. If it would restate the headline, set it to `""` — beat 1 renders cleanly without it. Never say the same thing twice in the opener.

**Step 3 — capture `assets/dashboard-preview.png`** for the hero's 3D preview: screenshot the populated `dashboard.html` at 1600×1000 @2x (Playwright if available; headless CI may skip — the hero hides the preview gracefully when the file is missing, but the page is much stronger with it. If skipped, record it in BUILD_NOTES as a follow-up).

**Step 4 — validate:**

```bash
node --check build-data.js
grep -ci "<previous-founder-name>" build-data.js   # must be 0 (also grep "lantern" on non-Lantern builds)
```

Then open `index.html?act=N` for N=0..9 and confirm every scene renders with this build's data; check `axes[].weight` sums to 100 and `companies[0]` is the intended hero account (act 6 blends it). Commit: `git commit -m "Phase 8c: scroll walkthrough entry (index.html + build-data.js)"`.

### Phase 8d (OPTIONAL, default OFF): Competitor teardown — the `competitors` module

Runs ONLY when `config.json.modules` is a non-empty array that includes
`"competitors"`. If the `modules` field is absent/empty, or is a selection that
omits `"competitors"`, SKIP this phase entirely (ship neither `competitors.html`
nor `competitors-data.js`; record `competitors: "skipped"` in Phase 10's
build-summary) — `competitors` is `default:false`, so unset = off. See
"Modules (registry-driven)".

When selected, produce a STANDALONE page (its own tab in the GTM section, NOT a
tab inside the dashboard): `competitors.html` (copied from the template, stamped)
+ `competitors-data.js` (`window.COMPETITORS_DATA`, generated). This ships next to
the renamed `dashboard.html` from Phase 8c and links back to it.

**Step 1 — research (basic-Exa pool; budget ~$1-2 for 5-8 competitors).** Seed
the candidate list from CONTEXT.md's **Competitive Landscape** table (Phase 4 —
category / players / why-they-fail, in the founder's own framing), then verify +
enrich each with FRESH external research via `web_search_exa` / `web_fetch_exa`:
positioning, funding stage, pricing model, notable customers, genuine strengths,
and the structural weakness the founder exploits. Pick the 5-8 that actually
compete for this founder's buyer (drop tangential ones). Every non-obvious claim
must trace to a real source URL you fetched — never assert funding/customers/
pricing from memory.

**Step 2 — generate `competitors-data.js`** per `template/competitors-data-template.js`
(the schema doc). Shape (`window.COMPETITORS_DATA`, schema_version 1):
`build_status:"ok"`; `founder:{name, positioning}`; `market_map:{axis_x:{label,low,high},
axis_y:{label,low,high}, placements:[{name,x,y,is_founder}]}` — the two axes are
the dimensions that separate this market (e.g. "Point tool ↔ Platform",
"SMB ↔ Enterprise"), `x`/`y` in 0-100, EXACTLY ONE placement `is_founder:true`
(the founder) and one placement per competitor; `competitors:[{name, domain,
category, one_liner, positioning, strengths[], weaknesses[], why_founder_wins,
funding_stage, pricing_model, notable_customers[], sources:[{title,url}]}]`
(≥2 https sources each); `summary`. Hyphens only, NEVER em dashes. Apostrophes
inside JS strings are curly (`’`) — the straight-quote silent-render gotcha
applies here too.

**Step 3 — stamp the page + the nav link.**
```bash
cp template/competitors.html ./competitors.html
# stamp the 3 placeholders exactly as Phase 8 does for the dashboard
#   {{PRODUCT_NAME}} {{THEME_ACCENT_DARK}} {{THEME_ACCENT_LIGHT}}  (from config.theme_color's preset in template/theme-presets.json; ember → the template default)
# reveal the dashboard's nav link (Phase 8c already renamed the dashboard to dashboard.html):
perl -0pi -e 's/<!--\s*COMPETITORS_NAV\s*-->/<a class="page-tab" href="competitors.html">Competitors<\/a>/' dashboard.html
grep -q 'href="competitors.html"' dashboard.html || echo "✗ BUILD ERROR: COMPETITORS_NAV anchor not stamped in dashboard.html"
```
(When this phase does NOT run, the inert `<!-- COMPETITORS_NAV -->` comment simply
stays in dashboard.html — invisible, no link, zero effect. Never leave the comment
AND ship competitors.html; never ship the link without competitors.html.)

**Step 4 — self-check (fail the ENTRY, repair, before committing):**
- `node --check competitors-data.js`; no `{{...}}` placeholders survive in
  competitors.html (grep); no em dashes in authored copy.
- 5-8 competitors; each has ≥2 `https://` sources; `why_founder_wins` reads as
  POSITIONING (why the founder is different/better for the buyer), never
  disparagement — this page can ship world-readable under the founder's brand on a
  public build (middleware is stripped), so every competitor claim must be sourced
  and fair.
- `market_map`: exactly one `is_founder:true`; every `placements[].name` equals
  `founder.name` or one of `competitors[].name`; all `x`/`y` in 0-100.
- Open `competitors.html` locally: cards render for every competitor, the market
  map plots every placement with the founder highlighted, sources are clickable,
  the back-link reaches `dashboard.html`, zero console errors. Then flip the theme
  toggle and confirm it persists (same localStorage key as the dashboard).

**Degrade (never fail the build):** if the research genuinely can't complete
(e.g. basic-Exa 402, or too few credible competitors surface), still ship BOTH
files — write `competitors-data.js` with `build_status:"unavailable"` and empty
`competitors:[]` / `placements:[]` (the page renders its designed empty state,
exactly like the dashboard's Network tab without `network-data.js`), stamp the
nav link anyway, record `competitors: "failed"` in the build-summary, and note it
in BUILD_NOTES. A skipped-because-unselected module is different (ship neither
file, `"skipped"`).

Commit: `git commit -m "Phase 8d: competitor teardown (competitors.html + competitors-data.js)"`.

### Phase 9: Write BUILD_NOTES.md

Following `template/BUILD_NOTES_TEMPLATE.md`, write `./BUILD_NOTES.md` at the repo root. Document:

- Signal axis labels chosen + justification
- Segments + reasoning
- Sections per company + reasoning
- Features kept/dropped + reasoning
- Notable copy decisions
- Companies featured & rationale (especially if Webset returned more than were used)
- **Webset details: ID, query, criteria, enrichment fields, completion time** (so user can refresh later)
- **Cinematic section** (per the template's BUILD_NOTES shell): narration voice decisions, wowNote axis + why, evidenceFeed sources, themeAccent choice, dashboard-preview.png status, self-check results
- **Reproducibility note:** point to `webset-spec.json`, `webset-response.json`, `lovelace-contacts.json` as the saved intermediates
- **Network tab note:** `network-data.js` is engine-generated from Affinity AFTER this build (or still pending, for a local build) — so a reviewer doesn't read the empty Network/Contacts tabs as a defect. Record the per-company `domain` values used (they're the Affinity join keys; a wrong domain silently drops the account).
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

# 2. No leftover placeholders/markers in the SHIPPED files (digit-safe regex +
#    bracket markers like [SECTION_2_LABEL]/[REPLACE]). dashboard.html is the
#    load-bearing addition — it's what deploys (F15: Frost shipped a live {{...}}).
grep -nE "\\{\\{[A-Z0-9_]+\\}\\}|\\[SECTION_2_LABEL\\]|\\[REPLACE\\]" data.js dashboard.html build-data.js index.html && echo "✗ Placeholders/markers remaining" || echo "✓ No placeholders"
grep -n "Inference Pain\\|Data Residency" dashboard.html | grep -v "^[[:space:]]*\\(//\\|\\*\\)" && echo "✗ Valar axis labels still in HTML" || echo "✓ Axis labels customized"
# (skip the second check if the founder genuinely is Valar)

# 2b. Dashboard integrity: engine-owned network file absent, exactly one script
#     tag for it, no mono regression, Newsreader present.
test ! -f network-data.js && echo "✓ no network-data.js (engine writes it post-build)" || echo "✗ network-data.js present — you fabricated or copied the fixture; delete it"
[ "$(grep -c 'src=\"network-data.js\"' dashboard.html)" = "1" ] && echo "✓ network-data script tag intact" || echo "✗ network-data.js script tag missing/duplicated"
grep -ci "JetBrains" dashboard.html | grep -q '^0$' && echo "✓ no mono in dashboard" || echo "✗ JetBrains Mono leaked into the dashboard"
grep -qE '\-\-font-num:[^;]*Newsreader' dashboard.html && echo "✓ --font-num maps to Newsreader" || echo "✗ --font-num not Newsreader (font overhaul regressed)"
grep -qE '\-\-font-mono:[^;]*Inter' dashboard.html && echo "✓ --font-mono repointed to Inter" || echo "✗ --font-mono not Inter (mono may render)"

# 3. Required output files all exist (two-page flow + cinematic data)
for f in CONTEXT.md data.js index.html dashboard.html build-data.js BUILD_NOTES.md; do
  test -f "$f" && echo "✓ $f" || echo "✗ MISSING: $f"
done

# 3b. Cinematic checks: build-data.js parses, no previous-founder leakage,
#     build.html untouched vs template
node --check build-data.js && echo "✓ build-data.js parses" || echo "✗ build-data.js PARSE ERROR"
diff -q template/build.html index.html && echo "✓ walkthrough verbatim" || echo "✗ index.html was edited — revert and move the change into build-data.js or the template repo"

# 4. Saved intermediates exist (reproducibility)
for f in webset-spec.json webset-response.json lovelace-contacts.json; do
  test -f "$f" && echo "✓ $f saved" || echo "✗ MISSING intermediate: $f"
done

# 5. Git history is clean (per-phase commits visible)
git log --oneline | head -15

# 6. Optional modules: files must exist IFF selected in config.json.modules.
#    competitors (skill:8d) is the only skill-generated optional page today.
#    An empty/absent modules field means "all defaults" → competitors OFF.
MODS=$(node -e "try{const m=JSON.parse(require('fs').readFileSync('config.json','utf8')).modules;process.stdout.write(Array.isArray(m)?m.join(','):'')}catch(e){}")
if printf '%s' ",$MODS," | grep -q ",competitors,"; then
  for f in competitors.html competitors-data.js; do test -s "$f" && echo "✓ $f (competitors selected)" || echo "✗ MISSING: $f — competitors selected but not generated"; done
  node --check competitors-data.js && echo "✓ competitors-data.js parses" || echo "✗ competitors-data.js PARSE ERROR"
  grep -q 'href="competitors.html"' dashboard.html && echo "✓ dashboard nav links to competitors" || echo "✗ COMPETITORS_NAV not stamped in dashboard.html"
else
  test ! -e competitors.html && test ! -e competitors-data.js && echo "✓ competitors not selected — no competitors files (correct)" || echo "✗ competitors NOT selected but competitors files exist — delete them (a stray page would deploy unverified)"
fi
```

**Write `build-summary.json`** (the LAST artifact — see "Modules (registry-driven)"). Map every registry module THIS SKILL owns to its outcome; leave engine-owned `network` out (the engine merges it post-build). Core modules (`walkthrough`, `dashboard`) are always `"ok"`; each optional skill module is `"ok"` (generated cleanly), `"skipped"` (optional, not selected), or `"failed"` (selected but shipped its degraded empty state). The status is DERIVED, not hand-set: a degraded competitors build ships `competitors.html` too, so file-existence alone can't tell `ok` from `failed` — read `COMPETITORS_DATA.build_status` (`"ok"` → ok, anything else → failed).
```bash
node -e '
  const fs=require("fs");
  let mods=[]; try{ const m=JSON.parse(fs.readFileSync("config.json","utf8")).modules; if(Array.isArray(m)) mods=m; }catch{}
  const sel = k => mods.length>0 && mods.includes(k);   // empty/absent selection = defaults; both optionals default:false = off
  const has = f => fs.existsSync(f);
  const landing = !sel("landing") ? "skipped" : (has("landing.html") ? "ok" : "failed");
  let competitors = "skipped";
  if (sel("competitors")) {
    competitors = "failed";
    if (has("competitors.html") && has("competitors-data.js")) {
      const m = fs.readFileSync("competitors-data.js","utf8").match(/build_status\s*:\s*[\"\x27]([a-z_]+)[\"\x27]/);
      competitors = (m && m[1] === "ok") ? "ok" : "failed";
    }
  }
  const summary = { modules: { walkthrough:"ok", dashboard:"ok", landing, competitors } };
  fs.writeFileSync("build-summary.json", JSON.stringify(summary,null,2)+"\n");
  console.log("✓ build-summary.json:", JSON.stringify(summary.modules));
'
```
Commit: `git commit -m "Phase 10: self-check + build-summary"`.

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
- [ ] Cinematic: every `narration` key filled (no generic fallbacks shipping); hero title = quality framing; `?act=0`..`?act=9` all render; reduced-motion shows the static version; `axes[].weight` sums to 100; exactly one axis carries `wowNote`; `evidenceFeed` lines trace to real company sources
- [ ] Cinematic warmth + TAM: `introWarmth` + `finaleWarmth` present and pass Rule #18 tone (confident partnership, distinct roles, not self-deprecating); `scan.universe` is the narrow-ICP estimate (not broad TAM) and matches `finaleSub`'s number; `heroStats` = 5 leading with the TAM stat; `scan.methodNote` tells the funnel story in ≤~40 words; `founder.tagline` does not restate `introHeadline`
- [ ] Dashboard: `{{FEEDBACK_CARD_BODY}}` replaced (no leftover token); the bottom "How'd we do?" card renders on every tab and in both light/dark themes
- [ ] Network: `<script src="network-data.js">` present; `EXCLUDED_CONNECTORS` is `[]` (engine filters the roster — only populate as a deliberate, documented override); Network/Contacts markup unedited per founder
- [ ] Theme: `config.theme_color` resolved; dashboard `{{THEME_ACCENT_DARK}}`/`{{THEME_ACCENT_LIGHT}}` filled from the preset (no leftover token, incl. `--green-h`), walkthrough `founder.themeAccent` = same preset's triples (or omitted for ember); accent recolors both pages incl. the med-tier chip (med IS the accent); high-tier(green)/low-tier(red) chips + purple links + teal star unchanged; ember default renders pixel-identical to before; unknown key fell back to ember
- [ ] Walkthrough finale CTA + skip link → `./dashboard.html` (walk index → dashboard once); beat 1 shows the founder's own mark floating, beat 2 pivots to the dashboard
- [ ] `{{PRODUCT_NAME}}` is the FULL brand name (e.g. "Frost Security", not "Frost") everywhere it renders — topbar brand, partner mark, feedback card (F16)
- [ ] Open `dashboard.html` locally: the 6-tab bar renders (All / 3 segments / Network / Contacts); the **Network and Contacts tabs show the designed empty state** (not an error, not the fixture's fictional people — `network-data.js` is engine-written post-build); no console/page errors

Fix any failures before delivery.

**Build artifacts cleanup (before final push to GitHub).** The build process generates several intermediate JSON files (`webset-spec.json`, `webset-response.json`, `founder-pick-research.json`, `lovelace-contacts.json`, `sumble-jobs.json`, `curated-10-list.json`, `inputs/`, `template/`) that are useful for reproducibility but clutter the deployable repo root. The pages the founder views are `index.html` (walkthrough) + `build-data.js` + `assets/` + `dashboard.html` + `data.js` (+ optional `CONTEXT.md` and `BUILD_NOTES.md` if you want them visible). Before final commit + push, write a `.gitignore` to exclude build artifacts from future commits, OR move them to a `.build/` subdirectory so they remain version-tracked but visually out of the way:

```bash
# Option A: keep artifacts tracked but tucked under .build/
mkdir -p .build
git mv webset-spec.json webset-response.json founder-pick-research.json lovelace-contacts.json sumble-jobs.json curated-10-list.json .build/
# inputs/ and template/ are kept at root since they're conceptually scoped to the build (read-only references)
# OR move them under .build/ too if the repo is going to be shared as a Sam-readable artifact

# Option B: write .gitignore excluding build artifacts
cat > .gitignore <<'EOF'
# Build intermediates — useful for reproducibility but not deployable
.build/
*.tmp.json
node_modules/
EOF
```

Recommendation: **Option A** when the repo will be shared with the founder (cleaner Git tree); **Option B** when only the maintainer touches the repo. Either way, the deploy artifacts (`index.html`, `data.js`) stay at the root.

**Public-deploy privacy (engine-owned).** The engine does NOT deploy the raw build root — it deploys only an allowlist of dashboard files (`index.html`, `dashboard.html`, `data.js`, `network-data.js`, `build-data.js`, `assets/`, `middleware.js`, `vercel.json`). This keeps `inputs/` (the founder's uploaded docs), `CONTEXT.md`, and the research JSON OFF the public site, and `middleware.js` + `FDI_DASHBOARD_PASSWORD` gate the whole thing behind Basic Auth. Do NOT relocate/rename the allowlisted files or the deploy will miss them. (If you ever deploy a build by hand, deploy from a clean directory containing only those files — never `vercel deploy` the workdir root.)

**Final commit if anything was fixed:**

```bash
git add -u
git commit -m "Phase 10: self-check fixes + build-artifact tidy"
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
- (Buyer ≠ founder is enforced in Phase 6a; the two-paragraph buyer profile re-extraction is the discipline.)
- **Long enrichment descriptions are not optional.** Every enrichment description should be a 50-100 word paragraph specifying signal, sources, format, and exclusions. "Documented pain points" returns generic. "Documented evidence that this company runs production AI inference at scale, citing engineering blogs or third-party press; if the company appears to BE an inference vendor rather than a consumer, mark NOT APPLICABLE" returns specific signal.
- (Phase 6f checkpoint, EXCLUDE clause discipline, and named-competitor exclusions are all enforced in Phase 6.)
- **Don't fight the async.** If the Webset takes 8 minutes, take 8 minutes. Don't try to populate data.js with placeholders during the wait, you'll do double work.
- **Founder voice in `gtm_thesis` is the leverage point on copy.** Splice verbatim quotes from `inputs/` or CONTEXT.md.
- **The wow signal should be visible from any angle.** Surface it in `gtm_thesis`, in `signals[]`, and in tags.
- (See Output Style Rule #14 — over-anchoring on the reference build is the central failure mode the skill is designed to prevent.)
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

**F11 — Hiring axis flatline from JOB_LISTINGS empty cascade.** Lantern May 6 build hit basic-Exa 402 mid-build, skipping Phase 6j entirely (the Sumble + careers + LinkedIn + ATS fallback ladder all depend on basic Exa). `JOB_LISTINGS[<co>] = []` for all 10 companies → `computeJobSignal()` default returned uniform 1 → Hiring axis carried zero discriminating signal across the dashboard. Three preventions: (1) Phase 0 preflight credit-pool probe halts the build BEFORE work starts when basic-Exa credits aren't funded; (2) Phase 6j Step 0 mines the Webset's already-paid-for role-evidence enrichment for verified named role-bearers as Tier-0 hiring signal that doesn't depend on basic-Exa; (3) Phase 7 axis-uniformity self-check fails the build if any axis has ≥80% identical scores across 10 entries, catching this and any future axis flatline regardless of root cause. Lantern May 6 also exposed a separate F4 recurrence — 2 of 10 gtm_thesis entries named target-company execs ("Brian Schlise", "Marco Schooley"); the existing subjective durability check missed both. Phase 7 self-check now runs an objective `\b[A-Z][a-z]+ [A-Z][a-z]+\b` regex over `gtm_thesis` and fails any capitalized two-word name that isn't in `PRIMARY_TEAM` or a recognized firm/fund.

**F12 — Silent render breakers (Lantern May 6).** Two edits broke the dashboard with no console error, page stuck on "Loading...": (1) an apostrophe in inserted copy ("group's") landing inside a single-quoted JS string literal; (2) a Python copy-edit script whose `rep()` return value wasn't assigned back, though that one crashed before writing rather than corrupting. Prevention: avoid apostrophes in copy that goes into single-quoted strings (or escape them), and run `node --check` on the extracted inline `<script>` after every copy edit. See "Dashboard visual defaults (v3)" gotchas.

**F13 — Two scoring systems disagreeing (Lantern May 6).** The tier (high/med) came from the weighted phase-7 composite while the displayed /100 total used an equal-weight `(a+b+c+d)*5` — so a card could show a green "high" chip next to a number that didn't earn it. Separately, `computeJobSignal` zeroed West Herr + Go Auto because all their `JOB_LISTINGS` URLs were bare `/careers` (filtered as generic). Prevention: one weighting applied in both the composite and the live `computeSignal`; derive tier from the displayed score; use deep careers URLs. Captured as v3 defaults #2 and #5.

**F14 — Display-serif "f" + mono-on-labels (Lantern May 6).** Fraunces at display size rendered a broken-looking lowercase "f" (user: "what is this F"); mono crept onto section labels and read as AI-slop. Prevention: one display face only (Space Grotesk since v3; previously Newsreader), Inter for UI, mono for numeric DATA ONLY — never on labels/eyebrows/headings. **Scope (2026-07-16):** this applies to `build.html` (walkthrough) + `landing.html` ONLY. The DASHBOARD moved to Inter + Newsreader figures with **NO monospace** — see v3 default #4. Do not re-add JetBrains Mono to the dashboard.

**F15 — Unstamped placeholder on the live dashboard (Frost, 2026-07).** `{{FEEDBACK_CARD_BODY}}` shipped raw on the live page: Phase 10's placeholder grep ran on `index.html`, but after Phase 8c `index.html` is the WALKTHROUGH — the deployed `dashboard.html` was never re-checked. Prevention: re-grep `dashboard.html` immediately after the 8c rename, and Phase 10's grep now targets `dashboard.html` (+ `build-data.js`) with the digit-safe `[A-Z0-9_]+` class (+ bracket markers). An engine Verify-step backstop fails the build on any `{{...}}` in the shipped files.

**F16 — Short product name (Frost, 2026-07).** `{{PRODUCT_NAME}}` was set to "Frost" instead of the full "Frost Security" — reads wrong in the topbar brand, partner mark, axis prose, and feedback card. Prevention: Phase 0 records `founder_name` as the FULL brand name (expand short dispatch inputs from `inputs/`); Phase 10 checklist verifies it in `dashboard.html`.

### What the reference build got right

- The 4-axis structure (Hiring + Opportunity + Inference Pain + Data Residency) was correct for the inference vertical and surfaced the wow exemplar (UHG) — the failure was rubric inflation, not axis selection.
- The Phase 6f checkpoint caught a 3.3% pass-rate criterion before the Webset fired — saved $5 and 10 minutes by re-firing with softened criteria.
- Founder voice splicing in gtm_thesis (verbatim Tom Amsterdam quotes from CONTEXT.md) materially improved the dashboard's tone over V1's hand-build.
- The interaction model and per-phase Git commits made the build auditable.

### Pattern: "Valar with names changed"

If your output for a non-inference founder reads like Valar with names changed — the same axis labels, the same persona descriptions, the same wow-evidence shape — you've under-delivered. The reference-build patterns are scaffolding for the discipline, not the content.
