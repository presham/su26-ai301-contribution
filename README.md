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

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
