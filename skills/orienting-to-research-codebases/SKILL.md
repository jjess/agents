---
name: orienting-to-research-codebases
description: Use when entering a large, heterogeneous, academic-research-grade codebase to orient yourself before making changes - prioritizes observation and documentation over modification
---

# Orienting to Research Codebases

## Overview

Academic and research codebases are unique ecosystems. They often contain:
- Multiple languages (Python + C++ + Fortran + MATLAB + R)
- Inconsistent conventions across sub-projects (different grad students, different years)
- Undocumented dependencies and "it works on my machine" configurations
- Historical artifacts ("we inherited this from the 1990s")
- Working code that looks "wrong" but can't be changed (publications depend on exact behavior)

**Core principle:** Observation before modification. Document the "why" before questioning the "how."

## When to Use

**ALWAYS use this skill when:**
- First encountering a large, unfamiliar codebase
- The code has "grown organically" over years
- Multiple authors with different styles
- You see files with extensions you don't recognize
- Dependencies look pinned to specific ancient versions
- Directory structure follows no obvious pattern

**Research mode is especially critical for:**
- Scientific computing code (HPC, simulations, data pipelines)
- Lab infrastructure (instrument control, data acquisition)
- Analysis code supporting published results
- Legacy systems being modernized

## The Research Mode Hard-Gate

<EXTREMELY-IMPORTANT>
You are in RESEARCH MODE. This is a "look but don't touch" phase.

**FORBIDDEN in Research Mode:**
- Refactoring to "clean up" code
- Changing naming conventions
- Upgrading dependencies
- Moving files to "better" locations
- Deleting "unused" code
- Writing new code before fully understanding existing code
- Running formatters or linters on existing files
- "Just fixing this one obvious bug"

**ALLOWED in Research Mode:**
- Reading files
- Mapping directory structures
- Documenting findings
- Asking clarifying questions
- Running tests to understand behavior
- Tracing execution flow
- Creating architectural diagrams
- Writing research notes

**If the user asks you to make changes:** Acknowledge the request, but respond with: "Before we modify this, let me finish documenting the current context so we understand what we're changing and why it exists this way."

Research mode ends when the user explicitly says "exit research mode," "let's make changes," or similar. Not before.
</EXTREMELY-IMPORTANT>

## The Archaeological Process

Think of research code archaeology. You're surveying a site before excavation.

### Phase 1: High-Level Survey (The Landscape)

First, understand the territory:

1. **Map the directory structure**
   ```bash
   tree -L 3 --dirsfirst
   # or: find . -maxdepth 3 -type d | head -50
   ```
   - Note: Unusual nesting patterns
   - Note: Multiple `src/` or `lib/` directories at different levels
   - Note: Language mixing (Python next to C++ next to shell scripts)

2. **Identify the build system(s)**
   Look for (in priority order):
   - `Makefile`, `CMakeLists.txt`, `setup.py`, `pyproject.toml`
   - `package.json`, `Cargo.toml`, `Makefile.PL`
   - `configure`, `build.sh`, `compile_all.sh`
   - Scripts that look like build scripts (often creatively named)

3. **Catalog languages and runtimes**
   - What files exist? (`.py`, `.cpp`, `.f90`, `.m`, `.R`, `.jl`, `.ipynb`)
   - What versions are pinned? (look at requirements, environment files)
   - Are there version conflicts? (Python 2 vs 3, different C++ standards)

4. **Find entry points**
   - `main()`, `if __name__ == "__main__"`, CLI scripts
   - Jupyter notebooks (often the real entry points in research)
   - Web interfaces, API endpoints
   - Configuration files that determine execution paths

**Output of Phase 1:** A "landscape report" documenting what exists.

### Phase 2: Dependency Archaeology

Research code often has complex dependency webs:

5. **External dependencies**
   - `requirements.txt`, `environment.yml`, `Pipfile`
   - System packages (`apt`, `yum`, `brew` lists)
   - External tools (compiled binaries, proprietary software)
   - Data dependencies (where does input data come from?)

6. **Internal dependencies**
   - Which modules import which?
   - Are there circular imports?
   - What's the "core" that everything depends on?
   - Are there optional dependencies? (features that only work if X is installed)

7. **Version archaeology**
   - Check `.git` history: When were dependencies last updated?
   - Look for `requirements.freeze.txt` or similar
   - Note version pins with suspicious specificity (`==1.2.3.post4+g12345`)
   - Git submodules? (often point to specific commits for reproducibility)

**Anti-Pattern to Avoid:** "These dependencies are old, I should upgrade them."
**Reality:** Old versions may be pinned for reproducibility (published results). Changing them invalidates comparison with prior work.

### Phase 3: Social Archaeology (The Human Layer)

Research code is social history:

8. **Identify stakeholders via git**
   ```bash
   git log --format="%an" | sort | uniq -c | sort -rn
   # Who touched this code recently?
   git log --since="6 months ago" --format="%an" | sort | uniq -c
   ```

9. **Find documentation (if any)**
   - `README*`, `CONTRIBUTING*`, `INSTALL*`
   - `docs/` directories
   - Comments at the top of files
   - Jupyter notebooks with explanatory text
   - Publication PDFs or links in comments

10. **Look for communication channels**
    - Issue trackers (GitHub/GitLab issues)
    - Slack/Discord channels mentioned in docs
    - Email addresses in git commits or file headers

### Phase 4: Code Quality Assessment (Non-Judgmental)

11. **Test infrastructure**
    - Do tests exist? (`test_*.py`, `*Test.cpp`, `tests/` directory)
    - Can you run them? (Often answer is "no" without significant setup)
    - Coverage? (Probably minimal in research code)

12. **Code health indicators** (document, don't judge)
    - File sizes (enormous files often indicate organic growth)
    - Function lengths
    - Comment density and quality
    - Presence of `TODO`, `FIXME`, `XXX`, `HACK` comments
    - Global mutable state
    - Commented-out code blocks

13. **Pattern inventory**
    - Naming conventions (even inconsistent ones - document them all)
    - Error handling patterns (or lack thereof)
    - Configuration approaches (env vars? config files? hardcoded?)
    - Logging (unified? scattershot? print statements?)

### Phase 5: Risk Assessment

14. **Identify sacred cows**
    - Code referenced in publications (DON'T TOUCH without extreme caution)
    - "Reference implementations" used by other groups
    - Data processing pipelines with regulatory implications
    - Anything labeled "validated" or "verified"

15. **Flag danger zones**
    - Database migrations
    - File format parsers (subtle bugs hard to detect)
    - Numerical algorithms (precision matters)
    - Random number handling (reproducibility)
    - Concurrency/parallelism (race conditions, deadlocks)

## Research Notes Template

Create a living document at `docs/superpowers/research-notes/YYYY-MM-DD-repo-orientation.md`:

```markdown
# Codebase Orientation: [Project Name]

**Date:** YYYY-MM-DD
**Researcher:** [Your name/ticket]
**Repository:** [Git remote URL]
**Current commit:** [SHA]

## Executive Summary

- **Primary language(s):** Python 3.8, C++14, some Fortran90
- **Size:** ~50K LOC, ~15 modules
- **Health status:** Functional but organic growth evident
- **Risk areas:** [List high-risk components]

## Directory Structure

[High-level map with notes on each major directory]

## Build System

[How to build/install, quirks noted]

## Key Dependencies

| Dependency | Version | Why This Version? | Risk |
|------------|---------|-------------------|------|
| numpy | 1.19.3 | Pinned for reproducibility (#123) | HIGH - don't upgrade |

## Entry Points

- `src/main.py` - Primary analysis pipeline
- `notebooks/exploratory/` - Jupyter workflow
- `scripts/batch_process.sh` - HPC batch jobs

## Conventions (Inconsistent)

- **Python:** Mixed snake_case and camelCase
- **C++:** Some files use tabs, some spaces
- **Naming:** `temp` variables everywhere (not actually temporary)

## Sacred Cows (DO NOT MODIFY)

- `src/legacy/simulation_core.py` - Used in Nature 2023 paper
- `data/reference_sets/` - Validated against external datasets

## Open Questions

- Why is scipy pinned to 1.5.2 specifically?
- What does `misc/` directory contain?
- Who maintains the HPC scripts?

## Exit Research Mode Checklist

Before making any changes:
- [ ] All risk areas documented
- [ ] Sacred cows identified
- [ ] Dependencies mapped
- [ ] Tests run (even if failing)
- [ ] User approves exit from research mode
```

## Red Flags - STOP and Document

**If you find yourself wanting to:**
- Run a code formatter on existing files
- "Quickly fix" obvious bugs
- Upgrade dependencies
- Move files to "clean up" the structure
- Rename variables to follow "best practices"
- Delete dead code (you think is dead)
- Add type hints to everything
- Rewrite in a "better" language

**STOP.** These are natural impulses, but they're destructive in research mode. Document the finding instead:

❌ "I'm going to fix the indentation in analysis.py"
✅ "ANOMALY: analysis.py uses 2-space indents while rest of codebase uses 4. Documenting in case it matters."

## Integration with Other Skills

**After exiting research mode:**
- Use `goose-superpowers:brainstorming` to plan changes
- Use `goose-superpowers:systematic-debugging` for any bugs found
- Use `goose-superpowers:test-driven-development` for new code (add tests first!)
- Use `goose-superpowers:writing-plans` for implementation plans

**Research mode can invoke:**
- `goose-superpowers:requesting-code-review` - If you want review of your research notes
- `goose-superpowers:systematic-debugging` - If you encounter broken code while exploring

## The Goal

By the end of research mode, you should be able to answer:
1. What is this codebase for?
2. How does it work (at coarse-grained level)?
3. What depends on what?
4. What's safe to change vs. what's sacred?
5. Who do I ask if I'm unsure?

You should NOT have modified any source files (except creating research notes in `docs/`).

## Remember

Research code is often "write-once, read-many, modify-never." Respect that. Your job is to understand, not to beautify. The messiness exists for reasons—document those reasons so future maintainers (including yourself in 6 months) understand the archaeology.

**Exit command:** The user can exit research mode by saying "exit research mode," "let's make changes," "I'm ready to modify," or similar explicit statements. Until then: observe, document, resist the urge to fix.
