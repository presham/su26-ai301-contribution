# su26-ai301-contribution
# Contribution 1: (Benchmark) Store the benchmark result in a more readable format

**Contribution Number:** 1  
**Student:** Resham Pokhrel  
**Issue:** https://github.com/NodeSecure/js-x-ray/issues/612  
**Status:** Phase I Complete 

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

Running the benchmark produces only `report.json` — accurate but not human-friendly.

### Affected Components

`workspaces/js-x-ray/benchmark/write-snapshot.ts` (serialization), which outputs to `workspaces/js-x-ray/benchmark/report.json`.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

The main challenge I faced was a Node.js version mismatch — the version the project requires differs from the one installed on my local system. To resolve this, and to keep the project's dependencies isolated from my local environment, I used Docker to run the project locally. Since Docker isn't referenced anywhere in the original repository, its documentation, or the contribution guidelines, I added a `Dockerfile.dev` to my fork (committed on the `add-dockerfile` branch and merged into `master`) so the setup is reproducible. It's available at `Dockerfile.dev` in the repo root.

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

**Note:** This is not a bug — the existing flow works correctly. The task is to additionally output the result from step 2 in a table format.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/resham57/js-x-ray/commit/7bb264662a3e0d56244ccb66c3f220b2804bcab5
- **Screenshots/logs:** <img width="887" height="783" alt="image" src="https://github.com/user-attachments/assets/fc588154-a72b-474a-acd8-f34ba29cdf5f" />
- **My findings:** Running the benchmark script produces a `report.json` file at `workspaces/js-x-ray/benchmark/report.json`. The data is correct and complete — per-benchmark stats (avg, min, max, percentiles, sample count, heap usage, and GC timings on heavier files) are all captured. The serialization happens in `write-snapshot.ts`, where the results object is written via `JSON.stringify(..., null, 2)`. The raw JSON is accurate but hard to scan at a glance, which confirms the issue: the same data would be far more readable rendered as a table. This is where the table-format output should be added, alongside the existing JSON (not replacing it, since CI consumes `report.json`).

---

## Solution Approach

### Analysis

This isn't a bug — it's a readability gap. The benchmark pipeline works correctly: `write-snapshot.ts` calls `benchmark()` (from `bench.ts`), reshapes the mitata results into a `relevantResults` object, and persists it with `JSON.stringify(relevantResults, null, 2)`. The "root cause" of the issue is simply that JSON is the *only* output format. The raw JSON is accurate and machine-friendly (and CI depends on it), but a human scanning it has to mentally parse nested objects and raw nanosecond/byte values to compare benchmarks. There is no human-readable rendering of the same data.

### Proposed Solution

Add a second, human-readable output **alongside** the existing JSON rather than replacing it. After the current `report.json` is written, generate a Markdown table (`report.md`) from the same `relevantResults` object. Each benchmark becomes a row, with timing values converted from nanoseconds to readable units (µs/ms) and heap from bytes to KB/MB. Keeping `report.json` byte-for-byte identical means the CI snapshot is unaffected; the table is purely additive.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:**  
The benchmark results are stored only as raw JSON in `workspaces/js-x-ray/benchmark/report.json`. The goal is to also store them in a more readable, table-like format, without changing or breaking the existing JSON output that CI relies on.

**Match:**  
The codebase already establishes the pattern to follow:
- `write-snapshot.ts` uses `new URL("report.json", import.meta.url)` + `writeFileSync` to persist output — the new file follows the same convention.
- The results object is built once and serialized; I reuse that exact object as the data source for the table, so JSON and table can never drift apart.
- mitata (the benchmarking lib) exposes `run({ format: "json" })` formatting, but this project intentionally captures the structured results object and serializes it manually — so the table generation stays in `write-snapshot.ts` for consistency, instead of relying on mitata's output formatting.


**Plan:** [Step-by-step implementation plan]
1. In `write-snapshot.ts`, keep the existing `report.json` write unchanged.
2. Add helper functions `formatDuration(ns)` (ns → ns/µs/ms/s) and `formatBytes(bytes)` (B/KB/MB/GB) to make values readable.
3. Add a `toMarkdown(report)` function that emits a metadata header (timestamp, runtime, CPU) plus a table with one row per benchmark (columns: name, avg, min, max, p75, p99, samples, heap avg, gc avg — with `—` fallback where `gc`/`heap` is absent).
4. Write the result to `report.md` via `writeFileSync(new URL("report.md", import.meta.url), ...)`.
5. Update the CI workflow so the snapshot step also stages/commits `report.md` (it currently auto-commits `report.json` on each PR).
6. Update the contributing/benchmark docs to mention the new `report.md` artifact.

**Implement:** https://github.com/resham57/js-x-ray/tree/benchmark-result-table

**Review:**
- [ ] `report.json` output is unchanged (no diff in the snapshot).
- [ ] Code follows existing style (TS, `import.meta.url` + `writeFileSync`, runs under `--experimental-strip-types`).
- [ ] No new runtime dependencies added.
- [ ] Handles optional fields safely (`gc`, `heap` may be missing).
- [ ] Linter and tests pass (`npm test` / project lint script).

**Evaluate:**
- Run `node --expose-gc --experimental-strip-types ./workspaces/js-x-ray/benchmark/write-snapshot.ts` inside the Docker container.
- Confirm `report.json` is identical to the pre-change version (diff check).
- Confirm `report.md` is generated and renders as a correct, readable table on GitHub.
- Verify values match between the two files (e.g. a 265622 ns avg in JSON shows as `265.62 µs` in the table).

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
