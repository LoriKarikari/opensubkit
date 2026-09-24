# Domain docs

How agents consume this repo's domain documentation.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root, the project glossary.
- **`.agents/adr/`**, the ADRs that touch the area you're about to work in.

If either doesn't exist yet, proceed silently. They're created when a term or decision actually gets resolved, not upfront.

## Layout

Single context.

```
/
├── AGENTS.md
├── CONTEXT.md
└── .agents/
    ├── docs/         agent config: issue tracker, triage labels, this file
    ├── adr/          architecture decision records
    └── research/     research findings, one file per research ticket
```

## Use the glossary's vocabulary

When your output names a domain concept (an issue title, a proposal, a test name), use the term as defined in `CONTEXT.md`. Don't drift to synonyms it lists under _Avoid_.

A concept missing from the glossary is a signal. Either you're inventing language the project doesn't use, or there's a real gap to resolve and add.

## Flag ADR conflicts

If your output contradicts an ADR, say so explicitly instead of silently overriding it.
