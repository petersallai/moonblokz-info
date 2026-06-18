# moonblokz-info Guidelines

## Purpose

The purpose of the `moonblokz-info` directory is to build and maintain a knowledge base for the MoonBlokz project.

This knowledge base is intended to support AI-assisted planning, development, documentation, and testing of new features. It should also help identify inconsistencies and underspecified areas across the project.

## Language Policy

All documentation under the `moonblokz-info` directory must be written in English, even if the prompt or source input used to generate it was provided in Hungarian or any other language.

## Source Fidelity

The role of the knowledge base is to organize, structure, and clarify information that is present in the available source materials. It must not invent new facts, assumptions, requirements, or conclusions that are not supported by those sources.

If a detail is missing from the source materials, it must not be introduced into the knowledge base as if it were established information. Missing, unclear, or uncertain points should instead be identified explicitly and handled according to the clarification rules in this document.

If the knowledge base and the current source materials diverge, the discrepancy must be called out explicitly and must not be ignored silently.

## Scope and Compactness

The knowledge base may become part of any MoonBlokz-related context, so its size must remain as small as possible while preserving the necessary information content.

Only actual knowledge-base content should be stored in its Markdown files. Material that does not belong to the knowledge itself, such as review notes, must not be written into the knowledge-base files.

Do not add low-information meta sections such as `Technical Writer View`, “how to read this file”, or similar document-usage boilerplate when they do not introduce real MoonBlokz knowledge. Short non-generic summaries of a file’s role are allowed, but they must stay compact and must not restate what is already obvious from the title, structure, or nearby sections.

## Modification Rules

If any inconsistency, illogical addition, or non-implementable solution is identified during any modification, the issue must be flagged immediately and clarified with the user. The problem must be identified explicitly, and the resolution decision must be left to the user.

## Proactive Conformance Cleanup

When a modification touches a document that also carries a pre-existing governance violation — such as discouraged boilerplate (for example a `Technical Writer View` or "how to read this file" section), a duplicate link list, or a `Related Documents` section that does not follow the required link-with-reason style — that violation should be brought into conformance as part of the same change, and the cleanup must be flagged explicitly in the response.

This latitude covers conformance and formatting only. It must not silently expand content: adding new links, facts, requirements, or conclusions remains governed by the Source Fidelity rule, and substantive issues that need a judgment call follow the Modification Rules above. Larger structural changes should be surfaced as optional follow-ups rather than applied unprompted.

## BMAD Processing Flow

Every piece of information added to the knowledge base must be processed sequentially through three BMAD agent perspectives.

1. **Business Analyst**: First, review the new information from the perspective of a business analyst and update the knowledge base accordingly.
2. **Architect**: Then, review the same information again from the perspective of an architect and further extend or revise the knowledge base based on that analysis.
3. **Technical Writer**: Finally, review the same information again from the perspective of a technical writer and refine or extend the knowledge base accordingly. As part of this step, also reconsider the internal structure of the affected knowledge-base files and adjust that structure when necessary.

## Information Placement and Redundancy

The knowledge base must avoid unnecessary redundancy across its files.

Each piece of information should appear in full detail only in the single file where it is most relevant. If the same information is also important elsewhere, the other files should reference the primary source instead of duplicating the content.

This applies especially to cross-cutting facts: single values or rules that several subsystem documents depend on, such as shared numeric constants, capacity caps, memory or flash budgets, physical or airtime limits, and FR/NFR numbers. Each cross-cutting fact has exactly one authoritative home and is stated in full only there:

- functional and non-functional requirements belong to the owning Product Requirements Document;
- data-structure layouts, RAM and stack budgets, const-generic values, public API shapes, and crate boundaries belong to the owning Architecture Decision Document.

A consolidation document, such as the System Constraints & Limits Reference, may gather cross-cutting facts in one place for navigation and may restate a value with a one-line implication and a link to its authoritative home. Such a document is explicitly non-authoritative and must never become a second source of truth; on any divergence the linked source wins.

When a cross-cutting fact changes, update its authoritative home and reconcile every restatement and link to it in the same change. A stale duplicate left elsewhere is a knowledge-base inconsistency and must be handled under the Source Fidelity and Post-Change Validation rules.

## FR Reference Namespacing

Functional-requirement (FR) numbers are assigned per Product Requirements Document, so the same number denotes unrelated requirements in different subsystems (for example, blockchain FR9 and storage FR9 are different requirements). Within a subsystem's own documents an FR may be cited by its bare number, because the document's subsystem already disambiguates it.

A citation that crosses subsystem boundaries — most often from a cross-cutting document such as the System Constraints & Limits Reference — must namespace the number with the owning subsystem's prefix (`BC-FR…` for blockchain, `ST-FR…` for storage) and link to the owning Product Requirements Document.

## Term Glossary

Terms that are used across subsystems or that carry more than one meaning (for example, "score", which names a radio relay metric, a radio-side vote-target input, and a blockchain accumulated-vote value) must have an entry in the cross-subsystem glossary that disambiguates the senses and links to each authoritative definition site. The glossary is non-authoritative: the linked definition site wins on any divergence.

When adding to or modifying the knowledge base, review the glossary as part of the same change:

- if the change introduces or newly surfaces a term that meets the inclusion criteria above, add a disambiguating entry that links to its authoritative definition site;
- if the change affects a term already in the glossary — a new sense to disambiguate, a changed meaning, or a moved or renamed authoritative definition site — update that entry in the same change.

This review only adds or revises entries that satisfy those criteria; a term that remains unambiguous within a single subsystem still does not belong in the glossary.

## Related Documents Sections

Each knowledge-base document should help readers reach the other documents most useful alongside it. When creating a new document, include a short "Related Documents" section that links those documents — typically the other concept, algorithm, and implementation files for the same subsystem, the owning Product Requirements Document and Architecture Decision Document, and any relevant cross-cutting reference such as the System Constraints & Limits Reference or the Glossary. When modifying a document that lacks such a section, add one if its related set is clear. Keep these sections to links with a short reason each, not restated content.

## Knowledge Base Entry Point

The primary entry point to this knowledge base is [`moonblokz-index.md`](./moonblokz-index.md).

Whenever this `AGENTS.md` file is active, every request and task should be treated as related to the MoonBlokz system by default, and the knowledge base should be used accordingly.

## Index Maintenance

The file [`moonblokz-index.md`](./moonblokz-index.md) serves as the knowledge-base table of contents for the `moonblokz-info` directory.

When creating a new knowledge-base file under `moonblokz-info`:

- a corresponding entry must be added to `moonblokz-index.md`,
- the entry must include a short summary that helps readers decide when to open the file,
- and the reading order or navigation guidance should be updated if the new file changes the recommended flow.

When modifying an existing knowledge-base file under `moonblokz-info`:

- `moonblokz-index.md` must be checked,
- and the short summary for that file must be updated if the scope, emphasis, or purpose of the document has changed.

## Post-Change Validation

After every knowledge base modification, a consistency, logical soundness, feasibility, redundancy, and source-fidelity review must be performed.

The review must be carried out, but its results must be returned in the response rather than stored in the knowledge-base Markdown files.
