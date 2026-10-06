# CLAUDE.md

`sqdist` — a single-binary Rust CLI that computes a **panel of independent similarity axes** across two strings for **typosquatting / homoglyph attack detection**: Levenshtein, Damerau-Levenshtein (OSA, adjacent transpositions), their UTS#39-skeleton variants, and confusable-involvement signals. The point is to separate a visual spoof (`pаypal` with Cyrillic а, which collapses to zero skeleton distance) from a benign typo (`gogle`, identical unweighted distance but no confusable characters).

## Commands

```sh
cargo build --release    # -> target/release/sqdist
cargo test               # all unit tests (in each module's #[cfg(test)] block; currently 136)
cargo test <name>        # one test, e.g. cargo test confusable_only_axis
cargo run -- <A> <B>     # run against two strings (note the -- before args)
cargo clippy && cargo fmt --check   # lint (no separate lint config)
```

Performance tooling (see [docs/performance-plan.md](docs/performance-plan.md)):

```sh
scripts/fetch-corpus.sh          # package-name corpora -> git-ignored corpus/
scripts/golden.sh capture|check  # output-regression gate over the full PyPI corpus (~2.5 min)
scripts/quality.sh capture|check # recall on 143 labeled pairs + emitted-row counts (~2 min)
scripts/bench.sh <name>          # hyperfine matrix -> bench-out/<name>/
cargo build --release --features dhat-heap \
  --config profile.release.strip=false --config profile.release.debug=1   # heap profile
```

**Any change that touches scoring must leave `scripts/golden.sh check` clean.** It pins the JSON output contract below; the optimizations to date are all output-preserving.

**Any change that touches the emit gate must also leave `scripts/quality.sh check` clean.** golden.sh pins output for the queries it names; quality.sh pins *detection*: per-technique recall over the 143 labeled typosquat pairs, plus emitted-row counts over a top-15k × full-PyPI sweep. It is the gate that would catch a prune trading recall for speed. Both were verified to fail on a deliberately narrowed gate, not merely to pass.

## Architecture

Ten source files plus build-time helpers:

- [src/main.rs](src/main.rs) — CLI arg parsing (`Opts`), I/O, the three modes (single-pair / `--stdin` batch / `--string`+`--list` watchlist), output formatting (human table + JSONL), and orchestration calling the panel + verdict / typosquat classification. Batch/list scoring runs the cheap part first: the identity gate (`identity_gate` + `scores_normalized`) decides which strings would be scored, `may_emit` asks whether the pair could survive the emit filter, and only survivors get a panel. List mode builds a `Query` once (a `Prepped` — string + char vector + skeleton — for the raw query and, when a normalizer is configured, for its normalized form), so the `--string` side is never re-normalized or re-skeletonized per candidate. The release profile (Cargo.toml) is tuned for a small fast binary (`lto`, `panic = "abort"`, `strip`).
- [src/distance.rs](src/distance.rs) — edit distances only: unweighted integer `levenshtein`, `damerau` OSA, `damerau_within` (the banded `O(n*k)` `damerau <= k` predicate used by the list-mode emit gate; generic over element type so callers can pass `&[u8]` for ASCII), and the alignment traceback (`AlignOp` + `align`, which returns `(Vec<AlignOp>, u64)` — the ops plus the Damerau distance from the matrix it already fills, so callers needing both never fill it twice). The skeleton model lives in `src/confusables.rs`, not here.
- [src/confusables.rs](src/confusables.rs) — runtime confusable model: `ConfusableMap` (active single-char map + digraph list built from the selected sources), `parse_sources` (validates the `--confusables` comma-list and returns `Result<Sources, String>`, where `Sources` is `{ flowcrypt: bool, digraph: bool }` — `uts39` is always on), and the `ConfusableMap` methods `skeleton_of`, `confusable`, `skeleton_chars` (the hot path: skeleton straight to `Vec<char>`, with a no-lookahead fast path when no digraphs are active), `skeleton_len` (a skeleton's char count without building it, for the emit gate), and `skeleton` (the `String` form, test-only). The map is source-parameterized, threaded through `PairContext`, and borrows the static UTS#39 table unless a supplement adds entries.
- [src/axes.rs](src/axes.rs) — the `Axis` trait, `AxisValue` (Int/Float/Bool/NA), `Direction`, `Phase`, `PairContext` (per-pair precompute: char vectors, skeleton char vectors, the alignment, and the Damerau distance it yielded), the 10 axis impls, the `ALL_AXES` registry (canonical order = single source of truth for JSON key order and emit order), the two-phase base/derived `build_panel`, and `--fields`/`--metric` parsing and validation (`parse_fields`, `validate_metric`, `metric_value`). `AxisValue::NA` renders as JSON `null` and human `n/a`; its `as_f64()` returns `None`, and `row_metric` falls back to `f64::INFINITY` so NA rows never match a finite `-t` and sort last.
- [src/keyboard.rs](src/keyboard.rs) — embedded stagger-aware US-QWERTY key-coordinate table for `keyboard_distance`; no external dependency. `MAX_KEY_DISTANCE` (the [0,1] normalizer) is a const; the O(K²) sweep that derives it is test-only, and `const_matches_computed` asserts they agree bit-for-bit.
- [src/normalize.rs](src/normalize.rs) — registry-name normalizer: `NormOp` (`Lower` / `Map` / `Collapse`), `apply_ops`, hand-rolled JSON `parse_ops` / `load_ops_file` (no serde), and `pypi_ops()` (PEP 503: lower → map `._-` → `-` → collapse `-`). Used by `--pypi` and `-n`/`--normalize <PATH>`.
- [src/verdict.rs](src/verdict.rs) — `Verdict` + `verdict()` (single-pair human spoof/benign tags) and `TyposquatClass` + `classify_typosquat()` for `--typosquat` (`identical` / `same_project` / `likely_typosquat` / `possible_combosquat` / `unrelated`). Classification is **not** an axis — never in `ALL_AXES` / `--fields`.
- [src/confusables_data.rs](src/confusables_data.rs) — **auto-generated, do not hand-edit.** `pub static CONFUSABLES: &[(u32, &str)]` (~6565 entries) sorted by code point, embedded at compile time so the binary needs no runtime data files or network.
- [src/flowcrypt_data.rs](src/flowcrypt_data.rs) — **auto-generated, do not hand-edit.** FlowCrypt `idn-homographs-database` single-char supplement: `pub static FLOWCRYPT: &[(u32, &str)]` (look-alike code point → UTS#39-anchored skeleton) + `pub static FLOWCRYPT_PROVENANCE: &str` (repo commit + retrieval date, surfaced by `-v`). Generated by `scripts/gen_flowcrypt.py`; the 19 MB source JSON is not committed.
- [src/digraph_data.rs](src/digraph_data.rs) — three curated multi-char digraph mappings (`vv→w`, `cl→d`, `nn→rn`) as `pub static DIGRAPHS: &[(&str, &str)]`. Hand-edited; changes are intentional additions.
- [Cargo.toml](Cargo.toml) — one runtime dependency (`unicode-security`). `dhat` is an optional dev-only heap profiler behind the `dhat-heap` feature.
- [build.rs](build.rs) — compile-time git SHA capture (`git rev-parse --short HEAD` → `SQDIST_GIT_SHA`, falls back to "unknown" for crates.io/git-less builds).
- [scripts/gen_confusables.py](scripts/gen_confusables.py) — regenerates `confusables_data.rs` from UTS #39 `confusables.txt`. Pure stdlib; resolves paths relative to the repo root, so run it from anywhere.
- [scripts/gen_flowcrypt.py](scripts/gen_flowcrypt.py) — regenerates `flowcrypt_data.rs` from the FlowCrypt JSON. Accepts `--homographs`, `--confusables`, `--source-commit`, `--source-date` for offline/pinned use; fetches from GitHub by default.

### Modes

Three modes, dispatched by a thin `main()`:

| Mode | Trigger | Pairing | JSON keys |
|---|---|---|---|
| single pair | two positionals | the two args | `a`/`b` |
| stdin batch | `--stdin` | pre-paired tab/comma lines | `a`/`b` |
| watchlist | `--string` + `--list` | `--string` × each file line | `input`/`match` |

**Emit gate (batch/list only).** `may_emit` in `main.rs` rejects candidates before any panel is built — on a registry corpus that is nearly all of them. Every test is *exact*, so the gate is a pure short-circuit, never a heuristic filter. Two stages, cheapest first:

1. **Length bound.** Edit distance is at least the length difference, on whichever strings the chosen metric scores (raw for `levenshtein`/`damerau`, skeleton for the skeleton axes; the other axes have no cheap bound and are not gated).
2. **Banded Damerau.** `distance::damerau_within(a, b, k)` — an `O(n*k)` OSA DP over the band `|i - j| <= k`, no traceback, early exit once the whole band exceeds `k`. It answers `damerau <= k`, not the distance. Exact for either metric because `damerau <= levenshtein`, and `k = floor(t)` because distances are integers. Skipped for `k > 4`, where the band stops paying. ASCII pairs run on bytes.

With `--typosquat`, a row is emitted only as `likely_typosquat` — needing `damerau <= 1` or `confusable_only` (equal skeletons) — or as a combosquat, which is a string test; the gate tests exactly those. **If you add or change an emit condition, widen the gate to match**, or rows will silently vanish. `gate_never_drops_a_row_the_filter_would_keep` cross-checks gated against unconditional scoring over every metric × threshold × mode.

**`--typosquat` profile:** opt-in package-registry detection profile. Default emit set is five axes (`equal`, `damerau`, `skeleton_damerau`, `confusable_only`, `keyboard_distance`); default `--metric` becomes `damerau` (unless `-m` was passed). Classifies each pair and emits `classification` + `reason` JSON keys after the axes. Cannot combine with `-t`/`--threshold`. A possible combosquat is a delimited affix/token of the other scored name (`-`, `_`, `.`, `/`; shorter side ≥ 3 chars).

**`--pypi` / `-n` normalize:** `--pypi` applies PEP 503 ops (lowercase; map `._-` → `-`; collapse `-`). `-n`/`--normalize <PATH>` appends ops from a JSON file (always a filesystem path — `--normalize pypi` opens a file named `pypi`; it is not a preset). Repeatable; `--pypi` and `-n` compose in argv order. When a normalizer ran and the normalized strings are equal but originals differ → identity-gate `same_project` (panel still on originals). Otherwise distances are on the normalized strings; JSON keeps original identifiers and adds `a_normalized`/`b_normalized` (or `input_normalized`/`match_normalized`).

### The 10-axis panel (canonical order)

`ALL_AXES` in `axes.rs` defines the canonical order for JSON keys and human-output rows. Base axes are pure functions of a `PairContext`; derived axes read already-computed base values.

| Key | Type | Phase | Meaning |
|---|---|---|---|
| `equal` | bool | base | the two strings are byte-identical |
| `levenshtein` | int | base | min single-char insert/delete/substitute edits |
| `damerau` | int | base | like levenshtein, but an adjacent transposition counts as one edit |
| `skeleton_levenshtein` | int | base | levenshtein after reducing both strings to their UTS#39 skeletons |
| `skeleton_damerau` | int | base | damerau after reducing both to UTS#39 skeletons (~0 when visually identical, incl. multi-char confusables like m↔rn) |
| `uts39_confusable_count` | int | base | # of substitution positions in the Damerau alignment whose two chars are UTS#39-confusable **(EXPERIMENTAL)** |
| `uts39_skeleton_delta` | int | derived | damerau − skeleton_damerau (saturating at 0); edits that vanish under skeletonization **(EXPERIMENTAL)** |
| `confusable_only` | bool | derived | strings differ but share an identical skeleton (highest-confidence spoof signal) |
| `script_restriction` | int | base | UTS#39 restriction level 0–5 of the pair (max of the two strings' levels); higher = more mixed-script and more suspicious. Latin+Cyrillic `pаypal` scores high, legitimate pure CJK stays low. Uses the `unicode-security` crate. |
| `keyboard_distance` | float\|NA | base | mean physical US-QWERTY key distance over substituted positions, normalized to [0,1]; near 0 = adjacent-key fat-finger, near 1 = far-apart/deliberate. NA (`null` / `n/a`) when either string is non-ASCII; zero substitutions = 0.0. |

### The confusable model

`ConfusableMap` in `src/confusables.rs` is built at startup from the `--confusables` sources (default `uts39`) and threaded through `PairContext` so all axes share one source set. Two characters are confusable when they share a **skeleton** under that set: `skeleton_of` binary-searches the sorted `CONFUSABLES` slice (plus any active supplement); `confusable(a, b)` compares skeletons (falling back to the char itself when unmapped), which transitively handles chains (Greek omicron, Cyrillic о, and Latin o all skeleton to the same thing). Skeleton comparison is **case-sensitive** per UTS #39 (`0`~`O` but not `0`~`o`) — tests encode this; don't "fix" it.

**`--confusables <LIST>`** sources:
- `uts39` — Unicode UTS#39 (authoritative, always active, ~6565 entries)
- `flowcrypt` — FlowCrypt `idn-homographs-database` single-char supplement (477 ASCII-base look-alikes anchored to UTS#39 skeletons; MIT; opt-in)
- `digraph` — 3 curated multi-char mappings: `vv→w`, `cl→d`, `nn→rn` (opt-in; `cl→d` has higher FP risk, e.g. `clear`/`dear`)

Unknown source → error (exit 2). UTS#39 wins on any collision. The three skeleton axes (`skeleton_levenshtein`, `skeleton_damerau`, `confusable_only`) reflect the active set.

**Multi-character skeletonization.** `ConfusableMap::skeleton(s)` maps each code point through the active sources and concatenates. With only `uts39` this catches `m` ↔ `rn` (a UTS#39 multi-char mapping); `digraph` closes the three curated gaps. These surface in `skeleton_damerau` (≈0 for a pure spoof) and set `confusable_only` (`a != b && skeleton(a) == skeleton(b)`, no equal-length requirement). Leetspeak (`3`→`e`) is intentionally excluded (UTS #39 does not treat it as visually confusable).

## Regenerating the data tables

UTS#39, when a new Unicode version ships:

```sh
curl -sSL https://www.unicode.org/Public/security/latest/confusables.txt -o confusables.txt
python3 scripts/gen_confusables.py   # rewrites src/confusables_data.rs
cargo test
```

`gen_confusables.py` keeps only entries whose **source is a single code point**, because `skeleton()` maps code points individually; multi-codepoint confusables are handled by the full-string skeletonization pass itself.

FlowCrypt (no upstream releases, so provenance is commit SHA + retrieval date):

```sh
python3 scripts/gen_flowcrypt.py   # online: fetches source JSON from GitHub; rewrites src/flowcrypt_data.rs
python3 scripts/gen_flowcrypt.py --homographs path/to/homographs.json --confusables path/to/confusables.txt \
  --source-commit <sha> --source-date <YYYY-MM-DD>   # offline / pinned
cargo test
```

## Output contract

Both human and JSON output expose the same fields; downstream pipelines rely on the JSON key names, so **keep them stable**. Key order and meaning: the 10-axis panel table above.

- **Identifier keys:** `a`/`b` (single pair, `--stdin`); `input`/`match` (watchlist). With a normalizer: also `a_normalized`/`b_normalized` or `input_normalized`/`match_normalized`. With `--typosquat`: also `classification` and `reason`, after the axes.
- **Batch and list output** is always JSONL. With `-t`, only rows at/under the threshold are emitted and the exit code is 1 when zero rows match (0 otherwise; always 0 without `-t`). With `--typosquat`, only `likely_typosquat` and `possible_combosquat` rows are emitted, exit 1 if none.
- **`--fields <comma-list>`** filters emitted axes in all modes; names are validated against `ALL_AXES` (an invalid name errors, listing valid keys). Identifier keys are always kept; fields emit in canonical order. `-t`/`--metric`/`--sort` operate on the full internal scores regardless. `classification` is not an axis and cannot appear here.
- **`--metric <axis>`** selects the numeric axis driving `-t` and `--sort`. Default `skeleton_damerau` (`damerau` under `--typosquat` unless `-m` was passed). The bool axes (`equal`, `confusable_only`) are rejected with an error listing the valid keys.
- **Single-pair human verdict** (not in JSON or batch modes): `[IDENTICAL]`, `[LIKELY SPOOF]`, or `[LIKELY BENIGN]` from `verdict()`. Spoof rule: `confusable_only || (homoglyph_share > 0.5 && max(len) >= 3 && len_diff_ratio <= len_tolerance)`, where `homoglyph_share = uts39_skeleton_delta / damerau`; tune with `--len-tolerance` (default 0.25, range [0,1]). With `--pypi`/`-n` and no `--typosquat`, identity-gated pairs get `[SAME PROJECT]`. With `--typosquat`, tags are `[IDENTICAL]` / `[SAME PROJECT]` / `[LIKELY TYPOSQUAT]` / `[POSSIBLE COMBOSQUAT]` / `[UNRELATED]` from `classify_typosquat()`.
- **`-v/--version`** prints `sqdist <version> (<git-sha>)` plus one `data:` line per embedded confusable source (regardless of `--confusables`). UTS#39 provenance is the version + date from `confusables.txt`; FlowCrypt provenance is the upstream commit SHA + retrieval date.

**History:** v0.3.0 was a breaking schema change (removed `homoglyph_damerau`, `normalized`, `skeleton_normalized`; added `equal`, `skeleton_levenshtein`, `uts39_confusable_count`, `uts39_skeleton_delta`; dropped `--hogl-weight`/`-w`; `--metric` takes an axis key). v0.5.0 added `--typosquat`, `--pypi`, `-n` (additive; default output unchanged). v0.6.0 was performance only; JSON is byte-identical to 0.5.0.
