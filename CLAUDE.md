# solution-architecture-lab

Public MkDocs site at `/solution-architecture-lab/`. Group rules in `../CLAUDE.md` apply.

## Purpose
Demonstrate solution-architect thinking (not just infrastructure): business context → requirements → options → decision → trade-offs. Audience: hiring managers and architects reviewing my portfolio.

## Content model
- Each case study follows: Context & drivers · Requirements (functional, NFRs, constraints) · Options considered · Decision (ADR) · Target architecture (C4: context → container) · Risks & trade-offs · What I'd do differently.
- Frameworks referenced: TOGAF ADM phases, C4 model, ADRs, Well-Architected pillars. Use them to structure, not to pad.

## Quality bar
- Every decision states the alternative rejected and why.
- NFRs are quantified (e.g., RTO/RPO, p95 latency, availability target).
- Prefer one deep, honest case study over several shallow ones.

## Build
`mkdocs build --strict` must pass. Update `nav:` in `mkdocs.yml` for new pages.
