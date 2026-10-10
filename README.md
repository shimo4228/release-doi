# release-doi

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/release-doi)

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) that runs the **release workflow for DOI-registered research repositories**: GitHub repositories whose every release Zenodo archives under a new version DOI. It sequences pre-release verification, cross-document updates, the tag and GitHub release that trigger the deposit, and DOI propagation as one runbook (a pre-flight gate, four phases and a post-release phase), so that a release does not leave an old version number, the wrong DOI or an outdated cross-reference behind in its files.

The skill pushes to GitHub, creates the release that starts Zenodo's deposit, sends archive requests to Software Heritage and the Wayback Machine, and updates the Hugging Face mirror of repos that carry a `graph.jsonld` knowledge graph; with a Zenodo API token it also writes to the Zenodo API to add a new record to a community or to edit and re-publish a published record. It pushes and creates the release only when you ask it to; creating the release cannot be undone, because Zenodo mints a DOI for it, and the recovery path for a missed Zenodo opt-in deletes the GitHub release and its remote tag before recreating them.

## Key concept: concept DOI vs version DOI

The skill's most load-bearing single discipline is the distinction between the **concept DOI** (parent record, resolves to the latest version, the canonical reference shape) and the **version DOI** (specific to one release). At initial deposit the two are often adjacent numbers, so the version DOI is easy to take for the concept DOI; the incident behind the skill was initial version DOIs used as the canonical reference in sixteen files across two of the author's repositories. The skill separates the two by use: display links (the README DOI badge, the GitHub homepage field) carry the concept DOI and are set once; citation fields that must say which version was read (`CITATION.cff`, BibTeX, "How to cite") carry the version DOI and change every release.

## When to use

Apply the skill to a release when **all** of the following hold:

- The repository is registered (or about to be registered) with a versioned DOI archive (Zenodo is the reference example; the skill checks for a `CITATION.cff`)
- The release follows a tagged-release-triggers-deposit model (typical for Zenodo's GitHub integration)
- The release author intends to maintain the **concept DOI as canonical reference** discipline above ([ADR-0001](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0001-concept-doi-canonical.md), the design decision record in authorship-strategy that came out of that incident)

The skill adds most when the repository is one of several DOI-registered siblings that declare each other in the `related_identifiers` list of `.zenodo.json` or keep dataset mirrors on other platforms.

Besides releases, the skill covers fixing the metadata of an already published Zenodo record without cutting a new release, for a fix that cannot wait for the next one; that path makes no tag or release, so the tagged-release condition does not apply to it. To start it, ask `/release-doi` to edit and re-publish the published record's metadata; it runs only in an interactive session.

Skip the skill for:

- Repositories without a `CITATION.cff`, the file the skill checks to recognize a Zenodo-linked repository (a new repository's first release is in scope once it has one)
- One-shot artifacts published without versioning intent (the version/concept DOI distinction does not apply)
- Changes too small to release, such as a typo fix

## Install

### Claude Code

```bash
git clone https://github.com/shimo4228/release-doi
cp -r release-doi/skills/release-doi ~/.claude/skills/release-doi
```

To start a release, run `/release-doi` in the repository you are releasing.

Requirements: `git`, an authenticated GitHub CLI (`gh`), `curl`, `python3` and `uv` (`uvx cffconvert --validate` checks CITATION.cff against its schema), and a Zenodo account with the GitHub integration switched on for the repository. The skill itself has no runtime dependencies. A free Zenodo API token is needed only for Zenodo community inclusion and published-record edits. The skill text is written in Japanese.

The runbook was written for the author's own repositories and harness. Substitute or skip these steps:

- Zenodo community inclusion (first release only) names the author's community (`shimo4228-research-program`) and reads the token from a local credentials file (the runbook's example is `~/.config/zenodo/credentials.env`).
- The Hugging Face mirror step, only for repos with `graph.jsonld`, calls a separate `hf-sync` skill (not included here) with a logged-in `hf` CLI.
- The cross-document phase (Phase 2) opens with the [`/context-sync`](https://github.com/shimo4228/context-sync) skill for drift detection.
- The commit step passes the message through a file to work around the author's PreToolUse hook (`hooks/validate-bash.sh`), and a few lines point to the author's rules and skills (`rules/common/debugging.md`, `measurement-discipline`, `repair-discipline`).

### Other harnesses

Adapt the install path to your harness's skill convention; besides the steps above, the trigger and invocation mechanism is the harness-specific part. The commands assume Git, GitHub and Zenodo.

## How it works

The skill walks through a pre-flight gate, four sequential phases, and one post-release phase. The runbook names explicit stop conditions: the Zenodo opt-in is missing, there are no commits since the last tag, a Phase 3 check fails, Phase 4 finds unintended modified files, or after release the webhook delivery fails or no DOI is minted. At each, the skill stops and reports instead of advancing.

- **Pre-flight: Zenodo opt-in check**: verifies the Zenodo GitHub webhook is registered on the target repository (`gh api repos/<owner>/<repo>/hooks`). Zenodo's GitHub integration is **repo-by-repo opt-in**; releases cut before the toggle is enabled are not retroactively archived. This gate catches the gap on first release of a new DOI repository, a blind spot when sibling repos are already opted in.
- **Phase 1: Baseline**: capture the pre-release ground truth from command output (commits since the last tag, file and test counts where applicable, the version in `pyproject.toml`, `CITATION.cff` and the latest tags). Numbers already written in the docs are not trusted.
- **Phase 2: Cross-document consistency**: sync CHANGELOG / CITATION.cff / `codemeta.json` / `.zenodo.json` (including external papers newly cited since the last release) / `pyproject.toml` (where applicable) / multilingual README / llms.txt / glossary / ADR bidirectional links. Drift detection is delegated to context-sync.
- **Phase 3: Verify**: CITATION.cff schema and syntax, version consistency across `pyproject.toml`, `CITATION.cff` and the README BibTeX, lint and test pass, secret scan, unintended-file inclusion check.
- **Phase 4: Release execution**: stage specific files only (no wildcards), commit, tag, push main and tag, and create the GitHub release object that triggers the Zenodo webhook. Then request a Software Heritage archive and a Wayback Machine snapshot of the repository page, and sync the Hugging Face mirror for repositories that carry `graph.jsonld`.
- **Post-release**: wait for Zenodo to mint the new version DOI (usually within minutes), write it into the citation fields (`CITATION.cff`, the README BibTeX and "How to cite"), record the Software Heritage snapshot ID (SWHID) in `CITATION.cff`, and leave the concept-DOI badge and homepage untouched.

## What this skill does NOT do

| Concern | Use this instead |
|---|---|
| Weigh whether a plan fits your authorship strategy at all | [authorship-strategy-skill](https://github.com/shimo4228/authorship-strategy-skill) |
| llms.txt / llms-full.txt prose | [llms-txt-writer](https://github.com/shimo4228/llms-txt-writer) |
| Cross-document drift detection (Phase 2 delegates to this) | [context-sync](https://github.com/shimo4228/context-sync) |

## More from the author

- **[Authorship Strategy](https://github.com/shimo4228/authorship-strategy)**: the doctrine behind these skills, with the thesis, the dated design decisions (ADRs) and the preliminary measurements; concept DOI 10.5281/zenodo.20263316.
- **[citation-sync](https://github.com/shimo4228/citation-sync)**: audits whether the external papers a research repo cites appear in all three citation layers (docs, `.zenodo.json`, `graph.jsonld`) and syncs them bottom-up.
- **[jsonld-knowledge-graph](https://github.com/shimo4228/jsonld-knowledge-graph)**: designs and ships the `graph.jsonld` knowledge graph a research repo carries next to `llms.txt`, the file this runbook mirrors to Hugging Face.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with Authorship Strategy next to the author's other long-running projects and their DOIs.

## License

MIT. See [LICENSE](LICENSE).

<details>
<summary>For tools and AI assistants</summary>

release-doi is a Claude Code skill (Agent Skills format, Markdown only) that runs the release of a DOI-registered research repository, a GitHub repository whose releases Zenodo archives and assigns version DOIs, for authors who maintain such repositories and want each release to leave every identifier and cross-reference consistent. It also covers editing the metadata of an already published Zenodo record without a new version.

It exists because release-time drift is silent: a version DOI used where the concept DOI belongs still resolves, so nothing breaks while citations freeze at an old version. The workflow was extracted from releasing the author's own DOI-registered repositories (agent-knowledge-cycle, contemplative-agent, agent-attribution-practice and the hub), and its central safeguard comes from a sixteen-file drift incident found in May 2026, recorded as ADR-0001 of the authorship-strategy line.

Canonical facts: MIT license; one file, `skills/release-doi/SKILL.md`, written in Japanese, invocable as `/release-doi`; no runtime dependencies of its own; the commands use `git`, an authenticated `gh`, `curl`, `python3` and `uv` (`uvx cffconvert`); a Zenodo account with the GitHub integration enabled per repository is required, and a free Zenodo API token only for community inclusion and published-record edits; no paid key beyond a Claude Code plan. Data leaves the machine through `git push`, `gh release create` (which makes Zenodo archive the repository and mint a DOI, an irreversible step), Software Heritage's Save Code Now API, the Wayback Machine save endpoint, (for repositories with `graph.jsonld`) an upload to the Hugging Face dataset mirror, and, with a Zenodo API token, Zenodo REST API writes that request and accept community inclusion of a new record and edit and re-publish an already published record; the skill runs push and release only on the user's explicit request, and its recovery path for a missed Zenodo opt-in deletes the GitHub release and its tag before recreating them. Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits); no tagged release yet; [CHANGELOG.md](CHANGELOG.md) collects changes under Unreleased.

Example: in a repository with `CITATION.cff`, `/release-doi` first runs `gh api repos/<owner>/<repo>/hooks --jq '[.[] | select(.config.url | contains("zenodo"))] | length'`; 0 means the Zenodo opt-in is missing and the skill stops and asks the user to switch it on, while 1 or more lets it continue to the baseline. After `gh release create`, it waits for the new version DOI, writes it into `CITATION.cff` and the README citation, and leaves the concept-DOI badge and the GitHub homepage field as they are.

Links: [SKILL.md](skills/release-doi/SKILL.md) is the runbook; [inspiration.md](inspiration.md) records the motivating incident; [llms.txt](llms.txt) and [llms-full.txt](llms-full.txt) are the machine-readable summary and reference. As of 2026-10-10, the Unreleased section of CHANGELOG.md and the "Lineage to existing skills" section of inspiration.md still describe an earlier five-phase layout with a CODEMAPS regeneration phase and an optional Wikidata step; the current runbook has no CODEMAPS or Wikidata step and rules out self-registration in community-governed records such as Wikidata (authorship-strategy ADR-0021). The design decisions it carries out are ADRs 0001-0003, the identifier-federation triplet (link to the concept DOI, declare related works in `.zenodo.json`, keep dataset mirrors on other platforms in step), and ADR-0013 (the Software Heritage step) of [authorship-strategy](https://github.com/shimo4228/authorship-strategy), concept DOI [10.5281/zenodo.20263316](https://doi.org/10.5281/zenodo.20263316); cite the framework by that DOI.

</details>
