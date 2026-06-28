# release-doi

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/release-doi) [![GitMCP](https://img.shields.io/endpoint?url=https://gitmcp.io/badge/shimo4228/release-doi)](https://gitmcp.io/shimo4228/release-doi) [![View Code Wiki](https://assets.codewiki.google/readme-badge/static.svg)](https://codewiki.google/github.com/shimo4228/release-doi)

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that runs the **release workflow for DOI-registered research repositories** following the identifier-federation triplet (ADRs 0001-0003) of the [authorship-strategy](https://github.com/shimo4228/authorship-strategy) research line. Sequences pre-release verification, tag-push, archive deposit, DOI propagation, and cross-platform federation update steps as a single five-phase runbook that prevents the most common release-time drift incidents (off-by-one canonical-reference mistakes, missed sibling cross-reference updates, accidental version-DOI canonicalization).

The skill is **harness-aware where it has to be** (it knows about Git, GitHub, Zenodo's auto-deposit-on-tag webhook, and CITATION.cff / `.zenodo.json` formats) but **harness-neutral where it can be**: the verify and deposit steps are written so an adopter using a different archive service (Software Heritage, OSF, Figshare) or a different release tooling chain can substitute the equivalent operations without re-deriving the runbook.

## When to use

Apply the skill when **all** of the following hold:

- The repository is registered (or about to be registered) with a versioned DOI archive (Zenodo is the reference example)
- The release follows a tagged-release-triggers-deposit model (typical for Zenodo's GitHub integration)
- The repository participates in a federation of sibling DOI-registered artifacts (`.zenodo.json` `relatedIdentifiers` declarations, cross-platform dataset mirrors)
- The release author intends to maintain the **concept DOI as canonical reference** discipline (ADR-0001 in the upstream research line)

Skip the skill for:

- Repositories without DOI registration (the workflow's central artifact, the DOI, does not exist)
- One-shot artifacts published without versioning intent (the version/concept DOI distinction does not apply)
- Repositories where the release author has explicitly opted out of the identifier-federation triplet (the skill's assumptions do not hold)

## Install

### Claude Code

```bash
git clone https://github.com/shimo4228/release-doi
cp -r release-doi/skills/release-doi ~/.claude/skills/release-doi
```

No runtime dependencies for the skill itself; verification commands inside use `git`, optional `python3 -m json.tool` for JSON-LD validation, and optional `cffconvert` for CITATION.cff syntax checking. The skill invokes the harness's release tooling (`git tag`, `gh release create` or equivalent) rather than wrapping them.

### Other harnesses

Adapt the install path to your harness's skill convention. The skill's runbook is written in a form that does not depend on Claude Code specifically; the trigger and invocation mechanism is the only harness-specific surface.

## How it works

The skill walks through a pre-flight gate, five sequential phases, and one post-release phase. Each phase has an explicit completion condition; the skill refuses to advance until the condition is met.

0. **Pre-flight: Zenodo opt-in check** — verifies the Zenodo GitHub webhook is registered on the target repository (`gh api repos/<owner>/<repo>/hooks`). Zenodo's GitHub integration is **repo-by-repo opt-in**; releases cut before the toggle is enabled are not retroactively archived. This gate catches the gap on first release of a new DOI repository — a blind spot when sibling repos are already opted in.
1. **Phase 1: Baseline** — capture the pre-release ground truth (commit count, file count, test count where applicable). This is the reference state against which Phase 4 verifies.
2. **Phase 2: CODEMAPS regeneration** — re-generate file-level architecture maps so they reflect the to-be-released state.
3. **Phase 3: Cross-document consistency** — sync CHANGELOG / CITATION.cff / `.zenodo.json` / `pyproject.toml` (where applicable) / multilingual README / llms.txt / glossary / ADR bidirectional links. Drift detection delegated to context-sync where available.
4. **Phase 4: Verify** — CITATION.cff syntax, version triple consistency, lint and test pass, secret scan, unintended-file inclusion check.
5. **Phase 5: Release execution** — stage specific files only (no wildcards), HEREDOC commit message, tag creation, push main and tag, create the Git host release object that triggers the archive webhook.
6. **Post-release** — wait for the archive's DOI minting (typically five to ten minutes for Zenodo), verify the **concept DOI** is what gets propagated (not the initial version DOI; the off-by-one drift incident this guards against is documented in ADR-0001 of the upstream research line), then update the CITATION.cff and `.zenodo.json` with the concept DOI and create a follow-up `chore(release):` commit.

## Key concept: concept DOI vs version DOI

The skill's most load-bearing single discipline is the distinction between the **concept DOI** (parent record, resolves to the latest version, the canonical reference shape) and the **version DOI** (specific to one release, used only for reproducibility citations). At initial deposit the two are typically issued as adjacent integers, and a user copying the citation snippet from the just-published version's UI page usually copies the *version* DOI by mistake. The skill's post-release phase explicitly asks the operator to verify which DOI is being propagated and to back out and correct if the version DOI was used.

This is not a hypothetical concern: the upstream research line's [ADR-0001](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0001-concept-doi-canonical.md) was extracted from an actual sixteen-file drift incident across the author's own sibling repositories. The skill exists in part to prevent that incident from recurring.

## What this skill does NOT do

| Concern | Use this instead |
|---|---|
| Decide whether to apply the identifier-federation triplet at all | [authorship-strategy-skill](https://github.com/shimo4228/authorship-strategy-skill) — applies the framework's trigger conditions |
| llms.txt / llms-full.txt prose | [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) |
| JSON-LD knowledge graph design | [jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph) |
| Cross-document drift detection (Phase 3 delegates to this) | [context-sync](https://github.com/shimo4228/context-sync) |
| File-level architecture map regeneration (Phase 2 delegates to this) | A `update-codemaps` skill or comparable, where available |

## Related research and skills

- **Doctrine repository**: [authorship-strategy](https://github.com/shimo4228/authorship-strategy) — the normative framework, five tactical ADRs (especially the identifier-federation triplet 0001-0003), and empirical baseline this skill is the operational instantiation of
- **Peer components** (other component skills of the same framework):
  - [authorship-strategy-skill](https://github.com/shimo4228/authorship-strategy-skill) — the framework's judgment-checklist form; this release-time skill assumes the framework has already determined the artifact is in-scope
  - [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) — operationalizes Layer 4 tactic 7's Answer.AI `llms.txt` convention; this release skill's Phase 3 invokes it when llms.txt regeneration is needed
  - [jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph) — operationalizes Layer 4 tactic 7's JSON-LD knowledge graph; this release skill's Phase 4 uses its verification commands
- **Sibling research lines** (at the research-program level): [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), [Contemplative Agent](https://github.com/shimo4228/contemplative-agent), [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice)

> **Terminology note.** This ecosystem reserves *sibling* for research-line-level peers; at the component-skill level the term *peer component* is used instead.

## License

MIT. See [LICENSE](LICENSE).
