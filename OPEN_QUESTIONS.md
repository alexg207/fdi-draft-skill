# FDI Skill — Open Questions

Things that need real-world testing, founder input, or user judgment to resolve. Not skill edits — design questions and validation gaps.

Last updated: post-`7154897` (Triple-audit Tier 1 + Tier 2 fixes complete).

---

## What hasn't been validated end-to-end

**1. Vertical generalization claim.** The skill claims to be vertical-agnostic after the recent refactor. Untested. The May 5 Valar V2 build is the only end-to-end run; that's the inference vertical. To confirm generalization works, someone needs to run the skill on:

- Another inference founder (control — should hit ≥80% V1 quality)
- A non-inference, non-tech-forward vertical (industrials predictive-maintenance, manufacturing IoT, climate hardware) — hardest test because Tier-2 source landscape diverges most
- A regulated-but-not-inference vertical (biotech / medtech / healthcare delivery) — tests whether the wow-evidence shape generalization actually works

If all three hit ≥75% V1 quality on the audit dimensions (Companies, ROW_SOURCES count, COMPANY_SOURCES median, JOB_LISTINGS coverage, gtm_thesis specificity, residency-axis discrimination), the skill is generalized. If industrials lands at 50%, the prose is more inference-biased than the refactor claims.

**2. Phase 7 subagent rule decay.** The subagent has 13 self-check items per company × 10 companies = 130 self-checks per build. Untested whether the subagent honors the late checks (companies 7-10) with the same rigor as early ones (1-3). One worth instrumenting: have the subagent emit a per-company self-check JSON object (`{company: "X", checks: {subtitle_under_18: true, ...}}`) so the main thread can verify nothing got skipped.

**3. Founder-pick research path effectiveness.** Phase 6m runs 4-5 directed Exa queries per founder-named pick not in Webset. Untested whether this actually produces ≥4 Tier-2 sources for a typical Tom-named-pick (e.g., HubSpot, Workday). Some companies have thin public engineering/research footprints; the queries may return generic Reuters redirects + the company's marketing site even with strong instructions.

**4. The "wow signal evidence shape" generalization.** Score-5 on the founder-specific axis requires "cited evidence in the wow signal's evidence shape." For inference, this is concrete (tried-and-blocked vendor). For industrials ("legacy-system reference"), biotech ("regulatory milestone"), fintech ("CFO earnings-call quote") — the shape language is in the skill but the *enforcement* is left to the model. Untested whether the subagent will actually distinguish a "legacy-system reference" from generic "uses old technology" filler.

**5. Sumble coverage for non-tech verticals.** Sumble is the primary hiring data source. Stated coverage is uneven for non-tech enterprises. Untested how often the fallback ladder (careers / LinkedIn / ATS) actually produces ≥1 verified job for a tier='high' company in industrials/biotech/healthcare.

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
- **`[SECTION_2_LABEL]` consistency**: literal "Inference Footprint" appearing in a non-inference build = F11 regression.
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
- `[SECTION_2_LABEL]` consistency (literal "Inference Footprint" appearing in non-inference build = F11 regression)
- Phase 7 subagent vs main-thread context split (did Phase 7 actually delegate?)
- End-to-end wall clock (target 30-50 min)
- End-to-end cost (target $7-15)

If all three verticals hit ≥75% V1 quality on the audit dimensions, the skill is generalized. If industrials lands at 50%, the prose is more inference-biased than the refactor claims and another iteration is needed.
