# sitegen — a faster drop-in for makefun-site 1.0.0

`run(input)` returns exactly the output tree makefun-site 1.0.0's `build` returns for the same
input, byte for byte, but faster (about 2.4x the house baseline's CPU time on the practice set:
~245 ms vs ~585 ms per 3-case set, warm).

## How it is faster

The reference is a clean, general implementation; this one removes the work it repeats:

- **HTML escaping, batched.** The reference escapes every character of inline text with its own
  `replace` call. Here plain text is gathered into runs and escaped in one regex pass per run,
  and the runs are found with `indexOf` scans instead of a per-character loop.
- **Markdown, fused.** Lexing and rendering are one pass over the lines, with hand-rolled
  classifiers (fence, heading, thematic break, list, continuation) instead of a regex per line,
  and the inline renderer works on string ranges so strong, emphasis and link contents are never
  copied into fresh strings.
- **Templates, flat.** Each template is compiled once into a flat op list (text, var, raw, for,
  if, include) with jump targets, and rendering walks it with a scope stack. A `for` loop reuses
  one scope frame for all its elements. Variable lookup, truthiness and value-to-text follow the
  reference exactly.
- **Cached within a case.** Slugs (titles and tags repeat), long dates and resolved link paths are
  cached per `run` call; nothing is carried from one case to the next.
- **Cheaper front matter.** The `key: value` lines, dates, tag lists and slugs are parsed with
  direct character checks, producing the reference's exact refusal strings and line numbers.

## What it does not change

The semantics are the reference's, read from `vendor/makefun-site/` in the kit: the front-matter
grammar and its refusal reasons, the Markdown subset (fences, ATX headings with unique ids,
thematic breaks, blockquotes, flat lists, paragraphs; code spans, strong, emphasis, links and
backslash escapes inline), the template dialect (`{{ }}`, `{{{ }}}`, `{% for %}`, `{% if %}`,
`{% include %}`, standalone-tag line stripping), the slug rule (NFKD, combining marks dropped,
64 characters), the permalink and tag rules, the index/tag pagination and the build report.
Where the README and the code differ, the code wins, and this implementation follows the code.

## Files

- `solve.js` — the whole implementation; `run(input)` is the entry point.
- `main.ts` — the house driver, unchanged: reads `/input/cases.json`, answers each case with
  `run`, writes `/output/answers.json`. It turns V8's incremental marking off before importing
  `solve.js`, as the baseline does.

## Verification

Checked against the reference (`makefun-site` 1.0.0) on the kit's practice set and on five
freshly generated sets from the kit's own generator (18 cases, ~1,070 sources each set):
every answer identical as a JSON value. The practice set also runs through the kit's
`grade-local`.
