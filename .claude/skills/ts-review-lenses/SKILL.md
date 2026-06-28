---
name: ts-review-lenses
description: >-
  Wire and run three adversarial TypeScript review lenses — taste (idiomatic
  with-the-grain TypeScript, domain-types-first), correctness-vs-reality
  (boundary validation, grounded tests,
  secret hygiene), architecture (boundaries, purity seam, dependency direction)
  — in a @swamp/software-factory run. Use when authoring or driving a factory's
  plan-review or code-review stage and these lenses are present: at author time,
  set the review stages up for the three lenses; at drive time, run each lens as
  an independent external reviewer via @atalanta/external-reviewer's
  external-review-findings workflow and record its findings. Also covers the
  same-context dispatch fallback when no external agent is configured.
---

# TS Review Lenses

Three adversarial code-review prompts, distilled from failure modes that escape
*green* test suites. Run them as **three separate passes** — one reviewer per
lens — so taste, correctness, and architecture concerns don't dilute each other.

| Lens | File | Obsession | Finding id prefix |
| --- | --- | --- | --- |
| A — Idiomatic taste | `references/lens-a-idiomatic-taste.md` | Idiomatic with-the-grain TS, domain-types-first, no transliteration, no paradigm zealotry | `TASTE-` |
| B — Correctness vs reality | `references/lens-b-correctness.md` | Boundary validation, grounded tests, secret hygiene, IO correctness | `CORR-` |
| C — Architecture | `references/lens-c-architecture.md` | Module boundaries, purity seam, dependency direction, change confinement | `ARCH-` |

A complete reviewer prompt is **one lens file + `references/contract-block.md`**
(the swamp/findings contract). The lens prose is transport-neutral; the contract
is what makes the output a recordable factory artifact. When two lenses conflict,
**correctness (B) outranks taste (A)**.

Pick lenses by stage: `code-review` → all three; `plan-review` → C, optionally A
(taste/correctness need real code to bite).

## When you are authoring a factory (set the review stages up for the lenses)

Trigger: you are filling a `@swamp/software-factory` definition's `plan-review`
or `code-review` stage (e.g. the `@atalanta/external-reviewer` skeleton's `TODO`
review stages) and these lenses are available.

```
Author checklist:
- [ ] Review stages are mode: dispatch with a kind: findings artifact
- [ ] The findings artifact schema matches references/contract-block.md
      (id, severity critical|high|medium|low, description, category?, resolved?, resolutionNote?)
- [ ] code-review uses lenses A+B+C; plan-review uses C (+A)
- [ ] Decide the transport: external reviewer (default) or dispatch fallback
- [ ] Note in the stage description that findings come from N per-lens passes
      with prefixes TASTE-/CORR-/ARCH-
- [ ] swamp model method run <factory> validate  → passes
```

Do **not** inline the lens prose into the factory YAML — keep it referenced from
these files so it stays versioned and editable in one place. The stage's job is
only to declare the findings artifact and its gates; the prompt is assembled at
drive time (below).

## When you are driving a review stage (run the lenses)

Trigger: a factory run has reached a `plan-review` or `code-review` stage whose
findings should come from these lenses.

Run **one external-reviewer invocation per lens**, then record the merged
findings. Use the factory repo as `cwd` so the reviewer's `swamp data query`
reads the right datastore.

```
Drive checklist (repeat the invoke+record for each chosen lens):
- [ ] record_dispatch on the factory stage (per the software-factory drive loop)
- [ ] Lens B (CORR-):   invoke → query invocation → resolve_findings
- [ ] Lens A (TASTE-):  invoke → query invocation → resolve_findings
- [ ] Lens C (ARCH-):   invoke → query invocation → resolve_findings
- [ ] Re-check factory status; advance when findings-clear passes
```

**Step 1 — invoke the lens.** Concatenate the lens file and the contract into the
`prompt` input (two `cat`s — the lens prose then the contract; do **not** `cat`
`using-with-external-reviewer.md`, which is explanation, not contract):

```bash
swamp workflow run @atalanta/external-reviewer/external-review-findings \
  --input factoryName=my-factory \
  --input workItem=ISSUE-42 \
  --input artifact=code-review \
  --input cwd=. \
  --input prompt="$(cat .claude/skills/ts-review-lenses/references/lens-b-correctness.md
                    cat .claude/skills/ts-review-lenses/references/contract-block.md)"
```

**Step 2 — read the findings back.** `invokeAndParse` persists the parsed JSON on
the reviewer model's `invocation` resource, not as a step output:

```bash
swamp data query 'modelName == "external-reviewer" && specName == "invocation" \
  && attributes.tags.workItem == "ISSUE-42" \
  && attributes.tags.artifact == "code-review"' \
  --select '{"findings": attributes.parsedResponse, "ok": attributes.success}' --json
```

**Step 3 — record on the factory.** Pass the returned findings to the factory's
`resolve_findings` (or `record_artifact` for the first recording). The
`TASTE-`/`CORR-`/`ARCH-` prefixes keep the three passes' ids from colliding when
the `findings-clear` gate evaluates them.

**Step 4 — repeat** for the remaining lenses, then advance the factory once
`findings-clear` is satisfied. Substitute `--input artifact=plan-review` for a
plan-review stage.

If the reviewer returns `ok: false` or empty `findings`, do not re-dispatch
blindly — read the invocation's `outputPreview` / transcript for why (auth,
provider, malformed JSON) per the software-factory dispatch guard.

## Fallback: same-context dispatch (no external agent)

If no external reviewer is configured, run the review stages as vanilla
`mode: dispatch` with one subagent per lens: each subagent's `systemPrompt` is
the lens file + `references/contract-block.md`, and it reviews the diff from its
own context instead of via `swamp data query`. Merge into one `kind: findings`
artifact with the same id prefixes. This shares the implementer's model family,
so it is review but not *independent* review — prefer the external path when both
are available.

## References

- [references/lens-a-idiomatic-taste.md](references/lens-a-idiomatic-taste.md) — Lens A prompt
- [references/lens-b-correctness.md](references/lens-b-correctness.md) — Lens B prompt
- [references/lens-c-architecture.md](references/lens-c-architecture.md) — Lens C prompt
- [references/contract-block.md](references/contract-block.md) — paste-clean swamp/findings contract appended to a lens at run time
- [references/using-with-external-reviewer.md](references/using-with-external-reviewer.md) — how the lens + contract become a prompt (explanation, not for `cat`)

Lenses A/B use TypeScript examples (zod, `as`, Deno); B's and C's substance is
language-portable. For non-TS work, keep B/C and adapt A.
