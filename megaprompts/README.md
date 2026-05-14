# Claude Skills Mega Prompts (v2 — 10 skills)

Production-grade mega prompts for generating a polished, distributable Claude skills library. Each mega prompt instructs Claude Code (or Claude.ai) to produce one skill file, with consistent quality standards, error handling, and portability across CLI + web contexts.

## Files

|File                                       |Purpose                                                    |Category    |
|-------------------------------------------|-----------------------------------------------------------|------------|
|`00-master-orchestrator.md`                |Chains all 10 mega prompts; supports full/pack/single modes|Orchestrator|
|`01-last-30-days-megaprompt.md`            |Multi-source research (Reddit + HN + Web + X)              |Core        |
|`02-take-a-step-back-megaprompt.md`        |Mid-conversation reflection                                |Core        |
|`03-notebooklm-megaprompt.md`              |NotebookLM browser automation                              |Core        |
|`04-landing-page-megaprompt.md`            |Premium HTML landing page generator                        |Core        |
|`05-brain-dump-megaprompt.md`              |Brain dump capture + organization                          |Core        |
|`06-email-setup-megaprompt.md`             |Email triage onboarding (paired with #7)                   |Email       |
|`07-email-triage-megaprompt.md`            |Email triage execution (paired with #6)                    |Email       |
|`08-consensus-grant-finder-megaprompt.md`  |NIH grant research (Consensus + RePORTER)                  |Research    |
|`09-literature-review-helper-megaprompt.md`|Strategic literature review (PICO/SPIDER)                  |Research    |
|`10-recommended-reading-list-megaprompt.md`|Course syllabus → reading list                             |Research    |

## Quality Standards (Applied Across All 10)

1. **Token efficiency** — Each generated skill targets ~2,000 words; research-pack skills allowed up to 2,800 (information-dense by nature)
1. **Distributable** — No personal references, no hardcoded paths, no brand-specific content
1. **Production error handling** — Every external dependency has documented failure modes + recovery
1. **Portable** — Works in both Claude Code CLI and Claude.ai web (with explicit notices when CLI-only)
1. **Convention consistency** — Same frontmatter format, section structure, tone within categories
1. **Research-pack conventions** — Skills 8-10 share Consensus rate-limiting, plan-tier detection, source discipline, audit log standards

## How to Use

### Option A: Generate the full library

```bash
# From your skills project root
claude-code "Execute the master orchestrator at ./megaprompts/00-master-orchestrator.md. Set SKILLS_DIR=./claude-skills. Mode: full."
```

### Option B: Generate a specific pack

```bash
# Just the research pack (skills 8-10)
claude-code "Execute the master orchestrator at ./megaprompts/00-master-orchestrator.md. Set SKILLS_DIR=./claude-skills. Mode: --pack=research."

# Just the email pack (skills 6-7, must run together)
claude-code "Execute the master orchestrator at ./megaprompts/00-master-orchestrator.md. Set SKILLS_DIR=./claude-skills. Mode: --pack=email."

# Core productivity skills (1-5)
claude-code "Execute the master orchestrator at ./megaprompts/00-master-orchestrator.md. Set SKILLS_DIR=./claude-skills. Mode: --pack=core."
```

### Option C: Generate one skill at a time

```bash
claude-code "Execute the mega prompt at ./megaprompts/08-consensus-grant-finder-megaprompt.md. Output to ./claude-skills/consensus-grant-finder/SKILL.md."
```

### Option D: Use in Claude.ai web

Paste any mega prompt into a Claude.ai chat. The output skill will be delivered as an artifact. Save the artifact text to your local skills directory.

## Customization Points

Before running, override these as needed:

|Variable         |Default           |Purpose                                     |
|-----------------|------------------|--------------------------------------------|
|`${SKILLS_DIR}`  |`./claude-skills/`|Where generated skills land                 |
|`${OUTPUT_DIR}`  |`./landing-pages/`|(Landing page) Where generated HTML files go|
|`${RESEARCH_DIR}`|`./research/`     |(Last-30-days) Where briefings save         |
|`${WORKSPACE}`   |`./`              |(Email skills) Where KB lives               |

## Pack Notes

### Core Pack (1-5)

General productivity skills. Mostly tool-light. Skills 2 (reflection) and 5 (brain dump) work without any external tools. Skill 3 (NotebookLM) requires browser automation. Skill 4 (landing page) outputs HTML. Skill 1 (last-30-days) uses web search + (optionally) browser automation for X/Twitter.

### Email Pack (6-7) — Paired

The knowledge base file contracts MUST match between setup and triage. Always generate setup first, triage second. The orchestrator validates this; manual single-skill generation requires the same order.

### Research Pack (8-10) — Shared Infrastructure

All three skills use:

- **Consensus MCP** for academic search
- **`docx` Node.js library** for document generation
- **Sequential execution** (1 query/sec rate limit)
- **Plan-tier detection** (free ~10/search, Pro ~20/search)
- **Source discipline** (only cite session tool-call results)
- **Three-count audit** (sent / received / cited)

The master orchestrator validates these conventions are consistent across all three. Generate the pack together for best consistency.

Additional research-pack dependencies:

- Skill 8 also needs `bash_tool` + `curl` (RePORTER POST API) and `web_fetch` (NOSI HTML)
- Skill 10 ships a bundled JavaScript helper script (`scripts/generate_reading_list.js`)

## Portability Notes Per Skill

|Skill              |Claude Code CLI|Claude.ai Web                               |
|-------------------|---------------|--------------------------------------------|
|01 last-30-days    |✅ Full         |✅ Most phases; X requires browser automation|
|02 take-a-step-back|✅ Full         |✅ Full                                      |
|03 notebooklm      |✅ Full         |❌ Requires browser automation               |
|04 landing-page    |✅ Full (file)  |✅ Full (artifact)                           |
|05 brain-dump      |✅ Full         |✅ Full (workspace detection may differ)     |
|06 email-setup     |✅ Full         |✅ With project files                        |
|07 email-triage    |✅ Full         |✅ With Gmail/Outlook MCP                    |
|08 grant-finder    |✅ Full         |✅ With Consensus MCP + Code Execution       |
|09 lit-review      |✅ Full         |✅ With Consensus MCP + Code Execution       |
|10 reading-list    |✅ Full         |✅ With Consensus MCP + Code Execution       |

## Cross-Skill Validation (Automatic via Master Orchestrator)

- No conflicting trigger phrases between skills
- Email setup/triage file contracts align
- Research-pack conventions consistent (rate limit, plan tier, audit, sources)
- Consistent voice and structure across all skills

## Next Steps After Generation

1. Test triggers in a fresh conversation per skill
1. Add the skills to your Claude project or `~/.claude/skills/`
1. For the research pack: verify Consensus MCP connection and `docx` install path
1. Iterate based on real usage — the master orchestrator can re-generate any single skill
1. Consider publishing the library publicly once polished

## Anti-Patterns Already Rejected by Default

All mega prompts already filter out:

- Hardcoded absolute paths
- Personal/brand references
- Single-tool lock-in without fallbacks (where reasonable)
- Vague trigger phrases
- Wall-of-text sections
- Pseudocode that won’t execute as written
- Inconsistent rate-limit / retry / audit conventions across the research pack

The goal is to build these skills systematically and avoid duplicates.
