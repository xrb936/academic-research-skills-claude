# Cross-Agent Quality Alignment — Full Definitions

Unified definitions to prevent inconsistency across agents.

| Concept | Definition | Applies To |
|---------|-----------|------------|
| **Peer-reviewed** | Published in a journal with formal peer review process (editorial review alone does not qualify). Conference proceedings count only if explicitly peer-reviewed | bibliography_agent, source_verification_agent |
| **Currency Rule** | Default: published within 5 years. Override by domain: CS/AI = 3 years, History/Philosophy = 20 years, Law = depends on jurisdiction changes. Seminal works exempt regardless of age | bibliography_agent, ethics_review_agent |
| **CRITICAL severity** | IRON RULE: Issue that, if unresolved, would invalidate a core conclusion or constitute academic misconduct. Requires immediate resolution before pipeline can proceed | All agents |
| **Source Tier** | tier_1 = top-quartile peer-reviewed journal; tier_2 = other peer-reviewed; tier_3 = academic but not peer-reviewed; tier_4 = grey literature. **Chinese source mapping**: CSSCI 期刊 / 北大核心期刊 = tier_1; 其他中文同行评审期刊（非核心） = tier_2; 硕博论文（来自 985/211/双一流院校） = tier_2, 其他院校硕博论文 = tier_3; 中文会议论文（同行评审确认） = tier_2, 未确认同行评审 = tier_3; 报纸文章 / 内部报告 = tier_4 | bibliography_agent, source_verification_agent |
| **Minimum Source Count** | full = 15+, quick = 5-8, lit-review = 25+, systematic-review = all eligible (no limit), fact-check = 3+ per claim | bibliography_agent |
| **Verification Threshold** | 100% DOI check + 50% WebSearch spot-check. For Chinese-language references: S2 Tier 0 → CNKI supplementary check (if S2_NOT_FOUND) → Tier 1 DOI → Tier 2 WebSearch. `CNKI_VERIFIED` is equivalent to `S2_VERIFIED` in evidence strength | source_verification_agent, ethics_review_agent |

> **Cross-Skill Reference**: See `shared/handoff_schemas.md` for inter-stage data exchange formats.
