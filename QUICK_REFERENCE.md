# FDI Skill Quick Reference

Distillation of the most-used disciplines from `SKILL.md`. For full prose see `SKILL.md`. Print-friendly.

---

## The 14 Output Style Rules

Rules apply to every shipped artifact (CONTEXT.md, data.js, index.html, BUILD_NOTES.md). They don't apply to user-facing chat during the build.

| # | Rule | Quick test |
|---|---|---|
| 1 | Avoid em dashes | ≤1 per company entry |
| 2 | Default geo: US + Canada only | unless user expanded scope in Phase 0 |
| 3 | No filler descriptors | no "industry-defining", "best-in-class", "innovative" |
| 4 | Numeric ranges, not point estimates | "$2M–$5M" not "$3.5M" if estimated |
| 5 | Cite or omit | every numeric claim has ROW_SOURCES entry, or is dropped |
| 6 | Final dashboard: 9 or 10 (10 preferred) | 9 strong > 10 with one weak |
| 7 | No placeholder text in output | no "needs verification", "TBD", "?" |
| 8 | Subtitle ≤ 18 words | no parenthetical financial metadata |
| 9 | Pain Points: financial consequence first | first sentence is CFO language |
| 10 | Locked field sets per section | Profile=5, `[SECTION_2_LABEL]`=4, GTM=5 |
| 11 | Banned tag values | no `Stage-1 ICP`, `Pipeline`, `Enterprise`, `ICP` as tags |
| 12 | Source quality: 6-target / 4-floor | ≥1 Tier-1 primary; ≥2 Tier-2 vertical-credible; ≤1 Tier-3 |
| 13 | gtm_thesis personnel-durable | no named individuals; survive personnel changes |
| 14 | Reference build is scaffolding, not content | don't anchor on Valar — derive from this build's CONTEXT.md |

---

## Interaction Model

**One question at a time.** Never bundle questions. Auto-proceed on minor decisions; stop only for significant ones.

| Phase | Mode | If | Then |
|---|---|---|---|
| 0 preflight | **STOP** | Either Exa credit pool returns 402 | Halt, surface to user with `dashboard.exa.ai/api-keys` link |
| 0a-b | **STOP** | Setup questions | One question per turn, sequential |
| 0c | Auto | Inputs minimum present | Post inventory + continue |
| 0c | STOP | Required input missing | Ask once |
| 1 | Auto | Always | Post 5-8 line digest + continue |
| 2 | **STOP** | Per gap | One question per turn, walk priority order |
| 3 | Auto | Always | Post checklist + continue |
| 4 | Auto | Always | Write CONTEXT.md, commit, continue |
| 5 | Auto | Standard 4-axis fits cleanly from CONTEXT.md | Post axis plan + continue |
| 5 | STOP | Need to deviate from 4-axis structure | Ask |
| 6a-e | Auto | Always | Spec assembles deterministically |
| 6f | **STOP** | Always | Webset is paid + async; explicit go-ahead required |
| 6g-h | Auto | Unless 30%+ vendor false-positives | Continue to curation |
| 6h | STOP | 30%+ vendor false-positives | Ask whether to re-fire |
| 6i | **STOP** | Always | User has final cut authority on the 10 |
| 6j-m | Auto | Always | |
| 7 | Auto | Subagent runs | Main thread receives status report |
| 8-9 | Auto | Always | |
| 10 | Auto | Unless self-check fails | Ask whether to fix or ship |

**Status update format for auto-proceed:** 1-3 lines. Example:
```
✓ Phase 5 axes locked (Hiring + Opportunity + Process Pain + Capex Cycle for industrials).
  SECTION_2_LABEL: "Capex Footprint". HIRING_KEYWORD_REGEX: /predictive maintenance|...
  WOW_EVIDENCE_SHAPE: cited legacy-system reference with named end-of-life. Moving to Phase 6.
```

---

## The 5 Phase-5 Artifacts (consumed downstream)

Produce these before leaving Phase 5; they're consumed by Phases 6, 7, 8.

1. **4 axis definitions** (name + measures + sources + 0-5 rubric + Webset enrichment column). Hiring + Opportunity + 2 founder-specific. Founder-specific axes derived from CONTEXT.md ICP Qualifier + lookalike anchors + wow signal.

2. **`SECTION_2_LABEL`** (string). Vertical-named section 2 label. Examples: "Inference Footprint" / "Workflow Footprint" / "Capex Footprint" / "Compliance Footprint" / "Trial Footprint". Used in data.js sections + Phase 7 self-check + Phase 8 index.html.

3. **`HIRING_KEYWORD_REGEX`** (JS-compatible regex). Generated from CONTEXT.md pain language + Phase 5 axes + antagonist exclusions. Consumed by Phase 6j (Sumble filter), Phase 6j fallback ladder, Phase 8 (index.html computeJobSignal).

4. **`WOW_EVIDENCE_SHAPE`** (one-line description). What cited evidence in the wow signal's shape would look like. Consumed by Phase 7 step 12 (score-5 evidence requirement on the founder-specific axes).

5. **Segment structure** (3 segments default: Pipeline / Mid-Market / Enterprise; rename or modify per founder GTM motion).

---

## Phase 7 Self-Check (15 items, run after every company entry)

| # | Check | Pass condition |
|---|---|---|
| 1 | Subtitle ≤ 18 words | Count words in subtitle. No parenthetical financial metadata. |
| 2 | No placeholder text | grep entry for "needs verification" / "TBD" / "?" / "[insert" / "lorem". Zero matches. |
| 3 | Profile = 5 rows | Industry, Revenue, Employees, Cloud Provider, AI Maturity. No "Founded", no "Headquarters", no "[Founder] Status". |
| 4 | `[SECTION_2_LABEL]` = 4 rows | Use Cases, Current Stack, Pain Points, Estimated Spend. |
| 5 | Pain Points financial framing | First sentence = CFO language (margin/COGS/opex). No "X; Y; Z" enumeration. |
| 6 | No banned tag values | grep `tags[]` for "Stage-1 ICP", "Pipeline", "Target", "ICP". Zero matches. |
| 7 | gtm_thesis swap test | Strip company name. Could you swap any other in? If yes, rewrite. |
| 8 | gtm_thesis durability test | No named individuals. No "highest-warmth account" comparative claims. Buyer/Champion are role types. |
| 9 | gtm_thesis durability regex | `\b[A-Z][a-z]+ [A-Z][a-z]+\b` matches → must be in PRIMARY_TEAM or recognized firm/fund, else fail (catches target-company exec names like "Brian Schlise"). |
| 10 | Antagonist consistency | If gtm_thesis says **NOT [persona]**, CONTACT_MAP can't list that persona as champion. |
| 11 | Em dashes ≤ 3 | Count em dashes across subtitle, overview, gtm_thesis, all sections. |
| 12 | ROW_SOURCES citation density | ≥4 of 9 rows cited (V1 averages 5/10). |
| 13 | COMPANY_SOURCES count | Target 6, hard floor 4. ≥1 Tier-1 + ≥2 Tier-2 + ≤1 Tier-3. |
| 14 | Source titles describe content | Format: `[Outlet] — [Specific topic]`. Bare outlet names fail. |
| 15 | Word caps | overview ≤80, gtm_thesis ≤75, opp_reason ≤50, distress_reason ≤60, residency_reason ≤90. |

**Global checks (run once after all 10 entries, before commit):**

| # | Check | Pass condition |
|---|---|---|
| G1 | Tier distribution | At most 7 of 10 `'high'`; at least 1 `'low'`. |
| G2 | Axis uniformity | No single axis (signal_score, competitive_distress, data_residency, hiring sub-score) has ≥80% identical values across the 10. Catches F11-shape flatlines. |

---

## Reference build failure modes (V2 May 5 — what NOT to repeat)

| ID | Failure mode | Defense |
|---|---|---|
| F1 | Placeholder leak ("Estimated Spend: needs verification") | Output Rule #7 |
| F2 | Source thinness (Celonis 3 sources w/ PR aggregator anchor) | Output Rule #12 |
| F3 | Wow-axis score inflation (UHG drowned in 4s and 5s) | Phase 5 axis-4 rubric + Phase 7 step 12 |
| F4 | Personnel-fragile gtm_thesis (Capital One named 4 humans) | Output Rule #13 + Phase 7 self-check |
| F5 | Antagonist contradiction (Mastercard listed Kiran Jayant + said NOT Tech Governance) | Phase 7 antagonist consistency lint |
| F6 | Founder-pick research gap (16 of 18 founder-named picks under-researched) | Phase 6m founder-pick research path |
| F7 | JOB_LISTINGS empty (27 of 30 because Webset returned NULL) | Phase 6j Sumble + 4-step fallback |
| F8 | Citation density gap (2 of 12 cited vs V1's 5 of 10) | Phase 7 step 13 URL extraction discipline |
| F9 | Source-type stacking (4 of 5 criteria read regulatory text → European-bank skew) | Phase 6e source-type tagging requires ≥3 distinct tags |
| F10 | Field structurally not externally derivable (annualized inference spend) | Either drop the field or define a defensible 4-bucket enum |
| F11 | Hiring axis flatline from JOB_LISTINGS empty cascade (Lantern May 6: basic-Exa 402 → Phase 6j skipped → uniform-1 Hiring) | Phase 0 preflight credit-pool probe + Phase 6j Step 0 (mine Webset role-evidence first) + Phase 7 axis-uniformity self-check (G2) |
| F12 | Silent render breaker — apostrophe in single-quoted JS string → page stuck "Loading...", no console error | Avoid/escape apostrophes in copy; `node --check` the inline `<script>` after every copy edit |
| F13 | Two scoring systems disagreeing (tier from weighted composite vs displayed equal-weight total); `computeJobSignal` zeroed companies whose listings were bare `/careers` | One weighting in both composite + live `computeSignal`; derive tier from displayed score; deep careers URLs (v2 defaults #2, #5) |
| F14 | Fraunces display "f" looked broken; mono crept onto labels (AI-slop) | Space Grotesk display (v3; was Newsreader) / Inter UI / JetBrains Mono for numeric DATA ONLY (v2 default #4) |

---

## Dashboard visual defaults (v2 — required every build)

Copy proven code from `~/fdi/lantern-auto/dashboard.html`. Full prose: SKILL.md "Dashboard visual defaults (v2)" before Phase 8.

| # | Default | Test |
|---|---|---|
| 1 | Dark mode default + light toggle | tokenized `:root` + `[data-theme="light"]`; no-FOUC head script; sun/moon button; both pass WCAG AA |
| 2 | Score-quality color coding, NO RED | green=best / amber=below; tier derived from score (`>=75?'high':'med'`); bar + number + chip agree |
| 3 | "All" tab is the default view | `state.tab='all'`; All tab first w/ live count; enter = all cards ranked, not 3 |
| 4 | Type: display face / Inter UI / mono DATA-ONLY | Space Grotesk since v3 (was Newsreader; not Fraunces); never mono on labels/eyebrows/headings |
| 5 | Stored composite == live weights | one weighting both places; visible axis legend |
| 6 | reduced-motion + no mobile h-scroll | `@media (prefers-reduced-motion)`; `overflow-x` rule targets nav's real class |

**Landing/cover page = Phase 8b (OPTIONAL, off by default since v3.1).** The walkthrough is the entry page; build a landing only on explicit ask.

**Target accounts:** default 10 (3-30); from Hub/Slack dispatch input or Phase 0. Webset pulls ~1.5-2x target; Phase 6i curates to target; never hardcode 10 downstream.

**Scroll walkthrough = Phase 8c (STANDARD, auto) — the ENTRY page.** `template/build.html` copied VERBATIM as the build's `index.html` + `build-data.js` synthesized by subagent from config/CONTEXT/webset-spec/scored companies (schema: `template/build-data-template.js`; contract + copy rules: TEMPLATE_GUIDE Section 16). Hero frames account QUALITY not count; one axis carries `wowNote`; evidenceFeed = real citations only; network stays role-illustrative. Final flow every build: index.html (walkthrough, two-beat founder opener w/ floating 3D founder mark) -> dashboard.html, all Ember.

---

## Cost / time per build

- Webset (15 returns × 10 enrichments): ~$2-4, ~5-10 min
- Lovelace contacts (10 companies × 2 queries): ~$1-2
- Sumble (10 fetches): minimal
- Phase 6m founder-pick research (4-5 queries × N founder-named picks): ~$1-3
- Phase 7 emergency Exa fetches (~3 per company × 10): ~$1-2
- **Total: ~$7-15 per build, ~30-50 min wall clock** (subagent runs Phase 7 in parallel with main-thread auto-proceed flow)

---

## Phase boundary commits

Each phase ends with a Git commit. Default messages:

```
Phase 0: working directory setup, template cloned, build config saved
Phase 4: CONTEXT.md generated from raw docs
Phase 6g: Webset spec saved before submission
Phase 6h: Webset response captured (N companies)
Phase 6j: Sumble jobs gathered (N/M companies covered)
Phase 6k: Lovelace contacts gathered (N profiles across M companies)
Phase 6m: Founder-pick research saved (N companies)
Phase 7: data.js populated (N companies × 4 axes × 3 sections)
Phase 8: index.html customized for [vertical] axis labels and branding (+ v2 defaults: dark/light, color-coding, All tab)
Phase 8b: landing / cover page (optional, off by default)
Phase 8c: scroll walkthrough entry (index.html + build-data.js)
Phase 9: BUILD_NOTES.md documenting structural decisions
Phase 10: self-check fixes (if needed)
```

Per-phase commits make `git diff` between phases legible for audit and enable clean rollback if a phase produces bad output.
