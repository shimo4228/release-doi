# Inspiration / Origin

This file records the canonical implementation that originated the
`release-doi` skill. Kept separate from `SKILL.md` so the skill stays
portable to other authors who do not share this origin context.

## Origin

The release workflow encoded in this skill was extracted from operating
four DOI-registered sibling research repositories in the same author's
research program — [`agent-knowledge-cycle`](https://github.com/shimo4228/agent-knowledge-cycle)
(DOI 10.5281/zenodo.19200726), [`contemplative-agent`](https://github.com/shimo4228/contemplative-agent)
(DOI 10.5281/zenodo.19212118), [`agent-attribution-practice`](https://github.com/shimo4228/agent-attribution-practice)
(DOI 10.5281/zenodo.19652013), and the federation hub
[`shimo4228/shimo4228`](https://github.com/shimo4228/shimo4228). The
workflow consolidates the release-time disciplines those repositories
share, with explicit safeguards against the drift incidents the author
encountered while operating them.

## The motivating incident

The most load-bearing safeguard in the skill — the post-release phase's
explicit concept-DOI verification step — was extracted from a
sixteen-file drift incident discovered in May 2026. Initial version
DOIs had been used as canonical references across two sibling
repositories, propagating into ORCID records, citation files, knowledge
graphs, and AI-facing reference files. The drift had not been detected
by either ingestion-side correctness checks (every URL resolved
correctly) or external citation queries (the version DOI was being
cited correctly by downstream consumers, just citing the wrong
artifact). The recovery was mechanical (a search-tool query located
all sixteen occurrences for systematic correction), but the cost of
discovering the drift was high enough that the author wrote it into a
standing policy: every release must include explicit verification of
which DOI is being propagated.

The standing policy became [ADR-0001 of the upstream research line](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0001-concept-doi-canonical.md);
this skill operationalizes the ADR as the post-release verification
step.

## Canonical doctrine repository

The skill's release runbook implements the identifier-federation
triplet from the upstream research line:

- [ADR-0001 Concept DOI as Canonical Reference](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0001-concept-doi-canonical.md) → post-release phase's concept-DOI verification
- [ADR-0002 DOI Federation via .zenodo.json](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0002-doi-federation-via-zenodo-json.md) → Phase 3's `.zenodo.json` consistency check and sibling cross-reference update guidance
- [ADR-0003 Cross-Platform Dataset Federation](https://github.com/shimo4228/authorship-strategy/blob/main/docs/adr/0003-cross-platform-dataset-federation.md) → Phase 3's Hugging Face Dataset mirror update sub-step

The doctrine repository contains the *normative articulation* of the
framework these ADRs are part of:

> [`authorship-strategy`](https://github.com/shimo4228/authorship-strategy)

The doctrine repository is independently DOI-registerable as the
fourth sibling research line in the author's program. This skill
exists to make the framework's release-time discipline executable
without requiring the agent (or its operator) to re-derive the
runbook from the ADRs on every release.

## Lineage to existing skills

The skill explicitly delegates several phases to other ecosystem
skills where they are installed:

- Phase 2 (CODEMAPS regeneration) delegates to an `update-codemaps` skill or comparable where available
- Phase 3 (Cross-document consistency) delegates drift detection to [`context-sync`](https://github.com/shimo4228/context-sync)
- Phase 3 invokes [`llms-txt-writer`](https://github.com/shimo4228/llms-txt-writer) when llms.txt regeneration is needed
- Phase 4 (Verify) uses [`jsonld-knowledge-graph`](https://github.com/shimo4228/jsonld-knowledge-graph)'s verification commands for `graph.jsonld` validation

When the dependencies are absent, the skill degrades gracefully to
manual checks, but the release-quality bar is higher when all are
installed.

## Lineage to memory

The skill's specific operations were informed by six project memory
entries the author had accumulated by May 2026 — graph.jsonld
validation observations, multilingual README policy decisions, ORCID
enrichment discipline, Zenodo relatedIdentifiers federation, the
concept-versus-version DOI drift incident, and Hugging Face Datasets
cross-federation patterns — each of which was converted into a
corresponding ADR in the doctrine repository.

## When this skill becomes obsolete

The framework's Layer 3 explicitly predicts that scaffolds dissolve.
This skill is itself a scaffold: it loads a specific class of release
workflow into a specific class of harness, and the workflow itself
depends on a specific class of archive-deposit substrate (Zenodo's
tagged-release-triggers-webhook integration is the reference instance).
When DOI infrastructure or release tooling shifts substantially — for
example, when archive services consolidate around a different
auto-deposit mechanism, or when DOI minting becomes embedded in Git
hosting platforms directly — this skill should be retired in favor of
whatever workflow the new substrate supports. The framework's
identifier-federation triplet (the doctrine the skill instantiates)
is the artifact the framework's own Layer 3 commits to preserving
across substrate shifts.
