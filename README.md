# @atalanta/ts-review-lenses

Three adversarial TypeScript code-review prompts ("lenses") for feeding to an
**independent reviewer** in a [`@swamp/software-factory`](https://swamp-club.com)
run. Distilled from failure modes that escaped *green* test suites during real
development.

> Run them as **three separate passes**. Mixing "is this elegant" with "is this
> provably right" dilutes both.

| Lens | Obsession | Finding ids |
| --- | --- | --- |
| **A — Idiomatic taste** | Idiomatic with-the-grain TS, domain-types-first, no transliteration, no paradigm zealotry | `TASTE-` |
| **B — Correctness vs reality** | Boundary validation, fixtures grounded in the real wire, secret hygiene, subprocess/IO correctness | `CORR-` |
| **C — Architecture** | Module boundaries, the purity seam, dependency direction, verified change confinement | `ARCH-` |

**Ranking when lenses conflict:** correctness-against-reality (B) beats taste (A).

## What this ships

A single `ts-review-lenses` **skill** bundle:

| File | What it is |
| --- | --- |
| `references/lens-a-idiomatic-taste.md` | Lens A review prose (transport-neutral) |
| `references/lens-b-correctness.md` | Lens B review prose |
| `references/lens-c-architecture.md` | Lens C review prose |
| `references/using-with-external-reviewer.md` | The machine-facing contract (read from swamp, emit findings JSON, id prefixes, re-review) appended to a lens at run time |
| `SKILL.md` | The wiring runbook |

No models, no workflows. The lens files are **transport-neutral prose**; the
swamp/findings contract is appended at run time so the same prose serves both
external review and same-context dispatch.

## Quick start

```bash
swamp extension pull @atalanta/ts-review-lenses
# then invoke the bundled `ts-review-lenses` skill and follow the runbook
```

### Primary path — external, independent review

Pairs with [`@atalanta/external-reviewer`](https://github.com/Atalanta/swamp-external-reviewer)
(an **optional** companion, not a hard dependency). The factory driver
implements; a separate external CLI agent reviews under one lens at a time,
sharing no context. One bridge invocation per lens:

```bash
swamp workflow run @atalanta/external-reviewer/external-review-findings \
  --input factoryName=my-factory --input workItem=ISSUE-42 \
  --input artifact=code-review --input cwd=. \
  --input prompt="$(cat .claude/skills/ts-review-lenses/references/lens-b-correctness.md
                    cat .claude/skills/ts-review-lenses/references/using-with-external-reviewer.md)"
```

Then query the reviewer's invocation for `attributes.parsedResponse` and record
the findings on the factory via `resolve_findings`. Repeat for lenses A and C.
The `TASTE-`/`CORR-`/`ARCH-` prefixes keep the three passes non-colliding.

### Fallback path — same-context dispatch

A factory that doesn't want an external agent uses vanilla `mode: dispatch`
review stages with each lens as a `systemPrompt`, one subagent per lens. The lens
prose is identical; only the appended contract differs. The bundle adds
capability; it removes nothing.

## Provenance

These encode failure modes that passed green tests in real work: lying fixtures
(tests written from an *assumed* external shape), boundary erasure (TS types
prove nothing about external data), transliteration (code that wears TS syntax
but thinks C#/Python/F#), string-surgery parsing, and secret leaks through error
messages and structural formatting. Lens B is the direct answer to most of them.

## License

MIT. See `LICENSE.txt`.
