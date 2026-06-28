# su26-ai301-contribution
# Contribution 1: (Benchmark) Store the benchmark result in a more readable format

**Contribution Number:** 1  
**Student:** Resham Pokhrel  
**Issue:** https://github.com/NodeSecure/js-x-ray/issues/612  
**Status:** Phase II Complete 

---

## Why I Chose This Issue

This issue caught my attention because it sits at a sweet spot between developer tooling and code quality - two things I genuinely care about. Transforming raw JSON benchmark data into a readable Markdown table is a focused, self-contained task that touches real skills: reading and parsing structured data, writing clean Node.js/TypeScript scripts, and understanding how CI pipelines automate file updates on pull requests.

I'm still building my depth in open-source contribution workflows, and this issue feels like the right size to do that well. I hope to get hands-on experience with mitata's output format, understand how automated benchmark tracking works in a production monorepo, and practice the full contribution cycle from reading the codebase to submitting a clean PR.

---

## Understanding the Issue

### Problem Description

Benchmark results are only saved as raw JSON, which is hard to read and compare at a glance.

### Expected Behavior

Results should also be available in a readable, table-like format (one row per benchmark, readable units).

### Current Behavior

Running the benchmark produces only `report.json` - accurate but not human-friendly.

### Affected Components

`workspaces/js-x-ray/benchmark/write-snapshot.ts` (serialization), which outputs to `workspaces/js-x-ray/benchmark/report.json`.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

The main challenge I faced was a Node.js version mismatch - the version the project requires differs from the one installed on my local system. To resolve this, and to keep the project's dependencies isolated from my local environment, I used Docker to run the project locally. Since Docker isn't referenced anywhere in the original repository, its documentation, or the contribution guidelines, I added a `Dockerfile.dev` to my fork (committed on the `add-dockerfile` branch and merged into `master`) so the setup is reproducible. It's available at `Dockerfile.dev` in the repo root.

Working branch: https://github.com/resham57/js-x-ray/tree/add-dockerfile


### Steps to Reproduce

1. Clone the forked repo and move into the project directory. For the first run, build and start the container:
  ```bash
  # Build the image
  docker build -f Dockerfile.dev -t js-x-ray-dev .
  
  # Run interactively, mounting your local repo into /app
  # (node_modules stays inside the container via the anonymous volume)
  docker run -it --rm \
    -v "$(pwd)":/app \
    -v /app/node_modules \
    js-x-ray-dev
  ```
2. Inside the container, run the following command to generate the benchmark result (currently produced in JSON format only):
```bash
node --expose-gc --experimental-strip-types ./workspaces/js-x-ray/benchmark/write-snapshot.ts
```
   
3. The command in step 2 generates a `report.json` file inside the `workspaces/js-x-ray/benchmark` directory.

**Note:** This is not a bug - the existing flow works correctly. The task is to additionally output the result from step 2 in a table format.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/resham57/js-x-ray/commit/7bb264662a3e0d56244ccb66c3f220b2804bcab5
- **Screenshots/logs:** <img width="887" height="783" alt="image" src="https://github.com/user-attachments/assets/fc588154-a72b-474a-acd8-f34ba29cdf5f" />
- **My findings:** Running the benchmark script produces a `report.json` file at `workspaces/js-x-ray/benchmark/report.json`. The data is correct and complete - per-benchmark stats (avg, min, max, percentiles, sample count, heap usage, and GC timings on heavier files) are all captured. The serialization happens in `write-snapshot.ts`, where the results object is written via `JSON.stringify(..., null, 2)`. The raw JSON is accurate but hard to scan at a glance, which confirms the issue: the same data would be far more readable rendered as a table. This is where the table-format output should be added, alongside the existing JSON (not replacing it, since CI consumes `report.json`).

---

## Solution Approach

### Analysis

This isn't a bug - it's a readability gap. The benchmark pipeline works correctly: `write-snapshot.ts` calls `benchmark()` (from `bench.ts`), reshapes the mitata results into a `relevantResults` object, and persists it with `JSON.stringify(relevantResults, null, 2)`. The "root cause" of the issue is simply that JSON is the *only* output format. The raw JSON is accurate and machine-friendly (and CI depends on it), but a human scanning it has to mentally parse nested objects and raw nanosecond/byte values to compare benchmarks. There is no human-readable rendering of the same data.

### Proposed Solution

Add a second, human-readable output **alongside** the existing JSON rather than replacing it. After the current `report.json` is written, generate a Markdown table (`report.md`) from the same `relevantResults` object. Each benchmark becomes a row, with timing values converted from nanoseconds to readable units (µs/ms) and heap from bytes to KB/MB. Keeping `report.json` byte-for-byte identical means the CI snapshot is unaffected; the table is purely additive.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:**  
The benchmark results are stored only as raw JSON in `workspaces/js-x-ray/benchmark/report.json`. The goal is to also store them in a more readable, table-like format, without changing or breaking the existing JSON output that CI relies on.

**Match:**  
The codebase already establishes the pattern to follow:
- `write-snapshot.ts` uses `new URL("report.json", import.meta.url)` + `writeFileSync` to persist output - the new file follows the same convention.
- The results object is built once and serialized; I reuse that exact object as the data source for the table, so JSON and table can never drift apart.
- mitata (the benchmarking lib) exposes `run({ format: "json" })` formatting, but this project intentionally captures the structured results object and serializes it manually - so the table generation stays in `write-snapshot.ts` for consistency, instead of relying on mitata's output formatting.


**Plan:**
1. In `write-snapshot.ts`, keep the existing `report.json` write unchanged.
2. Add helper functions `formatDuration(ns)` (ns → ns/µs/ms/s) and `formatBytes(bytes)` (B/KB/MB/GB) to make values readable.
3. Add a `toMarkdown(report)` function that emits a metadata header (timestamp, runtime, CPU) plus a table with one row per benchmark (columns: name, avg, min, max, p75, p99, samples, heap avg, gc avg - with `—` fallback where `gc`/`heap` is absent).
4. Write the result to `report.md` via `writeFileSync(new URL("report.md", import.meta.url), ...)`.
5. Update the CI workflow so the snapshot step also stages/commits `report.md` (it currently auto-commits `report.json` on each PR).
6. Update the contributing/benchmark docs to mention the new `report.md` artifact.

**Implement:** https://github.com/resham57/js-x-ray/tree/benchmark-result-table

**Review:**
- [x] `report.json` output is unchanged (no diff in the snapshot).
- [x] Code follows existing style (TS, `import.meta.url` + `writeFileSync`, runs under `--experimental-strip-types`).
- [x] No new runtime dependencies added.
- [x] Handles optional fields safely (`gc`, `heap` may be missing).
- [x] Linter and tests pass (`npm test` / project lint script).

**Evaluate:**
- Run `node --expose-gc --experimental-strip-types ./workspaces/js-x-ray/benchmark/write-snapshot.ts` inside the Docker container.
- Confirm `report.json` is identical to the pre-change version (diff check).
- Confirm `report.md` is generated and renders as a correct, readable table on GitHub.
- Verify values match between the two files (e.g. a 265622 ns avg in JSON shows as `265.62 µs` in the table).

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: formatDuration converts nanoseconds to the correct human-readable unit at each boundary (ns, µs, ms, s)
- [x] Test case 2: formatBytes converts bytes to the correct human-readable unit at each boundary (B, KB, MB, GB)
- [x] Test case 3: toMarkdown renders metadata header (timestamp, runtime, CPU)
- [x] Test case 4: toMarkdown renders table header and separator rows with all 12 columns
- [x] Test case 5: toMarkdown uses - fallback when heap and gc are absent from stats
- [x] Test case 6: toMarkdown renders formatted heap and gc values when present
- [x] Test case 7: toMarkdown renders multiple benchmark rows
- [x] Test case 8: toMarkdown output ends with a trailing newline

### Integration Tests

- [x] Integration scenario 1: Confirmed `write-snapshot.ts` correctly imports toMarkdown from the extracted `markdown.ts` module and produces both `report.json` and `report.md` from the same benchmark run
- [x] Integration scenario 2: Verified `report.md` content matches `report.json` data - spot-checked timing conversions (e.g., 233750 ns → 233.75 µs), byte conversions (e.g., 273523 bytes → 267.11 KB), and the - fallback for missing gc across all 11 benchmark rows

### Manual Testing

- Ran node --test workspaces/js-x-ray/test/markdown.spec.ts - 14/14 tests pass
- Ran existing test suite alongside new tests (warnings, Pipelines, NodeCounter, CollectableSet), no regressions
- Ran npx eslint workspaces - zero errors (after auto-fixing @stylistic/member-delimiter-style in the interface)

---

## Implementation Notes

### Week 1 Progress

**What was built:** A Markdown table report (`report.md`) generated alongside `report.json` during benchmark snapshot writes. The formatting logic was extracted into a standalone  `benchmark/markdown.ts` module with three exported functions: `formatDuration`, `formatBytes`, and `toMarkdown`. Unit tests were added covering all boundary conditions and rendering behavior.

**Challenges faced:** 
Other than the development environment setup - which was also relatively straightforward - I did not face any major challenges.

**Decisions made:**
- Defined a `BenchmarkReport` TypeScript interface to type the `toMarkdown` input, rather than relying on `typeof` against a runtime value.
- Only surfaced `heap.avg` and `gc.avg` in the Markdown table (not min/max/total) to keep the table readable. The full data remains in `report.json`.

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:**
  - `workspaces/js-x-ray/benchmark/write-snapshot.ts` - added import of toMarkdown, added report.md write call
  - `workspaces/js-x-ray/benchmark/report.json` - re-ran benchmark snapshot

- **Files created:**
  - `workspaces/js-x-ray/benchmark/markdown.ts` - extracted formatDuration, formatBytes, toMarkdown + BenchmarkReport interface
  - `workspaces/js-x-ray/benchmark/report.md` - generated Markdown table output
  - `workspaces/js-x-ray/test/markdown.spec.ts` - 14 unit tests across 3 describe blocks

- **Key commits:**
  - [5e4292a](https://github.com/resham57/js-x-ray/commit/5e4292a9848212bcef44168b1617554ba6c3d698) Updates write snapshot to support report-table feature
  - [16e58ef](https://github.com/resham57/js-x-ray/commit/16e58efcbe8d55e43d750bbe9190d9f358a62e97) Updates existing json report and adds new table report
  - [f7d69e9](https://github.com/resham57/js-x-ray/commit/f7d69e9c8f570541d33e48effafd44fcb0fdcea3) refactor(reports): move markdown generation to separate ts file
  - [aac0d5c](https://github.com/resham57/js-x-ray/commit/aac0d5c57b6fe07b222baf00e6a1dc14f79a518c) test: add markdown generation specs
  - [a062fbd](https://github.com/resham57/js-x-ray/commit/a062fbdcb99d6c1d0899fea55037e6c3095bb14f) test: add three test cases for markdown generation

- **Approach decisions:** Chose module extraction over in-file exports to avoid side-effect imports in tests. Used `describe/test` grouping (matching the project's `Pipelines.spec.ts` pattern) since the spec covers three distinct functions.

---

## Pull Request

**PR Link:** https://github.com/NodeSecure/js-x-ray/pull/654

**PR Description:**
- Add new benchmark formatting logic (`formatDuration`, `formatBytes`, `toMarkdown`) to create human-readable report `report.md` to an existing `benchmark/write-snapshot.ts` module
- Extract benchmark formatting logic (`formatDuration`, `formatBytes`, `toMarkdown`) into a new `benchmark/markdown.ts` module
- Update `write-snapshot.ts` to import from `markdown.ts` and write a human-readable `report.md` alongside the existing `report.json`
- Re-run benchmark snapshot (`report.json` + new `report.md`)
- Add unit tests for all three exported functions covering unit-boundary conversions, the `—` fallback when heap/gc data is absent, and multi-row rendering


**Maintainer Feedback:**
- [06/22/2026]: The maintainer liked the implementation and felt it was an improvement over the current JSON output. They asked me to (1) remove the existing JSON reporting logic and the `report.json` file, (2) unexport the helper functions (`formatDuration`, `formatBytes`) since they're only used internally, and (3) remove the now-redundant tests for those internal helpers. They also requested a changeset.

- [06/22/2026]: Addressed all feedback — removed the JSON report generation code and `report.json`, unexported the internal helpers (keeping only `toMarkdown` public), removed the corresponding tests, and added a changeset.

**Status:** Merged

---

## Learnings & Reflections

### Technical Skills Gained

- **Working in a TypeScript monorepo.** Got comfortable with npm workspaces - locating the right package (`workspaces/js-x-ray`), running scripts from the correct directory, and understanding how a workspace's tooling is separate from the repo root.
- **Running TypeScript without a build step.** Learned how Node executes `.ts` directly via `--experimental-strip-types`, why the project sticks to type-only TypeScript, and how flags like `--expose-gc` feed the benchmark tooling (mitata) for more stable measurements.
- **Benchmark tooling and data shaping.** Understood how mitata reports stats (nanoseconds for timing, bytes for heap) and wrote formatting logic to convert those into human-readable units and a Markdown table.
- **Testing with `node:test`.** Wrote deterministic unit tests for pure functions using the built-in test runner and `node:assert`, and learned why benchmark output itself (non-deterministic timings) shouldn't be asserted on.
- **Reading coverage reports.** Learned to interpret statement/branch/function/line coverage and understand why removing tests for internal helpers lowered branch coverage.
- **Project conventions.** Conventional Commits (correct type/scope, breaking-change footers), changesets for versioning/changelog automation, and running `npm run check` (tests + lint) before pushing.
- **Containerizing a dev environment** with a `Dockerfile.dev` and a bind mount, so the project's required Node version stayed isolated from my local machine.

### Challenges Overcome

- **Node version mismatch.** My local Node version didn't match the project's. I solved it with Docker - a `Dockerfile.dev` plus a bind mount so generated files appeared straight on my host without polluting my local environment.
- **Lint failures.** Hit `@stylistic/no-mixed-operators` on `bytes / 1_024 ** 3`. Learned it was a clarity rule (not a correctness bug) and resolved it by extracting named constants so each expression used a single operator.
- **Scoping the change correctly.** My first instinct was to treat this as a one-off script change, but I learned to keep the JSON output stable initially, design the table as additive, and only remove the JSON once the maintainer explicitly asked - and to flag the CI risk of that removal.
- **Responding to review.** Iterated on maintainer feedback: unexported internal helpers to keep the module's public API minimal, removed the now-redundant tests, and added a changeset.

### What I'd Do Differently Next Time

- **Check project conventions up front.** I corrected commit types/scopes several times (`refactor` vs `feat` vs `test`, wrong scope). Next time I'll read `CONTRIBUTING.md` and skim recent commit history *before* committing, so I get the format right the first time.
- **Add the changeset as part of the feature commit,** rather than as a follow-up after the maintainer asked - it's a standard expectation I now know to anticipate.

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
