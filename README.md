---
title: "Building and Operating an LLM Wiki"
subtitle: "A Source-Checked Practical Guide Based on Andrej Karpathy's LLM Wiki Pattern"
date: "2026-11-25"
status: "Reviewed draft"
language: "en"
primary_source: "https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"
---

# Building and Operating an LLM Wiki

> **Scope note:** This guide separates three kinds of content: **Karpathy's stated pattern**, **reasonable implications of that pattern**, and **additional implementation recommendations**. The latter are included to make the concept operational, but they should not be attributed to Andrej Karpathy unless explicitly identified as such.

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [What Karpathy Actually Proposes](#what-karpathy-actually-proposes)
3. [LLM Wiki Compared with Conventional RAG](#llm-wiki-compared-with-conventional-rag)
4. [Reference Architecture](#reference-architecture)
5. [Building the Wiki](#building-the-wiki)
6. [Operating Workflows](#operating-workflows)
7. [Indexing and Retrieval](#indexing-and-retrieval)
8. [Advantages](#advantages)
9. [Limitations, Risks, and Countermeasures](#limitations-risks-and-countermeasures)
10. [Comparison with Alternative Approaches](#comparison-with-alternative-approaches)
11. [Governance and Quality Assurance](#governance-and-quality-assurance)
12. [Security, Privacy, and Backup](#security-privacy-and-backup)
13. [Implementation Roadmap](#implementation-roadmap)
14. [Example Agent Instructions](#example-agent-instructions)
15. [When Not to Use an LLM Wiki](#when-not-to-use-an-llm-wiki)
16. [Conclusion](#conclusion)
17. [References](#references)

---

## Executive Summary

An **LLM Wiki** is a persistent, structured knowledge base maintained primarily by an LLM agent. Andrej Karpathy describes it as a different pattern from conventional retrieval-augmented generation (RAG): instead of reconstructing knowledge from raw document fragments for every question, the agent incrementally integrates new sources into a durable, interlinked collection of Markdown files.

Karpathy's pattern has three layers:

1. **Raw sources**, which are curated and immutable.
2. **The Wiki**, a collection of LLM-generated and LLM-maintained Markdown pages.
3. **The schema**, an instruction document that defines the Wiki's structure, conventions, and workflows.

The core value proposition is that the Wiki becomes a **persistent, compounding artifact**. Source summaries, entity pages, concept pages, comparisons, cross-references, and contradictions can be maintained over time rather than reconstructed in every session.

Karpathy presents the concept as deliberately abstract and modular. He does not prescribe a universal directory tree, metadata schema, approval model, security architecture, quality metrics, or production-ready implementation. Those elements in this guide are therefore clearly marked as **recommended extensions**, not claims about his original proposal.

A reliable implementation should preserve source provenance, keep raw documents unchanged, make generated synthesis distinguishable from source statements, use Git or equivalent version control, validate links and citations, and require human review for consequential changes. These controls are practical recommendations that address risks inherent in maintaining durable model-generated knowledge.

---

## What Karpathy Actually Proposes

### The central idea

Karpathy contrasts the LLM Wiki with a common RAG experience. In a typical RAG system, an LLM retrieves relevant chunks from source documents at query time and synthesises an answer. For a subtle question spanning several documents, the system must rediscover and reconnect relevant information each time.

His alternative is to let an LLM agent **incrementally build and maintain a persistent Wiki**. When a new source is added, the agent reads it, extracts important information, and integrates that information into the existing Wiki. It may add or revise entity pages, concept pages, topic summaries, links, and notes about contradictions.

The important distinction is not that retrieval disappears. Rather, the maintained Wiki becomes a reusable intermediate knowledge layer between the user and the raw source collection.

### Human and LLM roles in the original pattern

Karpathy characterises the human as responsible for:

- Choosing and curating sources
- Exploring the domain
- Asking good questions
- Directing what deserves emphasis
- Reading and checking the evolving Wiki

The LLM is responsible for much of the maintenance work:

- Summarising
- Cross-referencing
- Filing information
- Updating related pages
- Maintaining the index and log
- Identifying contradictions and gaps

He uses a programming analogy: **Obsidian is the IDE, the LLM is the programmer, and the Wiki is the codebase**.

### Three explicit layers

Karpathy explicitly describes:

- **Raw sources:** curated, immutable source documents that the LLM reads but does not modify.
- **The Wiki:** LLM-generated Markdown pages that the model creates and updates.
- **The schema:** an instruction file, such as `CLAUDE.md` or `AGENTS.md`, defining structure, conventions, and workflows.

![LLM Wiki three-layer architecture](llm-wiki-three-layer-architecture.png)

*Figure 1. Visual interpretation of Karpathy's three explicit layers, with human review added as an operational safeguard. Visual generated with Microsoft Copilot.*

### Explicit operations

Karpathy names three main operations:

1. **Ingest:** Process a new source, discuss key takeaways, write a summary, update relevant pages and the index, and append to the log.
2. **Query:** Ask questions against the Wiki, synthesise an answer with citations, and optionally file valuable answers back into the Wiki.
3. **Lint:** Periodically inspect the Wiki for contradictions, stale claims, orphan pages, missing concepts, missing links, and information gaps.

### Index and log

Karpathy distinguishes two special files:

- `index.md` is content-oriented and helps the human and the LLM discover relevant pages.
- `log.md` is chronological and records ingests, queries, lint passes, and other operations.

He suggests that an index can work at moderate scale, giving an approximate example of around 100 sources and hundreds of pages. This is an observation, not a guaranteed capacity limit or benchmark.

### Optional tooling

Karpathy describes the Wiki as plain Markdown and notes that it can be stored in a Git repository. He mentions:

- Obsidian for browsing and graph view
- Obsidian Web Clipper for capturing Web content
- Local image downloads for durable references
- Marp for Markdown-based presentations
- Dataview for queries over frontmatter
- `qmd` as an optional local hybrid search tool for Markdown

These tools are suggestions, not mandatory components.

---

## LLM Wiki Compared with Conventional RAG

![Conventional RAG compared with an LLM Wiki](rag-vs-llm-wiki.png)

*Figure 2. The conceptual difference between repeated query-time synthesis and a persistent maintained knowledge layer. Visual generated with Microsoft Copilot.*

| Characteristic | Conventional RAG | LLM Wiki pattern |
|---|---|---|
| Primary retrieval target | Usually chunks from source documents | Maintained Wiki pages, with raw sources still available |
| Synthesis | Usually performed for each question | Important synthesis is retained and updated |
| Persistence between questions | Usually limited | Central design goal |
| Cross-document relationships | Reconstructed at query time | Stored as links and maintained pages |
| Contradictions | May be surfaced during a query | Can be recorded persistently |
| Maintenance | Mostly indexing and source refresh | Page creation, integration, linking, linting, and refresh |
| Main failure risk | Retrieval misses or poor synthesis | Persistent error, drift, duplication, or stale synthesis |

The two patterns are not mutually exclusive. Karpathy explicitly allows search tooling as the Wiki grows. A practical system may use lexical or vector retrieval over both Wiki pages and raw sources.

---

## Reference Architecture

### Layer 1: Immutable raw sources

The raw layer contains original evidence, such as:

- Web pages saved as Markdown
- PDFs
- Research papers
- Meeting transcripts
- Notes
- Images
- Structured data

**Directly supported by Karpathy:** Raw sources should be immutable and serve as the source of truth.

**Recommended extension:** Record provenance metadata such as title, publisher, acquisition date, original URL, version, licence, classification, and a checksum for sensitive or authoritative documents.

### Layer 2: The maintained Wiki

The Wiki may contain:

- Source summaries
- Concept pages
- Entity pages
- Topic pages
- Comparisons
- Timelines
- Open questions
- Contradiction notes
- Durable analyses created from user queries

**Directly supported by Karpathy:** The LLM owns writing and maintenance of this layer, while the user reads and guides it.

**Recommended extension:** Do not treat “LLM ownership” as unrestricted autonomy in an enterprise or high-stakes environment. Use review gates for legal, security, safety, financial, or policy-relevant content.

### Layer 3: Schema and agent instructions

The schema should define:

- Directory structure
- Naming conventions
- Page types
- Linking conventions
- Citation rules
- Ingest, query, and lint workflows
- Rules for creating, merging, renaming, or deleting pages

Karpathy explicitly presents the schema as co-evolving with the user and the domain.

### Recommended directory structure

The following layout is an implementation recommendation, not a structure prescribed by Karpathy:

```text
llm-wiki/
├── AGENTS.md
├── README.md
├── index.md
├── log.md
├── raw/
│   ├── web/
│   ├── pdf/
│   ├── notes/
│   ├── transcripts/
│   └── assets/
├── wiki/
│   ├── concepts/
│   ├── entities/
│   ├── topics/
│   ├── comparisons/
│   ├── decisions/
│   ├── questions/
│   └── sources/
├── templates/
├── reports/
└── scripts/
```

For a long-running knowledge programme, separate domain Wikis are often easier to govern than one universal Wiki. For example, an application-security Wiki can use different templates and source hierarchies from an enterprise-integration Wiki.

### Recommended page metadata

```yaml
---
title: "OWASP SAMM"
type: "framework"
status: "reviewed"
created: "2026-11-25"
updated: "2026-11-25"
source_ids:
  - "src-owasp-samm-about"
source_count: 3
confidence: "high"
review_required: false
---
```

Karpathy mentions YAML frontmatter as useful with Dataview, but he does not prescribe these fields. Avoid pseudo-precision: a small vocabulary such as `high`, `medium`, `low`, and `disputed` is preferable to an uncalibrated numerical confidence score.

---

## Building the Wiki

### Define the purpose

Document:

- The domain and intended audience
- Questions the Wiki should answer
- Included and excluded source types
- Authoritative-source hierarchy
- Data classification and access rules
- Review thresholds

### Start with a pilot corpus

**Recommended extension:** Begin with 10 to 30 representative sources rather than bulk-ingesting thousands of files. Include overlapping sources, one disagreement, one long PDF, and one versioned or frequently updated source. Create 15 to 30 realistic test questions.

This recommendation is not from Karpathy's text. It is a risk-reduction method for testing schema quality, citation style, page granularity, and retrieval before scaling.

### Use stable source identifiers

**Recommended extension:** Assign deterministic IDs, for example:

```text
src-nist-ssdf-1-1
src-cisa-secure-by-design-2023
src-owasp-asvs-5-0-0
```

Stable IDs reduce broken references if a file is renamed.

### Design citations for verification

Karpathy says query answers should use citations, but he does not define a citation format. A practical implementation should point to the exact evidence where possible:

- PDF page and section
- Markdown heading
- Transcript timestamp
- Table, figure, requirement, or control identifier

```markdown
NIST groups SSDF practices into four practice groups.[^src-nist-ssdf-1-1-p8]

[^src-nist-ssdf-1-1-p8]: `raw/pdf/NIST.SP.800-218.pdf`, p. 8.
```

---

## Operating Workflows

### Ingest

The following controlled workflow elaborates Karpathy's ingest operation:

1. **Admit the source:** Preserve the original and record provenance.
2. **Analyse:** Extract concepts, entities, claims, dates, recommendations, and conflicts.
3. **Plan changes:** List pages to create or update before editing.
4. **Apply changes:** Update source summaries, topic pages, links, the index, and the log.
5. **Validate:** Check links, citations, structure, duplicates, and protected paths.
6. **Review and merge:** Inspect the diff and approve according to risk.

![LLM Wiki lifecycle](llm-wiki-lifecycle.png)

*Figure 3. A governed implementation of the ingest and maintenance lifecycle. Git, atomic changes, formal validation, and human approval are recommended controls added to Karpathy's abstract pattern. Visual generated with Microsoft Copilot.*

Karpathy states that one source may affect 10 to 15 Wiki pages in his practical experience. This should be treated as an example, not a target. Large cross-Wiki edits increase review difficulty and error-propagation risk.

### Query

1. Read the index or search the Wiki.
2. Open relevant pages.
3. Verify against raw sources for disputed, sensitive, or consequential questions.
4. Answer with citations.
5. Distinguish source statements from synthesis or recommendations.
6. File the result only if it has durable value.
7. Update the index and log when the result becomes a Wiki page.

Karpathy explicitly proposes filing useful comparisons, analyses, and discovered connections back into the Wiki so that they are not lost in chat history.

### Lint

Karpathy's lint operation includes finding:

- Contradictions
- Stale claims
- Orphan pages
- Important concepts without pages
- Missing cross-references
- Information gaps

A practical implementation should separate:

**Deterministic lint**

- Broken links
- Missing cited files
- Duplicate IDs
- Missing frontmatter
- Invalid paths
- Unexpected modification of raw sources
- Secret scanning

**Semantic lint**

- Contradictory claims
- Terminology drift
- Near-duplicate pages
- Weakly supported synthesis
- Newer sources that supersede older claims
- Pages that became too broad

Model-based lint findings should be presented as proposals because the model can misidentify both contradictions and duplicates.

### Logging

Karpathy proposes an append-only chronological `log.md`. A structured entry could be:

```markdown
## [2026-11-25] ingest | NIST SSDF 1.1
- Source: src-nist-ssdf-1-1
- Created: wiki/sources/nist-ssdf-1-1.md
- Updated: 4 pages
- Contradictions: 1
- Review: required
- Commit: abc1234
```

The exact fields above are a recommendation, not part of his prescribed format.

---

## Indexing and Retrieval

### Begin with the index

Karpathy recommends reading `index.md` first and then opening relevant pages. This simple method may be sufficient at moderate scale.

### Add plain-text search

```bash
rg -n "threat modeling|threat model" wiki/
```

Lexical search is effective for exact identifiers such as CVEs, CWE IDs, control numbers, product names, and quoted terms.

### Add hybrid retrieval when measured failures justify it

Karpathy mentions `qmd`, which combines local Markdown search, hybrid BM25/vector retrieval, and LLM reranking. A larger implementation may use:

- Full-text or BM25 search
- Vector search
- Metadata filters
- Reranking
- Retrieval over Wiki pages and raw sources

The Wiki pattern should not be described as “RAG-free.” It can operate without embedding-based infrastructure at moderate scale, but Karpathy explicitly treats additional search tooling as an option when the repository grows.

---

## Advantages

### Persistent, compounding knowledge

This is the principal advantage claimed by Karpathy. Useful synthesis, links, and contradictions survive beyond one conversation.

### Less repeated reconstruction

Previously integrated relationships can be reused. This may reduce repeated reasoning and improve continuity, although Karpathy does not publish a controlled cost or accuracy benchmark in the original idea file.

### Better human navigation

Markdown, links, indexes, and Obsidian graph view make the knowledge base directly inspectable by humans.

### Explicit evolution

A maintained page can record that sources disagree or that newer evidence supersedes an older claim.

### Portability

Plain Markdown is tool-independent and works with Obsidian, VS Code, GitHub, GitLab, and ordinary text editors.

### Version-controlled maintenance

Karpathy notes that the Wiki can be a Git repository, which enables history, branching, and collaboration. Whether this is “free” operationally depends on repository hosting, access management, and review processes.

---

## Limitations, Risks, and Countermeasures

These risks are mostly analytical consequences and implementation concerns. They should not be presented as statements made by Karpathy unless explicitly noted.

| Risk | Why it matters | Countermeasure |
|---|---|---|
| Persistent hallucination | A false generated claim can be reused repeatedly | Require precise citations and review material claims |
| Error propagation | One incorrect interpretation can update many pages | Cite raw evidence, limit change scope, review diffs |
| Staleness | A polished page may hide outdated evidence | Track versions, review dates, and supersession |
| Duplicate pages | Models can create overlapping concepts | “Extend before create,” duplicate detection, semantic lint |
| Structural entropy | Names, folders, and links may become inconsistent | Schema, templates, deterministic validation |
| Review overload | Automated editing can create large diffs | Atomic changes, risk tiers, maximum-change thresholds |
| Context and cost growth | Agents may reread excessive content | Index-first navigation, search, caching, smaller routing models |
| Model inconsistency | Different models may apply rules differently | Regression questions, templates, model/version logging |
| Confidentiality exposure | Sources may contain restricted information | Approved models, access control, classification, secret scanning |
| False completeness | A polished Wiki can conceal source gaps | Coverage maps, open questions, explicit uncertainty |

The claim that maintenance cost becomes “near zero” should be read as motivational language in Karpathy's explanation, not a measured guarantee. LLMs reduce repetitive bookkeeping but do not eliminate source curation, review, governance, model cost, or incident handling.

---

## Comparison with Alternative Approaches

| Dimension | LLM Wiki | Conventional RAG | Manually maintained Wiki | Notes folder |
|---|---|---|---|---|
| Core unit | Maintained knowledge page | Retrieved source chunk | Human-written page | Individual note |
| Accumulated synthesis | High by design | Usually limited between queries | High | Variable |
| Human effort | Curation and review | Source management and evaluation | Writing and maintenance | Low initially |
| Persistent model-error risk | High without controls | Usually limited to answers | Low model risk | Low model risk |
| Cross-source synthesis | Precompiled and updated | Reconstructed at query time | Human-created | Often ad hoc |
| Portability | High with Markdown | Platform-dependent | Medium to high | High |
| Scaling need | Search plus governance | Retrieval infrastructure | Contributor capacity | Increasing organisation effort |

A robust implementation is often hybrid: use the Wiki as the maintained synthesis layer, while retaining search or RAG over raw sources for verification and uncovered questions.

---

## Governance and Quality Assurance

Karpathy does not prescribe a formal governance model. The following is recommended for team or enterprise use.

### Review tiers

- **Tier 1, automated:** formatting, index updates, validated backlink repairs.
- **Tier 2, standard human review:** source summaries, concept-page additions, comparisons.
- **Tier 3, subject-matter review:** legal, regulatory, safety, security, financial, policy, or confidential claims.

### Source hierarchy

1. Official standards, legislation, regulation, and vendor documentation
2. Peer-reviewed research and authoritative institutions
3. Approved internal policies and architecture decisions
4. Reputable secondary analysis
5. Community discussions and informal notes
6. Model-generated hypotheses

### Useful quality measures

These are recommended measures, not Karpathy's metrics:

- Citation coverage for material factual claims
- Broken-link and missing-source count
- Orphan and near-duplicate page count
- Unresolved contradiction count
- Retrieval success on a fixed question set
- Unsupported-claim rate
- Human acceptance and correction rate
- Ingest cost, latency, and review time

Avoid claiming that arbitrary thresholds such as “90% retrieval accuracy” represent an industry standard. Establish baselines with a domain-specific evaluation set and adjust acceptance criteria to the risk of the use case.

---

## Security, Privacy, and Backup

Karpathy's idea file is not a security or compliance specification. A production implementation should add:

- Data classification before ingestion
- Approved LLM endpoints
- Least-privilege repository access
- Secret scanning
- Encryption of devices and backups
- Retention and deletion rules
- Provenance and licence records
- Audit logs for agent writes
- Approval for destructive actions

Karpathy notes that a Wiki can be stored in Git. For resilience, Git should be combined with an approved remote and an independent backup process. Obsidian Sync or OneDrive provides synchronisation, but synchronisation alone is not equivalent to a tested backup. Concurrent writes from a sync client and an agent can also create conflicts.

---

## Implementation Roadmap

### Phase 1: Foundation

- Define one domain and audience.
- Create the repository and directories.
- Write `AGENTS.md` and templates.
- Create `index.md` and `log.md`.
- Initialise Git.
- Add basic Markdown, link, and secret checks.

### Phase 2: Pilot

- Select 10 to 30 representative sources.
- Ingest one source at a time.
- Review every diff.
- Refine page types, metadata, and citations.
- Record recurring model failures in the schema.

### Phase 3: Evaluation

- Create 15 to 30 realistic questions.
- Compare Wiki answers with raw-source search or RAG.
- Measure retrieval, citation accuracy, completeness, latency, and cost.
- Review errors, not only successful demonstrations.

### Phase 4: Controlled operation

- Introduce review tiers.
- Run deterministic lint on every proposed change.
- Run semantic lint periodically.
- Track model, schema, and retrieval changes.
- Add hybrid search only when the index and lexical search no longer meet measured needs.

---

## Example Agent Instructions

```markdown
# LLM Wiki Maintainer Instructions

## Mission
Maintain a reliable, traceable, interlinked knowledge base for the defined domain.

## Non-negotiable rules
1. Never modify files under `raw/`.
2. Read `index.md` before creating a page.
3. Extend an existing page when it covers the same concept.
4. Cite raw sources for every material factual claim.
5. Distinguish source statements from model synthesis.
6. Preserve disagreements and cite each position.
7. Do not delete or rename pages without a migration plan.
8. Update backlinks, `index.md`, and `log.md` together.
9. Validate the repository before declaring completion.
10. Request human review for high-impact changes.

## Required ingest report
- Source metadata
- Summary
- Extracted claims and concepts
- Proposed file operations
- Contradictions and uncertainties
- Validation results
- Review recommendation
```

The exact instruction set is an implementation recommendation. Karpathy's claim is that the schema should communicate structure, conventions, and workflows to the agent and co-evolve with the user.

---

## When Not to Use an LLM Wiki

Prefer conventional search, RAG, or a simpler repository when:

- The corpus is static and questions are rare.
- Users mainly need exact document retrieval.
- Generated intermediate synthesis is not acceptable.
- No one can review or govern persistent model output.
- The domain changes faster than maintained synthesis can be refreshed.
- The sources cannot be processed by an approved LLM.
- An existing regulated records system already provides the required controls.
- The knowledge collection is small enough to maintain manually.

---

## Conclusion

Karpathy's LLM Wiki is best understood as a **knowledge-maintenance pattern**, not a complete software product or fixed architecture. Its defining ideas are immutable raw sources, an LLM-maintained Markdown Wiki, an evolving schema, and the ingest-query-lint operating cycle. Its strongest promise is accumulated synthesis: the system preserves useful structure and connections instead of rebuilding them for every question.

The practical challenge is that persistent synthesis also makes mistakes persistent. A serious implementation should therefore add precise citations, provenance, Git-based change control, deterministic validation, risk-based review, security controls, and domain-specific evaluation. Those measures extend Karpathy's concept; they should not be attributed to his original note.

For a pilot, use one domain, a small representative source set, direct human involvement, and explicit evaluation questions. Scale only after the Wiki demonstrates reliable retrieval, traceability, and manageable maintenance.

---

## References

1. Andrej Karpathy, **“LLM Wiki: A pattern for building personal knowledge bases using LLMs.”** GitHub Gist: <https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f>
2. Obsidian, **Obsidian Sync**: <https://obsidian.md/sync>
3. `qmd`, **local search for Markdown**: <https://github.com/tobi/qmd>
4. Vannevar Bush, **“As We May Think,”** *The Atlantic*, 1945. Karpathy presents the Memex as a related idea, not as a technical dependency of the LLM Wiki.

## Appendix: Accuracy Review Summary

The reviewed version corrects or qualifies the following types of statements from the earlier draft:

- It labels directory structures, metadata fields, source IDs, review tiers, security controls, metrics, and pilot sizes as **recommendations**, not as parts of Karpathy's specification.
- It avoids treating Karpathy's approximate scale example of around 100 sources and hundreds of pages as a formal capacity claim.
- It clarifies that an LLM Wiki can use search or RAG tooling and is not inherently “RAG-free.”
- It treats “one source may touch 10–15 pages” as an experiential example, not a target.
- It qualifies “maintenance cost is near zero” as motivational language rather than a measured guarantee.
- It distinguishes Karpathy's explicit three-layer architecture and ingest-query-lint cycle from enterprise governance added in this guide.
