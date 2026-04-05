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

## BMAD Processing Flow

Every piece of information added to the knowledge base must be processed sequentially through three BMAD agent perspectives.

1. **Business Analyst**: First, review the new information from the perspective of a business analyst and update the knowledge base accordingly.
2. **Architect**: Then, review the same information again from the perspective of an architect and further extend or revise the knowledge base based on that analysis.
3. **Technical Writer**: Finally, review the same information again from the perspective of a technical writer and refine or extend the knowledge base accordingly. As part of this step, also reconsider the internal structure of the affected knowledge-base files and adjust that structure when necessary.

## Information Placement and Redundancy

The knowledge base must avoid unnecessary redundancy across its files.

Each piece of information should appear in full detail only in the single file where it is most relevant. If the same information is also important elsewhere, the other files should reference the primary source instead of duplicating the content.

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
