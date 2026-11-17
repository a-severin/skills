---
name: grill-adr
description: >
  Write the outcome of the current working session as an Architecture Decision
  Record in docs/adr, following the repository's existing ADR template,
  numbering, and language. Mines the session (working-tree diff, new files,
  rejected alternatives, defects found during implementation) for context,
  evaluated options, the decision, consequences, and code references. Use when
  the user says "write an ADR", "record this decision", "/grill-adr", "document
  why we did this", "capture this session as an ADR", or asks for an
  architecture decision record after an implementation. Do NOT use for specs,
  design docs, changelogs, or commit messages.
modeSlugs:
  - documentation-writer
  - orchestrator
---

# grill-adr

Turn a finished working session into one ADR that a future reader can use to understand *why* the
code looks the way it does. The record is written from evidence present in the session, never from
imagination.

## When to use

- A non-trivial implementation just landed and the reasoning would otherwise be lost.
- Several alternatives were weighed, and some were rejected for specific reasons.
- The user asks to "grill" the session into an ADR or decision record.

## When NOT to use

- The session was mechanical (formatting, dependency bump, typo fix): there is no decision to
  record. Say so and stop.
- The user wants a spec, plan, RFC, or commit message instead.
- Nothing has been decided yet. An ADR records a decision that was made; if the decision is still
  open, write it with status `Proposed` and state that explicitly in the final report.

## Inputs

- Required: the session itself (conversation plus working-tree changes).
- Optional: title, ADR number, or scope limits supplied by the user.

## Workflow

1. **Survey the ADR folder.**
   - List `docs/adr`. Create it only if the repository has none.
   - Read the newest file in `docs/adr`. It is the template: copy its section names, heading style,
     frontmatter (if any), status wording, and language.
   - Next number = highest existing `NNNN` + 1, zero-padded to four digits. Never reuse, renumber,
     or rewrite an existing ADR.

2. **Harvest the evidence.**
   - `git status --porcelain`, `git diff`, `git diff --stat` for the real scope of the change.
   - New and untracked files usually reveal the shape of the decision.
   - From the conversation: the problem, the alternatives considered, why each rejected one failed,
     defects uncovered during implementation, trade-offs explicitly accepted.

3. **Draft** `docs/adr/NNNN-<kebab-case-title>.md`.
   - Title: `# ADR-NNNN: <the decision in one line>`, not a topic label.
   - File name: kebab-case, at most six words, no dates.
   - Language: match the existing ADRs. ADRs in this repo (`space-server`) are written in Russian;
     keep their section names verbatim.

4. **Fill the canonical sections**, using the repository's existing names. Skeleton below reflects
   the `space-server` shape; adapt names when the target repo differs.

5. **Verify before finishing.** Every path in the reference list exists, every line number points at
   the symbol described, and every count or number in the body comes from the session.

6. **Report** the created path, its title, and which alternatives were recorded as rejected.

## Skeleton

```markdown
# ADR-NNNN: <decision>

## Статус
Принято (Implemented) — YYYY-MM-DD.

## Контекст
Current state, constraints, the specific gap or risk. Quote real identifiers, tables, and routes.

## Рассмотренные варианты

| Вариант | Оценка |
| --- | --- |
| <alternative> | Отвергнуто. <concrete reason it cannot work here> |
| <alternative> | **Принято.** <why it holds> |

## Решение
One subsection per mechanism. Code or SQL fragments only where they carry the decision.

## Последствия
Положительные: ...
Отрицательные и ограничения: ...

## Отклонения, выявленные при реализации
Numbered list of plan defects found and fixed. Include only if the session had them.

## Ссылки
- [`File.cs`](relative/path/File.cs:LINE) — what it settles.
```

## Content rules

- One ADR = one decision. Split unrelated decisions into separate records.
- Rejected alternatives need the mechanism-level reason, not "considered, not chosen".
- Consequences must state the accepted cost (serialization, extra transactions, documented
  asymmetries), not only the benefits.
- The deviations section exists because plans are wrong: record what the plan got wrong and what
  the code actually does now.
- Cite code as repo-relative paths with line numbers; markdown links resolve from the repository
  root. Percent-encode spaces in file names (`005%20-%20ProjectSecurity.sql`).
- No dates besides the status date. Use the past tense for what the code does — no "we will later";
  open items belong in the final report, not the ADR body.
- Never document anything the session did not establish.

## Boundaries

- Writes only under `docs/adr/`. Does not touch code, migrations, or existing ADRs.
- A decision that reverses an earlier ADR becomes a new ADR carrying `Supersedes ADR-NNNN` in its
  status line; the old record is left as history.

## Troubleshooting

- **No `docs/adr` directory**: create it and start at `0001` with English section names.
- **Session contains several ideas**: pick the one decision the diff actually implements and record
  the rest as rejected alternatives or follow-ups.
- **Existing ADRs disagree with the code**: document the code's behaviour and note the divergence
  under the deviations section rather than silently "fixing" history.
