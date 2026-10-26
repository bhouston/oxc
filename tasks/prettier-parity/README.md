# Prettier adoption investigation

This is a fork-specific investigation for agent-swarm issue 152, not a change to
Oxfmt's formatting policy. Three implementation PRs need the original failing
examples before their scope can be established.

## Versions and limits

- Inspected Oxc commit `3fbef5bac00fca75d54d7a07b5d280e102f5b63b`.
- Compared the published native Windows package `oxfmt@0.74.0` with
  `prettier@3.9.10`, the version pinned in `apps/oxfmt/package.json`.
- Inspected Prettier source at `7664e36875350ad0f309287391858de19ed13eb7`.
- Set `printWidth` explicitly: Prettier defaults to 80, Oxfmt to 100.
- These are targeted output comparisons, not proof of complete compatibility.
  The published binary was tested; the inspected Rust checkout was not built.

## Findings

| Requested area                      | Evidence                                                                                                                                                                   | Remaining uncertainty                                                                   |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Class/interface `extends` wrapping  | 96 generated cases match exactly at widths 40/80/100.                                                                                                                      | The project's original failing example is unavailable.                                  |
| Generic `extends` constraints       | 200 of 210 generated cases match; 10 differ in union wrapping.                                                                                                             | Whether the requested change is the deliberate Prettier 3.9 union-style difference.     |
| Arrow-argument hugging              | 420 generated cases match and are idempotent; eight additional comparisons of upstream return-type fixtures match.                                                         | Whether the project hits a documented exact-width divergence or another untested shape. |
| Nested conditional-type indentation | Six upstream fixtures at three widths, with spaces/tabs: 36 cases, all idempotent; 34 exact matches. The other two differ in comment/union layout, not nested indentation. | Whether the project enables `experimentalTernaries`.                                    |

The generic constraint matrix covers type aliases, interfaces, classes, functions,
and arrows with primitive, named, qualified, generic, union, intersection, and
object constraints. The arrow matrix covers five return annotation shapes, seven
body shapes, three argument positions, and four widths.

### Extends wrapping

Related fixes already merged upstream:

- [#15892](https://github.com/oxc-project/oxc/pull/15892): consistent class and interface extends printing.
- [#16323](https://github.com/oxc-project/oxc/pull/16323): indentation of qualified class/interface heritage without type arguments.
- [#16334](https://github.com/oxc-project/oxc/pull/16334): class extends/implements indentation.

Prettier's simple superclass behavior is intentional:
[#18424](https://github.com/prettier/prettier/issues/18424) was closed as expected
behavior; the older wrapping discussion
[#10100](https://github.com/prettier/prettier/issues/10100) remains open.

A different, reproducible constraint mismatch is:

```ts
type A<T extends A | B | C | D | E | F | G | H | I> = T;
```

At width 40, Prettier retries the union on one indented line after `extends`;
Oxfmt expands its members with leading `|`. This comes from Prettier's merged
[union rewrite #18827](https://github.com/prettier/prettier/pull/18827), whose style
is contested in [#19733](https://github.com/prettier/prettier/issues/19733).
The opening comment of `crates/oxc_formatter/src/print/union_type.rs` explicitly
retains the previous expansion style pending a decision. See also
`crates/oxc_formatter/DIVERGENCES.md#union-annotation-flat-retry` and
[Oxc #25841](https://github.com/oxc-project/oxc/issues/25841).

### Arrow-argument hugging

[Oxc #25044](https://github.com/oxc-project/oxc/pull/25044), merged July 29, 2026,
already handles type-reference return annotations such as:

```ts
a.map((): A => ({
  name,
}));
```

Related Prettier changes are established behavior, not experimental options:
[#10316](https://github.com/prettier/prettier/pull/10316) and
[#10714](https://github.com/prettier/prettier/pull/10714).

[Oxc #21189](https://github.com/oxc-project/oxc/pull/21189) remains open for an
exact-print-width grouped-arrow case. Oxfmt explicitly documents its current
choice in `crates/oxc_formatter/DIVERGENCES.md#arrow-exact-fill-signature`.
Changing it would be an intentional fork policy decision, rather than enabling
general arrow hugging.

### Nested conditional types

This mixed true/false-branch example matches default Prettier at width 60 and is
idempotent in Oxfmt:

```ts
type Result<T> = T extends string
  ? T extends "first"
    ? "first branch"
    : "other string branch"
  : T extends number
    ? "number branch"
    : "fallback branch";
```

With `experimentalTernaries: true`, Prettier produces a different layout. Oxfmt
does not support that option. The work has already been discussed and submitted:

- [Open issue #14914](https://github.com/oxc-project/oxc/issues/14914).
- [Closed implementation PR #26008](https://github.com/oxc-project/oxc/pull/26008).
- [October 23 follow-up](https://github.com/oxc-project/oxc/issues/14914#issuecomment-6365259258)
  reports a rebased downstream implementation at `ssalbdivad/oxc` on
  `experimental-ternaries`, with union indentation, tab-width, and predicate-type
  fixes, published as `@ark/oxfmt`. Those downstream claims were not independently
  validated in this investigation.
- [Merged #25665](https://github.com/oxc-project/oxc/pull/25665) covers related union
  comment and indentation corrections.

## Adoption implications

Oxfmt offers native JS/TS formatting and other native or bundled-Prettier-backed
languages; it does not promise byte-for-byte parity. Review the current
[language support](https://oxc.rs/docs/guide/usage/formatter/language-support) and
[unsupported features](https://oxc.rs/docs/guide/usage/formatter/unsupported-features)
against the project's actual languages, plugins, and options before migrating.

Import sorting is a separate Oxfmt feature, disabled by default and based on
eslint-plugin-perfectionist. See the
[sorting documentation](https://oxc.rs/docs/guide/usage/formatter/sorting).
The requested `walls2/` versus `walls/` difference was left outside this work.

## Next implementation step

Obtain one minimal input and expected output for each observed difference, plus
the Prettier/Oxfmt versions, resolved options, parser, and plugins. In particular,
establish whether `experimentalTernaries` is enabled. Then create one independent
feature branch and draft PR per reproduced missing feature against `bhouston/oxc`.

Follow existing formatter IR builders and AST predicates, avoid source-text
heuristics and repeated formatting, and add fixture snapshots verified against
the pinned oracle. Run the required crate tests/conformance, both ordinary and
`detect_code_removal` feature checks, and Clippy for each implementation branch.

This investigation used OpenAI Codex and parallel coding agents. No production
formatter changes or performance claims are included.
