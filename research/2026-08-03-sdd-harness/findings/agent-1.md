# Agent 1 — Arc42 and Google's Open Knowledge Format (OKF)

## Key findings

- **arc42** is a free, open-source template (since 2005) for documenting software architecture in 12 sections; all sections are explicitly optional/tailorable, and the project itself publishes a "canvas" family (Architecture Communication Canvas, Architecture Inception Canvas, Tech Stack Canvas) as a deliberately lean, single-page alternative to the full template — [arc42 Overview](https://arc42.org/overview/), [Software Architecture Canvas](https://canvas.arc42.org/), [FAQ E-3](https://faq.arc42.org/questions/E-3/).
- **"OKF" is real** — the user is not misremembering. Google Cloud announced the **Open Knowledge Format (OKF)** on **June 12, 2026**: a vendor-neutral spec for packaging AI-agent context as a directory of markdown files with YAML frontmatter — [Google Cloud blog](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing), [GitHub spec (v0.2)](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).
- OKF's only mandatory field is `type`; everything else (title, description, resource, tags, provenance, trust, lifecycle) is optional — deliberately minimal, "just markdown, just files, just YAML frontmatter" — [SPEC.md](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).
- Independent practitioner critique: OKF standardizes the *container*, not the *semantics* — `type` values aren't registered, so two fully-conformant OKF bundles can use completely different vocabularies for the same concept. "A shared way to store context, not yet a shared way to make sense of it." — [Marc Bara, Medium](https://medium.com/@marc.bara.iniesta/googles-new-format-for-agent-context-a-standard-or-just-a-folder-82fb21d92041).

## Detail

### (a) arc42

**What it is.** arc42 (arc42.org) is a template/methodology for documenting software architecture, created by Dr. Gernot Starke and Dr. Peter Hruschka, in active use since 2005, free and open-source, with translations in many languages and tooling (AsciiDoc/Markdown templates, a docToolchain integration, and — as of 2026 — an "arc42 toolkit" of Claude Code/Copilot/Cursor skill prompts for LLM-assisted generation). [Overview](https://arc42.org/overview/), [arc42-toolkit GitHub](https://github.com/MSiccDev/arc42-toolkit).

**The 12 sections** (grouped thematically):
1. Introduction & Goals — requirements, quality goals, stakeholders
2. Constraints — regulatory/technical/organizational limits
3. Context & Scope — external systems/interfaces (business + technical context)
4. Solution Strategy — key approach decisions
5. Building Block View — static decomposition/modularization (usually the largest section)
6. Runtime View — key runtime/behavioral scenarios
7. Deployment View — infrastructure/deployment topology
8. Crosscutting Concepts — cross-cutting patterns, principles, technologies
9. Architectural Decisions — decisions not covered elsewhere (this is arc42's built-in ADR slot)
10. Quality Requirements — quality tree + concrete quality scenarios
11. Risks & Technical Debt
12. Glossary
[arc42 Overview](https://arc42.org/overview/), corroborated by [DeepWiki structure page](https://deepwiki.com/arc42/arc42-template/3-template-structure-and-content).

**Design philosophy / weight.** arc42's own materials explicitly frame it as anti-bloat: "describe only what stakeholders really need... record only the decisions you had to make anyway," used "on demand" so you supply only what's needed per project, explicitly compatible with agile/lean workflows ([FAQ E-1](https://faq.arc42.org/questions/E-1/), [FAQ E-3](https://faq.arc42.org/questions/E-3/)). All 12 sections are optional/tailorable by design; teams routinely skip sections that don't apply.

**Lean subset exists.** Yes — arc42 has an official lightweight variant family, the **Software Architecture Canvas** (formerly "Architecture Communication Canvas"), pitched as "as lean as architecture documentation can ever get" and literally a single page, now folded into the arc42 project itself. There are three canvas variants: Architecture Inception Canvas (greenfield), Architecture Communication Canvas (the "zip version" of full arc42, for communicating an existing architecture), and Tech Stack Canvas. [canvas.arc42.org](https://canvas.arc42.org/), [INNOQ blog on the ACC](https://www.innoq.com/en/blog/2023/07/architecture-communication-canvas/), [workingsoftware.dev announcement](https://www.workingsoftware.dev/the-software-architecture-canvas-is-now-part-of-arc42/).

**Practitioner assessment.** Consistent theme across FAQ and third-party blog posts: arc42 gives structure "without forcing you into a heavyweight process," and the acknowledged limitation is the opposite direction — for safety-/life-critical systems arc42's default pragmatism may be *insufficient* rigor, not excessive ([FAQ](https://faq.arc42.org/)). No credible source found calling full arc42 "too heavyweight" outright; criticism instead centers on teams misusing it by filling in all 12 sections exhaustively regardless of project size rather than tailoring it (implied across multiple practitioner posts, e.g. [hompus.nl series](https://blog.hompus.nl/2026/02/01/arc42-practical-series/)).

**Verdict for this harness.** Full 12-section arc42 is generic and enterprise-oriented — reasonable as a one-time, whole-project architecture doc, but too heavy and too broad-scope to reload per-feature inside a ~15k-token per-task budget; several sections (Constraints, Deployment View, Glossary, full Quality Tree) are largely static across features and shouldn't be re-derived or re-loaded per task. The **Architecture Communication Canvas / single-page canvas** is a much better structural match for a context-budgeted harness — it's explicitly designed to be the "zip version," fits comfortably in a small token budget, and arc42 section 9 (Architectural Decisions) is effectively a built-in, lighter-weight ADR slot rather than needing arc42's full machinery. Recommendation: use arc42's section *headings* as a checklist/vocabulary for what a good feature-level architecture note should cover, but adopt the canvas (or an even terser bespoke Markdown convention) as the actual per-feature artifact, not the full template.

### (b) "OKF" — Google's Open Knowledge Format

**Confirmed real, not a misremembering.** Google Cloud announced OKF on **June 12, 2026** (per multiple sources including [MarkTechPost, June 16 2026](https://www.marktechpost.com/2026/06/16/google-cloud-introduces-open-knowledge-format-okf-a-vendor-neutral-markdown-spec-for-giving-ai-agents-curated-context/)). It is genuinely called "Open Knowledge Format" with the acronym OKF — this is distinct from the older, unrelated "Open Knowledge Foundation" (OKFN, a UK nonprofit for open data, a different org entirely — worth flagging so it isn't conflated). Primary sources: [Google Cloud Blog announcement](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing) and the [official spec repo, GoogleCloudPlatform/knowledge-catalog, okf/SPEC.md, v0.2](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).

**What it actually is.** OKF formalizes the "LLM-wiki pattern": knowledge is packaged as a directory tree of `.md` files, each with YAML frontmatter, cross-linked via bundle-relative (`/path`) or relative markdown links. It's explicitly "just markdown, just files, just YAML frontmatter" — no schema registry, no central authority, no required tooling. Distributable as a git repo, tarball, or subdirectory.

**Structural details (v0.2 spec):**
- Only mandatory frontmatter field across all documents: `type`. A document with just `type` is "fully conformant."
- Recommended optional fields: `title`, `description`, `resource` (URI to underlying asset), `tags`.
- Two reserved filenames with special meaning: `index.md` (directory listing / progressive disclosure) and `log.md` (chronological change history). Everything else is a "concept document."
- Optional extension families: **Provenance** (`sources`: author, usage_count, last_modified), **Trust** (`generated`, `verified`), **Lifecycle** (`status`: draft/stable/deprecated; `stale_after` date).
- Actor identity convention: agents as `<producer>/<version>` (e.g. `reference_agent/gemini-2.5-pro`), humans as `human:<id>`, processes as `process:<id>`.
- Versioning: `<major>.<minor>`, declared via `okf_version` in the bundle-root `index.md`; consumers are told to attempt best-effort parsing across versions rather than reject.
- **Conformance is deliberately permissive**: consumers must not reject documents for missing optional fields, unrecognized `type` values, broken links, or missing indexes — "graceful degradation over strict validation."
- No explicit size/token/context-window limits in the spec itself; guidance leans toward "structural markdown over freeform prose" (headings, lists, tables, fenced code) for machine-parseability, but this is a style recommendation, not an enforced constraint.

**Practitioner/critical assessment.** The most substantive independent critique found ([Marc Bara, Medium, June 2026](https://medium.com/@marc.bara.iniesta/googles-new-format-for-agent-context-a-standard-or-just-a-folder-82fb21d92041)) argues OKF only standardizes the *container*, not *meaning*: since `type` values are free text and unregistered, two fully-conformant OKF bundles can describe the same concept ("BigQuery Table" vs. "table") incompatibly, so semantic interoperability across producers is unsolved. He also notes an implementation-reality gap — despite "vendor-neutral" framing, the reference implementation, demo dataset (BigQuery), and obvious ingestion path (Google's own Knowledge Catalog) all tilt toward the Google ecosystem — and that even the "required surface" isn't fully settled (Google's own reference parser requires four fields while the spec mandates only one). Other coverage (MarkTechPost, GitBook blog, various AI-blog summaries) is largely descriptive/promotional rather than critical and should be weighted lower.

**Verdict for this harness.** OKF's core idea — plain markdown + YAML frontmatter, git-native, human- and agent-readable, no proprietary tooling — is directionally *identical* to what a lean SDD harness should already be doing, and its permissive conformance model (only `type` required, graceful degradation) is compatible with a token-budgeted approach. However, adopting OKF *as a spec* adds machinery this harness likely doesn't need: reserved filenames (`index.md`/`log.md` semantics), the trust/provenance/lifecycle field families, actor-identity conventions, and version negotiation (`okf_version`) are all designed for a multi-producer, cross-organization knowledge-exchange problem (many teams' wikis feeding many different agents) — not the harness's actual problem (one team's specs feeding one Claude Code agent). Loading OKF's conventions costs a non-zero slice of the ~15k-token per-task budget for provenance/versioning fields the harness will rarely populate or need, and the semantic-fragmentation critique above means OKF alone doesn't even solve cross-file consistency by itself. Recommendation: borrow the *pattern* (markdown + minimal YAML frontmatter with a `type`-like field, index files for progressive disclosure) as a lightweight convention, but do not adopt OKF wholesale — a bespoke, harness-specific frontmatter schema with only the 2-3 fields actually needed (e.g., `type`, `status`, `related`) will be lighter and more predictable than importing OKF's full optional-field surface.

## Source quality notes

- arc42.org, docs.arc42.org, canvas.arc42.org, and faq.arc42.org are all primary/official sources maintained by the arc42 project — highly reliable.
- The Google Cloud Blog post and the GoogleCloudPlatform/knowledge-catalog GitHub repo (SPEC.md) are primary sources directly from the format's creator — highly reliable, and the spec file is versioned (v0.2) so it may continue to evolve; check for later versions if this research is revisited far in the future.
- MarkTechPost and GitBook Blog pieces are secondary but reputable, dated, and consistent with the primary sources — used only for corroboration/dating, not as sole sources of fact.
- Marc Bara's Medium post is an individual practitioner opinion piece (not peer-reviewed), but its technical claims are checkable against the spec and held up — treated as the most valuable critical source found, though it's a single voice, and the format itself is quite new (2026); no significant enterprise-adoption track record or independent post-mortems exist yet, since it's only about seven weeks old at time of research.
- Several other search hits (Suganthan, Flowtivity, MindStudio, witscode, Medium/@tahirbalarabe2) appeared to be SEO-style "explainer" blogs restating the Google Cloud blog with no new information — deliberately not relied upon beyond confirming existence/dates.
- No primary Google source explicitly discusses OKF in terms of LLM context-window/token economics; that framing is this agent's own analysis applied to the spec's content, not a claim from Google.

## Open questions

- Whether the harness's sibling standards (OpenSpec, spec-kit, unifiedprocess.ai — out of my boundary) already define their own frontmatter/ADR conventions that might make adopting either arc42-canvas or an OKF-like pattern redundant — worth the lead cross-checking against those agents' findings.
- OKF is only ~7 weeks old as of this research (announced June 12, 2026); no long-term adoption or criticism track record exists yet. If this research is revisited later, check for a v0.3+ spec or expanded independent critique.
- Confirmed but worth flagging explicitly to the user: "OKF" is *also* a long-standing acronym for the unrelated **Open Knowledge Foundation** (open-data nonprofit) — not investigated further here since it's outside scope, but the lead/report should disambiguate to avoid confusing the two in the final write-up.

## Ranked sources

1. https://arc42.org/overview/ — official arc42 project page, primary source for the template and its 12 sections.
2. https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing — official Google Cloud announcement of OKF, primary source.
3. https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md — the actual OKF v0.2 technical specification, primary and most authoritative source on structure.
4. https://canvas.arc42.org/ — official arc42 Software Architecture Canvas site, primary source for the lean/single-page variant.
5. https://faq.arc42.org/questions/E-3/ and https://faq.arc42.org/questions/E-1/ — official arc42 FAQ addressing lean/agile fit directly.
6. https://medium.com/@marc.bara.iniesta/googles-new-format-for-agent-context-a-standard-or-just-a-folder-82fb21d92041 — independent practitioner critique of OKF, technically substantiated against the spec.
7. https://www.marktechpost.com/2026/06/16/google-cloud-introduces-open-knowledge-format-okf-a-vendor-neutral-markdown-spec-for-giving-ai-agents-curated-context/ — reputable secondary tech-news coverage, useful for corroborating the announcement date.
8. https://www.workingsoftware.dev/the-software-architecture-canvas-is-now-part-of-arc42/ and https://www.innoq.com/en/blog/2023/07/architecture-communication-canvas/ — reputable practitioner/consultancy blogs corroborating the canvas family's role as arc42's lean variant.

<!-- STATUS: COMPLETE -->
