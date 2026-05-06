# FDI Skill — Open Questions

Things that need real-world testing, founder input, or user judgment to resolve. Not skill edits — design questions and validation gaps.

Last updated: post-Lantern-FDI-2026-05-06 audit (5 patches: Phase 6j Step 0 Webset-mining, Phase 0 credit-pool preflight, Phase 7 axis-uniformity self-check, gtm_thesis durability regex, F11 entry).

---

## Lantern FDI 2026-05-06 — validation results

Third end-to-end run of the skill (Valar V2 May 5 → Plural May 6 → Lantern May 6). Lantern is **AI purchasing agents for wholesale distributors** — first distribution-vertical build (HVAC / plumbing / electrical / industrial supply mid-market). Validation results vs prior Q's:

- **Q1 (vertical generalization)** — **further validated**. Section 2 cleanly relabeled to "Buy-Side Footprint", axes "Operational Pain" + "ERP Fit", HIRING_KEYWORD_REGEX scoped to Buyer / Purchasing Manager / Demand Planner titles. Webset returned 15 ICP-correct distributors with zero vendor false-positives. ICP correctness confirmed by user-side SDR review.

- **Q2 (subagent rule decay)** — **mostly held**. Em-dash count averaged 0.7 per company (vs 28.7 in Plural and 16.6 in Valar V1) — strongest stylistic improvement to date. Word caps held under ceilings. BUT 4 of 10 entries violated Pain Points "lead with financial consequence" rule, and 2 of 10 entries violated `gtm_thesis` durability (named "Brian Schlise", "Marco Schooley" — target-company execs). Subjective self-checks weren't enough; objective regex self-check now codified for durability (Pain Points framing held subjective per user judgment).

- **NEW Q14: Hiring axis flatline cascade** — **F11 catalogued**. Basic-Exa MCP hit 402 mid-build; Phase 6j skipped entirely (Sumble + careers + LinkedIn + ATS fallback ladder all depend on basic Exa). All 10 JOB_LISTINGS empty → `computeJobSignal()` default returned uniform 1 across the dashboard. Three preventions now in skill: Phase 0 credit-pool preflight, Phase 6j Step 0 (mine Webset role-evidence first), Phase 7 axis-uniformity self-check.

- **NEW Q15: credit-pool distinction not surfaced**. Websets MCP and basic Exa MCP share the dashboard but bill on separate pools. Either can hit 402 while the other works. Now documented in Prerequisites + probed in Phase 0 preflight.

- **NEW Q16: Profile field set leakage to non-tech-forward verticals**. Cloud Provider + AI Maturity rows carry weak signal for distribution / industrials / healthcare-delivery / manufacturing builds. User decision (this iteration): keep universal field set; defer vertical-aware Profile presets. Re-open if observed in subsequent non-tech build.

---

## Plural FDI 2026-05-06 — validation results

The Plural FDI build was the second end-to-end run of the skill (after Valar V2 May 5). Plural is the **first non-inference vertical** the skill has been tested against — Kubernetes fleet management for regulated-finance + healthcare-payer + telecom enterprises. Validation results vs the open questions:

- **Q1 (vertical generalization)** — **partially validated**. The skill produced a credibly-vertical-shaped Plural dashboard: Section 2 cleanly relabeled to "Kubernetes Footprint", axes correctly named "Kubernetes Fleet Pain" + "AI Mandate × Governance", HIRING_KEYWORD_REGEX scoped to Platform Engineering / SRE / AI Platform titles. Webset returned 3 ICP-correct companies (JPMorgan, RBC, Manulife) with strong primary sources. ICP correctness confirmed by user. Body-copy quality regressed (see Q2 below).

- **Q2 (subagent rule decay)** — **validated as a real issue**. The Phase 7 subagent ignored Output Style Rule #1 (em dashes ≤1) by a 28× factor (28.7/co vs ceiling 1) and ignored implicit length expectations (gtm_thesis 132w avg vs Valar V1's 40w). Per-company self-check JSON emission, now added to SKILL.md Phase 7, is the fix.

- **Q3 (founder-pick research effectiveness)** — **partially validated**. With 6 founder-named picks needing research and the original 4-5 query-per-company budget = 24-30 queries, Plural build had to compress to 2-3 queries per company. Quality held (4-5 Tier-2 sources per company achieved). Skill now has a scale-down rule when picks > 5.

- **Q4 (wow shape generalization)** — **validated**. The hybrid Wow Shape C ("K8s fleet pain + AI governance friction") worked end-to-end. Score-5 evidence rubric required cited 50+ clusters AND named senior AI exec AND governance friction quote — 5 of 10 Plural companies hit it (JPMorgan, RBC, Manulife, Barclays, Kaiser); 5 capped at 4 with documented reasons. The wow-shape generalization holds for non-inference verticals when the founder's wow is articulated cleanly in CONTEXT.md.

- **Q5 (Sumble coverage non-tech)** — **deprecated for skill V2**. Plural build skipped the Sumble dependency entirely; Phase 7 subagent did inline `WebSearch` + `WebFetch` for hiring evidence per company during data.js population. Worked fine. Skill could note that Sumble is optional when the Phase 7 subagent has WebSearch/WebFetch access (most builds do).

- **NEW Q11: Webset slow-search early-cancel triggers**. Surfaced by the Plural build: K8s-50+-clusters criterion dropped from 11.9% → 6.5% pass rate; canceling at 41% complete with 4 returns was net-positive vs. waiting 30+ more minutes for marginal additions. Skill now codifies early-cancel rules in Phase 6g.

- **NEW Q12: existing Webset intersection-mining**. Plural build triangulated against pre-existing persona Websets in the workspace (Buckets A/B/C from May 5 SRE/AI/Platform campaigns) for company-frequency curation signal. Worth generalizing: when the workspace has prior persona Websets for the same founder, mine them at Phase 6h-6i for cross-Webset company density. Currently informal; could be Phase 6 step.

- **NEW Q13: opp_reason challenge acknowledgment was missing as a rule**. Valar V1 Mastercard pattern (flagging in-house competing capability) wasn't codified; Plural's opp_reasons were uniformly bullish. Now Output Style Rule #17.

---

## What hasn't been validated end-to-end

**1. Vertical generalization claim — partially validated by Plural build (regulated-finance + healthcare + telecom).** Still unvalidated:

- Industrials predictive-maintenance, manufacturing IoT, climate hardware — hardest test because Tier-2 source landscape diverges most
- Biotech / medtech — tests whether the wow-evidence shape generalization extends to FDA filings / clinical trial registries as primary records
- Pure consumer / D2C — tests whether B2B SaaS-shaped Profile rows ("Cloud Provider", "AI Maturity") still serve

If next two verticals hit ≥75% V1 quality, generalization claim is fully resolved.

**2. Phase 7 subagent rule decay — VALIDATED AS REAL.** Plural build confirmed the issue (gtm_thesis bloat, em-dash ignore). Fix shipped (per-company self-check JSON in SKILL.md Phase 7). Untested whether the JSON-emission instrumentation is actually honored by future subagent runs — the meta-question is whether the subagent will skip emitting the JSON when it would expose its own rule violations.

**3. Founder-pick research path effectiveness — partially validated.** Plural build had 6 picks; 2-3 queries/co produced ≥4 Tier-2 sources per company in 5 of 6 cases. The 6th (DTCC) hit exactly 4 sources at the floor. Untested for verticals where private-company primary records are harder to find (early-stage biotech, consumer brands without SEC presence).

**4. The "wow signal evidence shape" generalization — VALIDATED for hybrid shapes (Plural Wow C).** Untested:

- Industrials "legacy-system reference" (e.g., 10-K cites $X annual unplanned-downtime cost)
- Biotech "regulatory milestone" (e.g., FDA submission pending on a single-vendor pipeline)
- Fintech regtech "CFO earnings-call quote about a named compliance gap"

Single-evidence-shape verticals haven't been run yet.

**5. Sumble coverage for non-tech verticals — DEPRECATED.** Plural build skipped Sumble entirely; Phase 7 subagent did inline WebSearch for hiring per company. Worked fine. Sumble is optional when the Phase 7 subagent has direct WebSearch + WebFetch access. Update SKILL.md Phase 6j to say: "Sumble is preferred-when-available; if the build is running with WebSearch/WebFetch in the Phase 7 subagent, you may skip Phase 6j entirely and let the subagent fetch hiring per company during data.js population."

---

## Design questions that don't have an obvious right answer

**6. Should the 10-company target be vertical-aware?** Tech-forward verticals have lots of public source content per company; you can credibly research 10 deeply. Industrials/biotech may only support 6-7 deeply-researched entries. The skill currently says "10 (or 9 if you can't reach the floor)." Would 6-8 be a better default for source-thin verticals? Trade-off: smaller dashboards land less but read more credible.

**7. SECTION_2_LABEL — should the model pick or should the user?** The skill has the model pick at Phase 5. Pro: the model knows the vertical's natural language. Con: the model might pick something off-tone for the founder's actual GTM messaging. Worth adding a Phase 5 confirmation question if the standard label doesn't fit cleanly?

**8. Locked field sets (Profile=5, Section 2=4, GTM=5).** Inherited from V1 Valar. Untested whether the field shapes generalize. For a B2C consumer founder, "Cloud Provider" doesn't make sense; could be "Distribution Channels" or similar. The 5-row Profile is locked but the field names assume B2B SaaS-shaped data. Worth parameterizing the rows themselves, not just Section 2's name?

**9. Antagonist persona consistency check — is the lint smart enough?** Currently grep-based: if gtm_thesis says `**NOT** Technology Governance`, CONTACT_MAP can't list a contact with "Technology Governance" in their title. Real-world failure mode: the antagonist might be a *function* (e.g., "ML engineering") and the contact title might be "Senior ML Architect" — semantically the same but no string match. The lint will miss this. Would benefit from a semantic check (subagent task) but adds complexity.

**10. Geographic scope vs source-type diversification interaction.** Default geo is US + Canada. For a global founder, source-type diversification (`[regulatory]`, `[earnings]`, `[engineering]`, etc.) should pull from non-US press too. Current skill doesn't address this. May surface as European trade press getting tagged `[regulatory]` because of GDPR even when the founder isn't regulated-vertical.

---

## What the next build run will tell us

Variables to track on the next end-to-end run (whether Valar refresh or new founder):

- **Final company count**: 9 or 10? (Output Rule #6 expects 9-10, with 9 acceptable)
- **Tier distribution**: target ~5h / 4m / 1l. If everything ends up high or low, tiers carry no information.
- **Sources per company (median)**: target 6, hard floor 4. Below 4.5 median = source-discipline failed.
- **JOB_LISTINGS coverage**: target ≥9 of 10 companies populated with ≥1 verified role.
- **gtm_thesis name-strip survival rate**: spot-check 3 entries; target 3 of 3 surviving the swap test.
- **Score distribution on founder-specific axes**: should be a real spread 0-5, not all 4-5. If everything is 4 or 5, F3 score-inflation regression.
- **`[SECTION_2_LABEL]` consistency**: literal "Inference Footprint" appearing in a non-inference build = section-2-label-leakage regression (Output Style Rule #14 violation).
- **Subagent vs main-thread context split**: did Phase 7 actually run in a subagent, or did the main thread try to load all the JSON?
- **End-to-end wall clock**: target ~30-50 min. >90 min suggests rule application is blocking flow.
- **End-to-end cost**: target $7-15. >$25 suggests excessive Exa fetches.

---

## Process gaps not yet addressed

These are real but the user hasn't asked to close them yet:

**A. CONTEXT.md gets out of sync with subsequent edits.** Phase 4 produces CONTEXT.md from inputs. Later phases (5-7) sometimes reveal new founder context that should backflow. There's no mechanism to keep CONTEXT.md current. Workaround: Phase 9 BUILD_NOTES.md captures deltas. Better: Phase 10 self-check could include a CONTEXT.md freshness check.

**B. Phase 6f checkpoint runs only once.** If the user changes their mind after the Webset fires (Phase 6g), there's no clean re-fire path with a tweaked spec. The only option is to manually cancel + delete the webset (we did this in V2 build) and restart Phase 6g. Worth codifying as a documented sub-flow.

**C. The skill assumes one founder per build.** No support for multi-founder verticals (e.g., founder + cofounder + CTO each having distinct buyer profiles). Edge case but possible.

**D. No automated end-to-end smoke test.** The skill has 13+ self-check items but no orchestrating script that runs them all + reports pass/fail. A `make audit` or similar would be worth ~50 lines of bash.

**E. Vertical playbook self-seeding (audit recommendation #4).** When a build for a novel vertical succeeds, there's currently no mechanism to capture the per-build axes/personas/sources/wow shape and promote them to canonical knowledge for future builds. Each new vertical starts from scratch. Could be a one-line prompt at end of Phase 10: "Save this build's vertical fingerprint to a catalog file for future reference?"

---

## What to do with this file

Update after each end-to-end run. Move resolved questions to a `CHANGELOG.md` or delete. New questions get appended. Goal: keep a tight backlog of uncertainty, not let the skill claim certainty it hasn't earned.

---

## Tier 3 audit findings (deferred — need user judgment before fixing)

The triple audit (cross-repo, cross-reference, dry-run) surfaced these issues. Tier 1 (deterministic fixes) and Tier 2 (safe surgical edits) were applied. Tier 3 items were held because they require judgment about template-rendering side effects or substantial new content the user should review.

### T3-1: data.js placeholder rewrite ✅ RESOLVED (commit `ef4ace4` in fdi-template)
Acme Corp rewritten to canonical 5/4/5-row demonstration. Banned rows (Founded, Headquarters, "[Founder] Status") removed. Other 4 placeholders (Beta, Gamma, Delta, Epsilon) normalized to same locked shape. Section 2 placeholder renamed `{{OPPORTUNITY_SECTION_TITLE}}` → `{{SECTION_2_LABEL}}`. node --check passes.

### T3-2: index.html `{{...}}` placeholder conversion ✅ RESOLVED (commit `ef4ace4` in fdi-template)
9 hardcoded inference-vertical strings converted to 8 new `{{...}}` placeholders: `{{AXIS1_LABEL}}`, `{{AXIS1_DESCRIPTION}}`, `{{AXIS2_LABEL}}`, `{{AXIS2_DESCRIPTION}}`, `{{HIRING_AXIS_DESCRIPTION}}`, `{{HIRING_FALLBACK_TEXT}}`, `{{HIRING_KEYWORD_REGEX}}`, `{{SEGMENT_MIDMARKET_SUBTITLE}}`. Phase 8 grep validation now self-validating: any remaining `{{...}}` = bug. SKILL.md Phase 8 step list rewritten to enumerate all 10 placeholders with sources/values.

Final scan: zero inference-vertical hardcoded strings in index.html outside the JS-comment legend (lines 526-527, intentional documentation).

### T3-3: Add non-inference worked examples to TEMPLATE_GUIDE Section 9 (MEDIUM)
Section 9's worked examples are 100% Valar/inference. Output Style Rule #14 says "if the output sounds like Valar with names changed, you've under-delivered" — but the only craft reference Phase 7 sends the subagent to is 100% inference. The anti-anchor callout (just added) helps but isn't a substitute for parallel non-inference examples.

**Fix:** Add 1-2 worked examples per Section 9 subsection from a healthcare-workflow or industrials build. Sourcing the examples is the bottleneck — could synthesize from public companies, or wait until a real non-inference build produces them.

**Why deferred:** Substantial new content; better done from real build outputs than synthesized. ~2-3 hours of work to do credibly.

### T3-4: SECTION_2_LABEL example expansion in SKILL.md (LOW)
Phase 5 lists "Capex Footprint" for industrials greenfield but doesn't cover predictive maintenance specifically. Dry-run audit flagged this — a predictive-maintenance founder will either misuse "Capex Footprint" or invent a new label without confidence.

**Fix:** Add a 6-8 row vertical → label table in Phase 5 covering predictive maintenance ("Reliability Footprint" or "Uptime Footprint"), supply chain ("Throughput Footprint"), energy ("Asset Footprint"), manufacturing automation, etc.

**Why deferred:** Risks inflating the skill back toward per-vertical templates that the refactor explicitly removed. Better to let the model derive the label from CONTEXT.md and post a status update than encode a table. ~15 minutes of work, but design call to make first.

### T3-5: Phase 8 antagonist-aware regex enforcement (MEDIUM)
Phase 5 produces `HIRING_KEYWORD_REGEX` with antagonist exclusions in mind, but the example regexes shown don't model antagonist subtraction — a careless subagent might paste an example regex verbatim, including roles the founder flagged as antagonists.

**Fix:** Phase 5 artifact #2 should require an explicit "excluded terms (antagonist)" comment block alongside the regex. Phase 8 should grep the final regex for any antagonist-role keywords as a final check.

**Why deferred:** ~10 minutes of edit, but worth verifying behavior on a real non-inference build first to ensure the rule actually catches the failure mode.

### T3-6: 15 self-check item naming consistency (LOW)
The Phase 7 self-check has 15 items but the SKILL.md narrative references some by name ("the durability test", "the antagonist consistency check") and others by description. Naming all 15 consistently would help the subagent verify completeness.

**Fix:** Add a name to each self-check item bullet (e.g., `**Subtitle test:** ≤ 18 words...`, `**Placeholder leak test:** No "needs verification"...`, etc.). Then "the subagent must run all 15 named tests" becomes lintable.

**Why deferred:** ~20 minutes of edit, low priority.

---

## Validation backlog (run before declaring "vertical-agnostic" complete)

After Tier 3 is closed, run end-to-end on:
1. Another inference founder (control)
2. An industrials predictive-maintenance founder (test — Tier-2 source landscape diverges most)
3. A biotech/medtech founder (hardest test — wow shape and source landscape both diverge)

Track per build:
- Final company count (target 9-10)
- Tier distribution (5h/4m/1l)
- Sources per company median (target 6, hard floor 4)
- JOB_LISTINGS coverage (target ≥9/10)
- gtm_thesis name-strip survival (target 3/3 spot-checks)
- Score distribution on founder-specific axes (real spread 0-5, not all 4-5)
- `[SECTION_2_LABEL]` consistency (literal "Inference Footprint" appearing in non-inference build = section-2-label-leakage regression, Output Style Rule #14 violation)
- Phase 7 subagent vs main-thread context split (did Phase 7 actually delegate?)
- End-to-end wall clock (target 30-50 min)
- End-to-end cost (target $7-15)

If all three verticals hit ≥75% V1 quality on the audit dimensions, the skill is generalized. If industrials lands at 50%, the prose is more inference-biased than the refactor claims and another iteration is needed.
