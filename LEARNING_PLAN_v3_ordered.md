# Learning plan: Salsa → `ra_ap_*` → a retrieval-backed coding agent

**Pace:** 5h/day, slow reading. ~11 days of reading, ~6 of building. Roughly three weeks.

**Ordering:** the four salsa projects are read in your order — 294 → 333 → 465 → 730
lines — so the material grows as you go. `3-salsa-calc` is read core-files-only (skip its
800-line parser), and `4-salsa-sample` is read in full.

**Measured scope:** the `ra_ap_*` stack is ~438k non-test lines across 33 crates.
This plan reads ~5,300 of them — about 1.2%. Everything else is on-demand reference.

**Sources live in this workspace.** The `ra_ap_*` crates themselves are in the cargo
registry cache, not in any project here:

```
/home/schmidh/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/ra_ap_*-0.0.352
```

One trap to avoid: `CodeQL-Extractor/traps/.../ra_ap_*/src/lib.rs/` contains **CodeQL TRAP
databases** (`*.trap.zst`), not source. Zero readable `.rs` files. Ignore it entirely.

---

## The three ideas that carry the most value

Everything below exists to get you these:

1. **Queries are memoized and invalidated per-field.** Why `full_diagnostics` is fast on an
   unchanged project, and why a one-file edit doesn't recompute everything.
2. **The database is mutable only through `&mut`.** This is the `apply_change` deadlock
   documented in `ra-ide-sample/README.md`. It's the first bug you'll hit.
3. **Syntax tree and HIR are different trees, and HIR nodes may have no file.** When your
   agent asks what an identifier refers to and the answer came from a macro expansion,
   the HIR node lives in a synthetic file. You must map back with
   `Semantics::original_range` or hand the LLM a range in a file that doesn't exist.

#3 is why days 6–9 exist. Interned structs, `specify`, and accumulators-as-a-concept can be
looked up when needed.

---

## Day 1 · The six macros

**Read:** `salsa/1-salsa-overview/src/main.rs` (294). Assembled from the Salsa book's
overview chapter, annotated section by section — each block is labelled with the heading it
corresponds to.

This is the right opener: it's the shortest file, it introduces every macro at once, and
it's a straight read with nothing to run first.

- **Input** (`#[salsa::input]`) — the only mutable state. The struct stores nothing; it's a
  newtype over an integer id. Read via getters that take the db, write via generated
  setters that take `&mut db`. Each setter bumps the revision. **That `&mut` vs `&` split is
  the whole safety story.**
- **Tracked function** (`#[salsa::tracked]`) — a memoized query. Constraints: db first, then
  either nothing, one Salsa struct, or `Eq + Hash` args (interned together into a key).
- **Tracked struct** (`#[salsa::tracked]`) — an intermediate value owned by its creating
  query. The subtlety that trips everyone is `#[tracked]`: untracked fields define *who the
  item is*, tracked fields define *what it currently says*. On re-execution Salsa matches by
  identity and diffs tracked fields individually, so invalidation is per-field.
- **Interned** (`#[salsa::interned]`) — cheap equality. `Word::new(db, "foo")` returns the
  same id every time, so `w1 == w3` is one integer compare.
- **Accumulator** (`#[salsa::accumulator]`) — side outputs that don't pollute the return
  value. Read with `query::accumulated::<T>`. This is exactly how rust-analyzer reports type
  errors.
- **`specify`** — override a query's result for one specific struct, called inside the query
  that created it.

Then read `salsa/1-salsa-overview/README.md` — a strong defense of why you can't just use
`String` for a memoized field (memoization hashes and compares inputs constantly, and
interning makes that O(1) and shares storage). Short, and it pre-empts a question you'll have.

Also run `cargo run` here. The `main` function is staged so you can watch which queries
re-execute after `set_contents` — `eprintln!(" [track] …")` markers fire on stderr.

**Skippable if short on time:** `specify`. Interning is worth keeping.

---

## Day 2 · The spreadsheet

**Read:** `salsa/2-salsa-excel-replica/src/main.rs` (333) — a reactive Excel model. Small
enough to finish in a session, and it applies day 1's macros to something that isn't a
compiler.

```sh
cd salsa/2-salsa-excel-replica && cargo run
```

The file is already numbered into sections (1. Database, 2. Input, 3.1 Parse, 3.2 Evaluate,
Cycle Detection, 4. Accumulator) — follow them. Three payoffs:

- **`cycle_result` recovery.** `evaluate_cell` is declared
  `#[salsa::tracked(cycle_result = evaluate_cell_cycle_recovery, ...)]` (`main.rs:169`).
  Set `A1 = "= B1"` and `B1 = "= A1"` and the recovery function returns `Err("#CYCLE!")`
  instead of panicking. The most reusable thing in the file.
- **Why a cycle-participating query can't also accumulate.** The comment at `main.rs:165`
  is explicit — Salsa panics with "Fixpoint iteration doesn't support accumulated values."
  Hence the two-function split: `evaluate_cell` is the pure value query and participates in
  cycle resolution, while `evaluate_cell_and_report` is the diagnostic wrapper that never
  recurses into itself and so can safely `accumulate`. A real architectural constraint,
  not a style choice.
- **The custom `Db` view trait.** `evaluate_ast` needs to look up cells by coordinate, which
  is plain data on the database struct rather than a salsa ingredient. The `#[salsa::db]`
  trait exposing `get_cell` is how a tracked function reaches it without `Any`-based
  downcasting. `ra_ap`'s `RootDatabase` uses exactly this pattern for ambient state.

The incremental payoff is visible in the output: change `A1`, and `parse_formula` stays
memoized for `B1` and `C1` while only the arithmetic re-runs.

There's also an `assets/` directory — ten `.docx` files that read like the design
conversation that produced this (including "03 - could I build an Excel replica with it"
and "07 - Salsa's cycle-detection recovery strategy"). Skim them if you want the reasoning
behind the structure; not required.

---

## Day 3 · A real compiler, lightly

**Read** `salsa/3-salsa-calc/` — the core only:

- `src/ir.rs` (139) — `SourceProgram` input, `VariableId`/`FunctionId` interned,
  `Program`/`Function`/`Span` tracked, plus plain `SalsaValue` types inside them.
- `src/db.rs` (55) — database impl plus the event-log callback, with
  `enable_logging()` / `take_logs()` to read it back.
- `src/compile.rs` (9) — the entire dependency spine in one function:

  ```rust
  #[salsa::tracked(returns(copy))]
  pub fn compile(db: &dyn crate::Db, source_program: SourceProgram) {
      let program = parse_statements(db, source_program);
      type_check_program(db, program);
  }
  ```

- `src/type_check.rs` (262) — a real checker, and a good place to see accumulators used
  the way `hir_ty` uses them.

**Skip `src/parser.rs` (800).** A hand-written parser teaches nothing about Salsa that days
1–2 didn't. This is the single biggest time saver in the plan.

`README.md` is a concept→`file:line` table — use it as a lookup, not something to read
linearly. There's also a written tutorial in `tutorial/` (07-checker.md, 08-interpreter.md)
if you want the reasoning rather than the code.

---

## Days 4–5 · The incremental behavior, made visible

**Read (in this order):** `salsa/4-salsa-sample/` — all of it. This is the largest of the
four, and it's last because it's where the abstraction from the first three days becomes
observable.

- `src/db.rs` (133) — the boilerplate again, but now with the event callback wired for
  display rather than just `eprintln!`.
- `src/files.rs` (104) — the data model: `SourceFile` (input), `ParsedFile` (tracked
  struct), `Diagnostic` (accumulator).
- `src/queries.rs` (273) — `parse_file`, `line_count`, `total_todos` (recursive),
  `build_order` (graph walk with cycle detection). Includes 5 tests.

**Do this first, before reading `queries.rs`:**

```sh
cd salsa/4-salsa-sample && cargo run
```

The five scenarios are the whole lesson. Watch `EXECUTE` vs `REUSE`:

| Run | What it demonstrates |
|---|---|
| 1. Cold start | every query executes |
| 2. No change | pure cache hits, no output at all |
| 3. TODO in leaf file | **only the `parse_file`/`total_todos` chain re-runs; `build_order` and `line_count` are REUSEd** |
| 4. Missing import | accumulator fires; `line_count` stays cached — its field didn't change |
| 5. Dependency cycle | `build_order` recovers and reports the path; `total_todos` has no recovery so Salsa would panic — compare with day 2's `cycle_result` |

Then `cargo test` — `editing_a_leaf_only_re_executes_downstream_queries`
(`queries.rs:229`) asserts scenario 3 directly.

**Gate for day 5:** be able to explain why `line_count` survives the scenario-4 change.
The answer is that invalidation tracks *fields* and *edges*, not files. Don't start
day 6 until you can say it out loud.

**Also on day 5:** `salsa::attach(db, || …)` is what makes printed keys human-readable, and
`DidValidateMemoizedValue` is the "memo was still valid" event you saw logged as `REUSE`.
Both are worth one more look now that the whole picture is assembled.

---

## Days 6–7 · The `ra_ap_*` tour, part 1: syntax and loading

**Read:** `CodeQL-Extractor/examples/` — the repo's own "tour of the `ra_ap_*` crates."
Read the output as much as the code; each program prints commentary as it runs.

- `examples/01_syntax.rs` (571) — what is actually in a parse tree. Needs **no project
  and no Cargo**, so you can start immediately.
- `examples/02_workspace.rs` (434) — how you get from a path to a parsed file. The full
  load pipeline: `load_workspace_at`, `Vfs`, `SourceFile`, `SyntaxNode`.

```sh
cd CodeQL-Extractor
cargo run --example 01_syntax -- src/config.rs
cargo run --example 02_workspace -- .
```

These use only `ra_ap_*` crates the repo already depends on, so they build alongside
the extractor at no extra cost.

**Why this is here:** it's 1,000 lines standing in for the 42k lines of `ra_ap_syntax`
plus the load pipeline. Read it *after* Salsa so you know what a salsa input is when you
see `SourceFile` and a tracked query when you see the parse step.

---

## Days 8–9 · The `ra_ap_*` tour, part 2: semantics

**Read:** `CodeQL-Extractor/examples/03_semantics.rs` (679).

```sh
cargo run --example 03_semantics -- . src/main.rs
```

This is the **most important file in the plan** for your agent. It answers "what does a
name *refer to?" — the exact question retrieval tooling asks — and it teaches the
syntax↔HIR distinction:

```
SyntaxNode   (ra_ap_syntax)   per-FILE,    lossless, untyped, includes whitespace
        │
        │  source_to_def / Resolver
        ▼
HIR          (ra_ap_hir)      per-CRATE,   typed, resolved, macros expanded
```

Two consequences that bite:

- A `SyntaxNode` always maps back to a real file and range.
- **A HIR node might not.** Macro expansions live in synthetic files. Use
  `Semantics::original_range` to map back — section 4 of the example shows this.

It also introduces `EditionedFileId`: the same file gets a different HIR file per
edition, because a 2015 dependency and your 2024 crate parse differently. That is *why*
`base_db::EditionedFileId` and `span::EditionedFileId` are distinct types — the gotcha
quoted in the agent README has a reason.

**Read `examples/README.md` first.** It's ~40 lines and states the mental model, the
reading order, and the two consequences explicitly.

**Gate for day 9:** implement it yourself — given a `SyntaxNode`, walk it to a
definition via `Semantics` and return a real file+range. If you hit a node with no file
of its own and `original_range` doesn't resolve it, you haven't finished. This is the
core retrieval primitive your agent will use on every request.

---

## Days 10–11 · The agent you're extending

**Read:** `rust-agent-demo/src/`

- `analyzer.rs` (279) — the wrapper. `LoadOptions` (five knobs: `all_targets`,
  `set_test`, `load_out_dirs_from_check`, `prefill_caches`, `num_worker_threads`) and
  the cancellation contract in the module doc comment. Now readable, since days 6–7
  covered the pipeline it wraps.
- `functions.rs` (187) — your first `hir` walk: `Crate` → `Module` → items, with
  `CrateOrigin::is_local()` filtering to workspace members and `HirDisplay`
  rendering semantic signatures. This is where you need `hir`.
- `daemon.rs` (201) — resident JSONL server. The process model you'll actually ship.

**Run, between reads:**

```sh
cd rust-agent-demo
cargo run -- demo-project     # the toy agent loop
cargo run --release -- demo-project --functions
cargo run --release -- demo-project --serve
```

**Read `README.md` end to end** — the "Rules that will save you" section alone is worth
the days. Specifically:

- Snapshots are cancelled by edits; one snapshot per query, never hold one across `apply_text`.
- The proc-macro client must outlive the program (`Analyzer::load` leaks it deliberately).
- rust-analyzer never reads disk after load — `apply_text` updates memory only.
- `Cargo.toml` edits need `reload`, which re-applies every file text and is therefore
  *not* incremental.
- Give salsa deep stacks (`stdx::thread::DEFAULT_STACK_SIZE`); type inference can
  overflow the default.

---

## Days 12–13 · The wider query surface

**Read:** `ra-ide-sample/src/`

- `demos.rs` (452) — one function per IDE feature.
- `workspace.rs` (131) — `AnalysisHost` + `ChangeWithProcMacros`, the plumbing.
- `configs.rs` (183) — the per-feature config structs, none of which implement `Default`.

**Run:**

```sh
cd ra-ide-sample && cargo run     # interactive menu
```

This is where you find out which of the 87 public `Analysis` methods you actually want.
Current coverage: hover, goto-definition, completions, diagnostics, rename, outlining,
folding, highlighting, inlay hints, symbol search.

---

## Day 14 (optional) · Inference

**Read:** `CodeQL-Extractor/examples/04_inference.rs` (564) — what *type* is this
expression.

```sh
cargo run --example 04_inference -- . src/crate_graph.rs
```

Only if you want type information in your agent's tools. This is the one place the tour
goes beyond the extractor — `src/` itself never does. Skip it if you're short on time and
read it later when a tool needs types.

---

## Days 15–20 · Build

Roughly a third of the total plan. Suggested order:

1. **Query abstraction layer** — a trait over the ~15 retrieval and mutation methods
   you're keeping, each returning JSON. This is the seam that lets you add agent tools
   without touching the analyzer core.
2. **Tool surface** — extend `daemon.rs`'s JSONL protocol to expose each query as a tool
   with a JSON schema. Reuse the existing protocol rather than inventing a second one.
3. **One tool end to end** — goto-definition → LLM decision → `SourceChange` →
   `apply_change` → re-query diagnostics. This exercises the whole stack and is where
   the cancellation contract will bite if you got it wrong.
4. **Breadth** — add the remaining tools, then add an edit-and-verify loop.

### Performance constraints (non-negotiable)

From `rust-agent-demo/README.md`, measured on this machine:

- **Resident, never subprocess-per-query.** Workspace load is 3.6 s (small crate,
  release, trimmed defaults); subsequent queries are 0.1–3 ms. First `diagnostics`
  call is ~352 ms as trait impls get computed; warm is 3 ms. `apply_text` is 6.8 ms.
  A subprocess-per-query design pays 3.6 s forever.
- **Optimize dependencies.** `[profile.dev.package."*"] opt-level = 2` is already set in
  `rust-agent-demo/Cargo.toml`. Debug is 12–16× slower. Without this a "simple" query
  takes 45 s and you'll assume your code is wrong.
- **`prefill_caches` is a no-op at `num_worker_threads: 0`** — `parallel_prime_caches`
  spawns workers in `0..num_worker_threads`, so zero threads returns immediately. Only
  matters if you set threads > 0.

---

## Deliberately excluded

Don't read these. They're implementation detail behind an API you'll call:

| Excluded | Lines | Why |
|---|---|---|
| `CodeQL-Extractor/src/generated/` | 16,108 | Generated. 88% of the extractor's `src/` is machine-written. |
| `CodeQL-Extractor/yeast*`, `tree-sitter-extractor` | 14,148 | tree-sitter for Ruby and Python. Unrelated to Rust semantics. |
| `CodeQL-Extractor/src/` (hand-written) | 2,980 | Only if you ever maintain the extractor. |
| `hir_ty` (all: `infer`, `next_solver`, `mir`, `method_resolution`) | 70,653 | Your agent calls `full_diagnostics` and gets inference results. Never reasons about them. Example `04_inference` covers the concept. |
| `hir_def` | 32,628 | Names-to-entities, behind `Analysis`. |
| `hir_expand` | 14,512 | Macro expansion, behind the loader. |
| `tt`, `mbe` | ~7,200 | Token trees, procedural macros. |
| `ide_assists` | 110,956 | 300+ independent handlers. Read four carefully, not three hundred. |
| `ide`, `ide_completion`, `ide_diagnostics`, `ide_ssr` (beyond demos) | ~196,000 | Sample via `demos.rs`, then go on-demand. |
| `vfs-notify` | 5,040 | Only if you touch file watching. |
| `3-salsa-calc/parser.rs` | 800 | See day 3. |

If you *do* eventually want the inference engine itself, `hir_ty/next_solver` (19.9k) is
the future and `hir_ty/infer` (14.5k) is what most current code calls. Pick one, don't
read both. Budget a week.

---

## Reference (open on demand, don't read in advance)

`rust-agent-demo/README.md` — the practical rules, performance numbers, and gotchas.
The most valuable section is "Gotchas discovered while wiring this up":

- Published names aren't always snake_case. `load-cargo` stayed hyphenated
  (`ra_ap_load-cargo`) because it was never in the autopublisher's rename list. If cargo
  says "no matching package", check `crates.io/api/v1/crates/<name>`.
- `unicode-ident` must be pinned to `=1.0.24`; `ra-ap-rustc_lexer` has a build-time
  assertion tying it to `unicode-properties`' Unicode version. (Same pin appears in
  `CodeQL-Extractor/Cargo.toml` with a fuller explanation.)
- API drift against `0.0.352`: `Analysis::file_structure` takes a `&FileStructureConfig`;
  `StructureNode` exposes `parent` not `depth`; `Diagnostic.range` is a
  `FileRange { file_id, range }`; `DiagnosticsConfig` has no `Default`, only
  `test_sample()`; `Analysis::from_single_file` wants a `triomphe::Arc`.
- `base_db::EditionedFileId` (salsa-interned) and `span::EditionedFileId` are distinct
  types. No `From` between them. Days 8–9 explain why.
- The VFS holds sysroot and dependency files too, not just your code — filter by project
  root (`Analyzer::local_rs_files`).
- `full_diagnostics` emits a torrent of `tracing` events at INFO. Default the subscriber
  to `WARN`.

The `ra_ap_*` crates have **no semver guarantees** — auto-published per rust-analyzer
release as `0.0.<build>`, pinning each other exactly. Pin all to the same `=0.0.x`,
upgrade as a set, check `docs.rs/ra_ap_ide/<version>` for drift.

---

## Progress checklist

- [ ] D1  `1-salsa-overview/main.rs` — all six macros; run it
- [ ] D1  `1-salsa-overview/README.md`
- [ ] D2  `2-salsa-excel-replica/main.rs` — cycle recovery, accumulator split, `Db` view trait
- [ ] D3  `3-salsa-calc/ir.rs`, `db.rs`, `compile.rs`, `type_check.rs`
- [ ] D3  *(skip `parser.rs`)*
- [ ] D4  `4-salsa-sample/db.rs`, `files.rs`; `cargo run` — watched scenarios 1–3
- [ ] D5  `4-salsa-sample/queries.rs`, `cargo test`
- [ ] D5  **Gate:** can explain why `line_count` survives the scenario-4 change
- [ ] D6  `CodeQL-Extractor/examples/README.md`, `01_syntax.rs`; run it
- [ ] D7  `02_workspace.rs`; run it
- [ ] D8  `03_semantics.rs` sections 1–3; run it
- [ ] D9  `03_semantics.rs` section 4 (`original_range`)
- [ ] D9  **Gate:** walk a `SyntaxNode` to a definition, return a real file+range
- [ ] D10 `rust-agent-demo/analyzer.rs`, `cargo run -- demo-project`
- [ ] D11 `functions.rs`, `daemon.rs`, `--serve`; README "Rules that will save you"
- [ ] D12 `ra-ide-sample/demos.rs`, `cargo run`
- [ ] D13 `workspace.rs`, `configs.rs`
- [ ] D14 *(optional)* `04_inference.rs`
- [ ] D15 Query trait + JSON serialization
- [ ] D16 Tool surface over JSONL
- [ ] D17–18 One tool end to end (goto-def → edit → verify)
- [ ] D19–20 Breadth + edit-and-verify loop