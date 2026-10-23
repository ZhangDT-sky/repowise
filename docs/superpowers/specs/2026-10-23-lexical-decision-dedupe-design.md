# Lexical Decision Dedupe

## Goal

Allow users of a zero-spend index built with the mock embedder to inspect and
fold reworded decision duplicates without pretending that mock vectors have
semantic similarity.

## User-facing behavior

`repowise decision dedupe` keeps its current behavior and continues to require
a real embedder. A new explicit `--lexical` flag enables the alternate matcher.
The flag is intentionally opt-in because the command can delete records when
run with `--apply`.

In lexical mode:

- no vector store is required or opened;
- candidates are foldable decision records from the same repository;
- two records are comparable only when they share at least one evidence commit;
- the compared text is the existing `decision_match_text(title, decision)`;
- similarity uses `difflib.SequenceMatcher` over normalized text;
- the default threshold is `0.70`, and `--tau` overrides it in the existing
  `0.5..1.0` range;
- dry-run output, JSON output, `--apply`, canonical selection, evidence moves,
  aliases, and reference repair use the existing dedupe plan/fold behavior.

The regular semantic path remains unchanged. Lexical plans report the selected
threshold and are not marked degraded merely because no vector store exists.

## Design

The CLI chooses one of two planning backends. The semantic backend retains the
current embedder and vector-store checks. The lexical backend calls a new core
planner that loads the same foldable records, builds an index from each record's
`evidence_commits_json`, and creates symmetric edges for pairs sharing a commit
whose normalized match text reaches the threshold. It then reuses `_star_cover`
to produce the existing fold clusters.

Evidence commits are read from the record-level JSON field because that data is
available to the dedupe command without changing the database schema. Records
with no evidence commit, or with no commit in common with another record, do not
produce lexical edges.

Normalization is deterministic and conservative: case-fold, collapse
whitespace, and compare the title-plus-decision text as one string. The matcher
does not compare arbitrary records across the repository, so ordinary decisions
that happen to use similar language are excluded unless their provenance points
to the same commit.

## Testing

Add unit coverage for:

1. identical and reworded records sharing a commit produce a lexical cluster;
2. records with similar text but different commits do not match;
3. records missing evidence commits do not match;
4. the threshold controls the edge;
5. lexical planning reuses the existing fold/apply semantics and remains
   idempotent;
6. the CLI accepts `--lexical`, skips embedder/vector-store construction in
   that mode, and preserves the current refusal without the flag on mock;
7. semantic dedupe behavior and existing reports remain unchanged.

## Non-goals

Automatic fallback from mock vectors, changes to the index-time dedupe path,
changes to decision identity or evidence schema, and cross-commit semantic
dedupe are outside this issue.
