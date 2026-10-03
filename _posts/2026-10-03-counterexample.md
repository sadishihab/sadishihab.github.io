---
title: "100% Mutation Score, Still Wrong: Building a PR Verifier on IBM Bob 2.0"
date: 2026-10-03
layout: post
permalink: /counterexample/
categories: [AI, Python, DevOps, Testing]
tags: [AI Agents, IBM Bob, Mutation Testing, Code Review, Python, Pytest, GitHub Actions, CI/CD, Developer Tools, LLM]
description: "How I built Counterexample, a pull-request verifier that combines diff-scoped mutation testing with IBM Bob 2.0 subagents that try to break each claim a PR makes. On a real PR, mutation testing scored 100% and the code was still wrong."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG Application Developer*  
*October 03, 2026*  

<br>

### **Summary**

Here is a pull request with a green CI badge, a passing test suite, and a 100% mutation score. By every automated measure, it is safe to merge.

It is wrong in three different ways.

**Counterexample** is the tool I built to close that gap. It reviews a pull request with two independent layers of evidence. The first is **diff-scoped mutation testing**: inject small bugs into only the lines the PR changed, and check whether the PR's own tests notice. The second is **claim falsification on IBM Bob 2.0**: extract the concrete claims the PR makes from its diff and linked issue, then spawn one subagent per claim, each writing and actually running a test designed to break it.

On that pull request, mutation testing alone scored 100%. Claim falsification came back with **4 of 5 claims falsified**, each backed by a failing test that really ran.

This post covers how both layers work, why mutation testing missed every bug (the reason is more interesting than "it's weak"), what happened when the same review ran twice and disagreed with itself, and the three bugs I found in my own tool, one of which was quietly corrupting the repository it was meant to verify.  
<br>
🔗 **GitHub Repository:** [sadishihab/counterexample](https://github.com/sadishihab/counterexample)  
🔗 **The reviewed pull request:** [counterexample-demo-checkout PR #1](https://github.com/sadishihab/counterexample-demo-checkout/pull/1)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. The Problem: Green CI Is a Weak Signal](#1-the-problem-green-ci-is-a-weak-signal)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. Layer One: Diff-Scoped Mutation Testing](#3-layer-one-diff-scoped-mutation-testing)
- [4. Layer Two: Claim Falsification on IBM Bob 2.0](#4-layer-two-claim-falsification-on-ibm-bob-20)
- [5. The Test PR: A Bug Hiding Behind Green Tests](#5-the-test-pr-a-bug-hiding-behind-green-tests)
- [6. The Result: 100% Against 4 of 5](#6-the-result-100-against-4-of-5)
- [7. Same Evidence, Different Verdict](#7-same-evidence-different-verdict)
- [8. Three Bugs in My Own Tool](#8-three-bugs-in-my-own-tool)
- [9. CI That Says What It Does Not Check](#9-ci-that-says-what-it-does-not-check)
- [10. How I Worked: Four Surfaces](#10-how-i-worked-four-surfaces)
- [11. Limits and What Comes Next](#11-limits-and-what-comes-next)
- [12. Learning Outcomes](#12-learning-outcomes)
- [References](#references)
- [License](#license)
<br>

### **Key Technologies Used**

`Python 3.11` · `ast` · `git diff` · `pytest` · `concurrent.futures` · `IBM Bob 2.0 (custom modes, skills, subagents)` · `GitHub Actions` · `actions/github-script`  
<br>

### **1. The Problem: Green CI Is a Weak Signal**

AI assistants now write pull requests faster than teams can review them. When IBM launched Bob 2.0, it cited research showing that 85% of DevSecOps professionals agree AI has shifted the bottleneck from writing code to reviewing and validating it.

Tests are supposed to carry that weight, but there is a structural problem. When the same reasoning writes the implementation and the tests, the tests tend to describe the code rather than the requirement. A wrong formula and a test that asserts the wrong result will pass together, every time.

Reading every line does not scale either. So the design rule from day one was: **evidence over opinions.** Every finding must be backed by a test that actually ran. "This looks risky" is not a finding.

That splits into two questions, and each gets its own layer:

- **Would the PR's own tests notice if the changed code were broken?** That is mutation testing.
- **Does the PR actually do what its linked issue says?** That is claim falsification.

They sound similar. They are not, and the test PR in this post shows exactly why you need both.  
<br>

#### **Architecture Diagram**
```text
                    pull request (diff + linked issue)
                                   │
          ┌────────────────────────┴────────────────────────┐
          ▼                                                 ▼
  Layer 1: mutation testing                     Layer 2: claim falsification
  (deterministic engine, runs in CI)            (IBM Bob 2.0, Counterexample mode)
          │                                                 │
  diff.py    changed line ranges                extract-claims  →  5 claims
  mutate.py  AST mutants on those lines only    5 parallel subagents (falsify-claim)
  runner.py  pytest per mutant, isolated copy   each writes and runs one adversarial test
          │                                                 │
  killed / survived / error                     FALSIFIED / HELD + real test output
          └────────────────────────┬────────────────────────┘
                                   ▼
                     receipt.py → one Review Receipt
             BUGS FOUND if any claim is falsified or score < 80%
```
<br>

### **2. Folder Structure**
```bash
counterexample/
├── engine/
│   ├── diff.py        # git diff → changed line ranges per .py file
│   ├── mutate.py      # AST mutants, restricted to changed lines
│   ├── runner.py      # pytest per mutant, isolated temp copies, in parallel
│   ├── receipt.py     # self-contained HTML Review Receipt
│   └── cli.py         # counterexample review --repo --base --head [--claims] [--out]
├── .bob/              # source drafts for the Bob mode, skills and workflow
├── tests/             # 15 tests, several end-to-end against a real repo
├── bob_sessions/      # Bob task session summaries
└── AGENTS.md          # rules for every agent that touches the repo

counterexample-demo-checkout/
├── checkout/pricing.py        # the service under review
├── docs/ISSUE-42.md           # the requirement the PR claims to implement
├── .github/workflows/         # runs Counterexample on every pull request
└── review-evidence/           # Bob's adversarial tests, claims JSON, receipts
```
<br>

### **3. Layer One: Diff-Scoped Mutation Testing**

Mutation testing asks a blunt question: if this code were broken, would anyone notice? It injects a small bug (a *mutant*), reruns the tests, and records the outcome. If a test fails, the mutant is **killed**. If everything still passes, it **survived**, and that is a gap in the tests.

Classic mutation testing is slow because it mutates the whole codebase. A reviewer does not need that. The question is about *this* pull request, so Counterexample only mutates the lines the PR changed.

**Step 1: find the changed lines.** A zero-context `git diff` gives exact hunk headers, and a single regex turns them into line ranges. No third-party diff parser.

```python
# engine/diff.py (simplified)
_HUNK_HEADER = re.compile(r"^@@ -\d+(?:,\d+)? \+(\d+)(?:,(\d+))? @@")

def changed_line_ranges(repo_path, base, head):
    diff = subprocess.run(
        ["git", "diff", "--unified=0", f"{base}..{head}"],
        cwd=repo_path, check=True, capture_output=True, text=True,
    ).stdout
    # track the current file from "+++ b/..." lines, then for each hunk:
    #   start = int(match.group(1)); count = int(match.group(2) or 1)
    #   record (start, start + count - 1) unless count == 0 (pure deletion)
```

**Step 2: mutate the AST, only inside those ranges.** Four operator families, applied one at a time:

- comparison flips: `>` ↔ `>=`, `<` ↔ `<=`, `==` ↔ `!=`
- boolean swaps: `and` ↔ `or`
- arithmetic swaps: `+` ↔ `-`, `*` ↔ `/`
- return-value tweaks: `return x` → `return None`

For each match, the tool deep-copies the tree, changes exactly one node, and regenerates source with `ast.unparse()`.

```python
# engine/mutate.py (simplified)
for index, node in enumerate(ast.walk(tree)):
    lineno = getattr(node, "lineno", None)
    if lineno is None or not in_changed_ranges(lineno, changed_ranges):
        continue                     # never touch lines the PR didn't change
    if isinstance(node, ast.Compare):
        mutants.extend(compare_flips(tree, index, node))
    elif isinstance(node, ast.BinOp):
        mutants.append(arith_swap(tree, index, node))
    # ... BoolOp and Return handled the same way
```

**Step 3: run every mutant in isolation, in parallel.** Each mutant gets its own temp copy of the repository, so four workers can run at once without touching each other or the real files.

```python
# engine/runner.py (simplified)
def _run_single_mutant(repo_path, mutant):
    tmp_dir = tempfile.mkdtemp(prefix="counterexample-mutant-")
    try:
        dest = Path(tmp_dir) / "repo"
        shutil.copytree(repo_path, dest, ignore=shutil.ignore_patterns(".git"))
        (dest / mutant.file_path).write_text(mutant.mutated_source)
        proc = subprocess.run(
            [sys.executable, "-m", "pytest", "-q", "tests"],
            cwd=dest, capture_output=True, text=True, timeout=30, check=False,
        )
        ...  # 0 → survived, 1 → killed, anything else → error
    finally:
        shutil.rmtree(tmp_dir, ignore_errors=True)
```

The score is `killed / (killed + survived)`. Errors are excluded from the score and listed separately, which turned out to matter a great deal (see section 8).  
<br>

### **4. Layer Two: Claim Falsification on IBM Bob 2.0**

IBM Bob is an AI development partner that works with full repository context. Version 2.0 added **subagents**: focused workers that run in their own context windows, in parallel, each approved before it starts. That is exactly the shape this problem needs. Counterexample uses three Bob features.

**A custom mode.** The `Counterexample` mode has one job, stated in its role definition:

> You are Counterexample, a PR verification agent. Your only job is to find evidence that a pull request is wrong — never to praise it or summarize it.

Its tool access is scoped to the job: Read, Edit, Execute, Skill, Subtask and Subagent are on; Browser, MCP and mode-switching are off. Scoping tools is the cheapest guardrail there is. A mode that cannot browse cannot wander off.

**Two custom skills.** Skills are reusable instruction sets, which makes the method versioned and repeatable rather than living in a prompt I retype.

- **`extract-claims`** reads the diff and the linked issue and outputs a JSON list of *falsifiable* claims, each traced to its source requirement. "Handles coupons correctly" is not a claim. "The order total never goes below 0.00" is. Requirements the diff does not appear to implement are flagged instead of silently dropped. The skill is forbidden from writing tests.
- **`falsify-claim`** takes exactly one claim, writes one pytest test aimed at its most likely failure (boundaries, zero, combined inputs, ordering), **runs it**, and reports one of two verdicts:

```json
{"claim_id": "claim-2", "verdict": "FALSIFIED", "test_code": "...", "result": "..."}
```

`HELD` comes with a required caveat: the test found no counterexample, which is not proof the claim is true. The skill may write test code only. It is never allowed to edit the PR's source.

**Parallel subagents.** One task prompt in Counterexample mode is enough. Bob extracted five claims, then spawned five subagents, one per claim, each in its own context, each writing and running its own adversarial test. The first full review cost **1.71 Bobcoins** at about 25.8k tokens of context.  
<br>

### **5. The Test PR: A Bug Hiding Behind Green Tests**

To evaluate this properly I needed a pull request where I knew the truth. I built a small checkout pricing service and a PR modelled on what AI-generated changes often look like: plausible code, tidy docstrings, passing tests.

The linked issue, `ISSUE-42`, asks for stacked coupons:

1. At most one fixed coupon and at most one percent coupon together
2. The fixed coupon is applied first; the percent applies to **what remains** after it
3. The order total must never go below 0.00
4. Two coupons of the same kind raise `ValueError`

The PR implements it with two planted defects: the percent discount is computed from the original subtotal, and the zero floor from the old implementation disappears in the refactor.

```python
total = subtotal
if fixed_coupons:
    total -= fixed_coupons[0].value
if percent_coupons:
    total -= subtotal * percent_coupons[0].value / Decimal("100")   # should be total
return total.quantize(CENTS, rounding=ROUND_HALF_UP)               # max(total, ZERO) is gone
```

Its test suite looks thorough and every test passes. Here is the important one:

```python
def test_calculate_total_stacks_fixed_and_percent_coupons() -> None:
    items = [LineItem(name="widget", unit_price=Decimal("200.00"), qty=1)]
    fixed = Coupon(code="SAVE10", kind="fixed", value=Decimal("10.00"))
    percent = Coupon(code="TENPCT", kind="percent", value=Decimal("10"))
    assert calculate_total(items, [fixed, percent]) == Decimal("170.00")
```

Per the issue: 200 minus 10 is 190, and 10% of 190 is 19, so the total should be **171.00**. The test asserts **170.00**, the buggy answer. It is not testing the requirement. It is testing the implementation.  
<br>

### **6. The Result: 100% Against 4 of 5**

**Mutation testing:** 13 mutants on the changed lines, every one of them killed. Score: **100%**. The CI comment on the PR said ✅ LOOKS SOLID.

That result is correct, and it is still useless here, for two structural reasons:

1. **Mutation testing measures sensitivity, not correctness.** A mutant is killed when the tests notice that behaviour *changed*. The stacking test asserts 170.00, so flipping a `-` to a `+` changes the result and fails the test. The suite is perfectly sensitive to deviations from the wrong answer.
2. **It can only mutate code that exists.** The missing zero floor is a deleted line, so there is nothing to mutate. The wrong-variable bug is a name substitution (`subtotal` where `total` belongs), and none of the four operator families produce one.

**Claim falsification:**

| Claim | Verdict | Evidence |
|---|---|---|
| Percent applies to the post-fixed remainder | ✗ FALSIFIED | subtotal 200, fixed 10, percent 10% → 170.00, spec requires 171.00 |
| Total never goes below 0.00 | ✗ FALSIFIED | subtotal 10, fixed 50 → **−40.00** |
| Duplicate coupon kind raises `ValueError` | ✓ HELD | 5 of 5 adversarial tests passed |
| Result is independent of coupon order | ✗ FALSIFIED | both orderings return 170.00, spec requires 171.00 |
| Old single-coupon calls still work | ✗ FALSIFIED | `TypeError: 'Coupon' object is not iterable` |

Claims 1 and 2 are the two defects I planted. Claim 4 is claim 1 seen from another angle.

Claim 5 I did not plant. The refactor changed the parameter from a single `coupon` to a list of `coupons`, and the guard `coupons = coupons or []` lets a truthy bare `Coupon` pass straight through to a list comprehension that tries to iterate it. Every existing caller of the old API crashes. Nothing in the test suite calls it the old way, so CI could never see it.

Both layers land in one Review Receipt: the **BUGS FOUND** banner at the top, the five claims with their evidence, and underneath them a mutation score of 100% with "no surviving mutants". That contrast, on one page, is the whole argument for running both layers.  
<br>

### **7. Same Evidence, Different Verdict**

I ran the full review twice, independently. The claims came out essentially the same, and the adversarial tests reached the same numbers. One verdict did not.

- **First run:** claim 4 **HELD**, with a note that it "holds for the wrong reason", since both orderings agree with each other only because both use the wrong base.
- **Second run:** claim 4 **FALSIFIED**, on the grounds that order-independence around an incorrect value should not count as the claim holding.

Both readings are defensible. Does "independent of coupon order" mean the two orderings agree with each other, or that each ordering produces the correct value? The test output was identical. The interpretation of a natural-language claim was not.

Three things follow from that:

- **Keep the evidence next to the verdict.** The receipt shows the inputs, the expected value and the actual value for every claim, so a human can see why, not just what.
- **Pick one result and make every artifact agree.** I published the stricter reading and updated the claims file, the markdown receipt and the HTML receipt in a single commit, so nothing in the repository contradicts the rest.
- **Precise claims produce stable verdicts.** "For subtotal 200, fixed 10 and percent 10%, both orderings return 171.00" leaves nothing to interpret. Tightening claims at extraction time is the real fix.  
<br>

### **8. Three Bugs in My Own Tool**

Building a verifier is a humbling way to learn that you ship bugs too. These three were all found by evidence, not by suspicion.

**❌ The mutation runner was corrupting the repository it verified**

The demo repo's working tree started showing changes nobody had made. The source and its test had been reformatted: single quotes everywhere, blank lines removed, no newline at the end of the file. And `==` had been flipped to `!=` in both.

That formatting is exactly what `ast.unparse()` produces, and `==` → `!=` is one of my own operators. It looked like a mutant because it was one. But another agentic tool had the same workspace open at the time, which made for a more exciting explanation, and I spent a while on that theory before the evidence won. The real cause was one line of `pathlib` behaviour:

```python
>>> from pathlib import Path
>>> Path("/tmp/counterexample-mutant-x/repo") / "/home/sadi/projects/demo/checkout/pricing.py"
PosixPath('/home/sadi/projects/demo/checkout/pricing.py')
```

When the right-hand side is absolute, `pathlib` discards the left side. The CLI passed absolute paths to the mutator, the mutator stored them on each mutant, and the runner joined them onto its temp copy. Four threads were writing mutants straight onto the real file, and whichever finished last stayed on disk. Worse, an earlier test that asserted a mix of killed and survived mutants had been passing *because* of the race.

The fix keeps mutant paths repo-relative, and the runner now refuses anything else:

```python
if Path(mutant.file_path).is_absolute():
    raise ValueError(f"Mutant.file_path must be repo-relative, got absolute path: {mutant.file_path}")
```

A write that can escape its sandbox should fail loudly, never succeed somewhere else. And when the evidence points at your own code, believe it before reaching for a better story.

**❌ A stale working tree produced a 0% score**

The CLI diffed two git refs but read the files from disk. With `main` checked out and `--head feature/stacked-coupons`, the feature branch's line numbers were applied to the old file, producing nonsense mutants and a 0% score. The fix: `review` now checks out `--head` itself, and fails loudly on a dirty tree or a missing ref instead of carrying on.

**❌ CI reported 0% for a reason the receipt could not show**

The first real CI run reported 13 mutants and a 0% score. Every local run reproducibly scored 100%. The receipt counted errors but did not display them, and the CI's Python 3.11 was not available on my machine.

Instead of guessing, I changed the receipt to show the detail for every errored mutant, pushed, and read the next artifact:

```text
review-evidence/test_claim5_backward_compat.py:26: in <module>
    from pricing import LineItem, Coupon, calculate_total, calculate_subtotal
E   ModuleNotFoundError: No module named 'pricing'
```

GitHub's `pull_request` event checks out a merge commit of the PR into its base branch, which pulled `review-evidence/` in from `main`. The runner invoked a bare `pytest -q`, which collects any `test_*.py` anywhere in the tree, so every mutant failed at collection in exactly the same way. The fix was one word: `pytest -q tests`. The next CI run scored 100% with no errors section.

Finding it needed visibility. Fixing it needed one line. When a metric can be wrong for structural reasons, make the failure details visible first.  
<br>

### **9. CI That Says What It Does Not Check**

A GitHub Action runs the mutation layer on every pull request. It checks out the full history (`fetch-depth: 0`, so the diff against `origin/main` resolves), installs Counterexample straight from GitHub, runs the review, posts a summary comment, and uploads the HTML receipt as an artifact.

Once the CI bug was fixed, a different problem appeared. The Action posted ✅ LOOKS SOLID on a PR that I knew had four falsified claims. The claim layer runs on demand in Bob IDE, so CI only ever sees the mutation layer.

Rather than let a green check imply more than it checked, the comment now says so:

> *This is automated mutation testing only. It checks whether the PR's own tests would catch an injected bug — it does not verify the PR's logic against its stated requirements.*

A green check that tells you what it did not check is worth more than one that implies it checked everything.  
<br>

### **10. How I Worked: Four Surfaces**

I built this with AI assistance, with the same discipline I would expect from a team:

1. **Planning chat:** design, review, and breaking work into small, checkable steps
2. **Claude Code in the repo:** the engine, tests, CI and commits, bound by rules in `AGENTS.md` (type hints everywhere, stdlib first, every module tested, mutations only on changed lines)
3. **IBM Bob IDE:** the Counterexample mode, the two skills, and the reviews themselves
4. **A plain shell:** independent verification of everything the assistants claim, with `git status`, `git log` and `git diff`

Every commit message records which tool did the work, `(Claude Code)` or `(IBM Bob)`, and every Bob task session summary is saved in `bob_sessions/`. Verification included installing the package fresh from GitHub into a clean virtual environment and running it against a fresh clone, because "it works on my checkout" is not the same claim as "it installs and works".

The plain shell is what caught the corrupted working tree in section 8. A single `git diff` showed the `ast.unparse()` fingerprint.  
<br>

### **11. Limits and What Comes Next**

Being honest about the edges is part of the method:

- **Python and pytest only**, with four operator families. Name substitutions and deleted code are outside what mutation can see, which is exactly why the second layer exists.
- **The claim layer runs in Bob IDE, not in CI.** Bob Shell has a non-interactive mode built for automation, which is the obvious next step for running claim falsification on every PR. I have not tested that yet.
- **Bob's verdicts are transcribed into a claims file** that the receipt generator reads. Automating that hand-off is next.
- **Loosely worded claims produce unstable verdicts** (section 7). Tighter extraction is the fix.
- **One PR proves the approach works. It is not a benchmark.** Running it against real open-source pull requests is the next real test.  
<br>

### **12. Learning Outcomes**

- Built **diff-scoped mutation testing** from the AST up, mutating only the lines a pull request changed  
- Learned that **mutation testing measures sensitivity, not correctness**: tests that assert a wrong value still kill every mutant  
- Learned that **mutation cannot see deleted code or name substitutions**, which defines where a second layer is needed  
- Designed **agentic claim falsification** on IBM Bob 2.0 with a scoped custom mode, two versioned skills and parallel subagents  
- Kept every verdict **backed by an executed test**, with `HELD` explicitly meaning "no counterexample found", not "correct"  
- Saw **LLM verdicts vary between runs** on identical evidence, and learned that precise claims are the fix  
- Found a **pathlib absolute-path join** that let concurrent writes escape their sandbox, and made it fail loudly  
- Debugged CI by **adding visibility first**, then fixing the one-word cause in pytest collection scope  
- Made CI **state its own scope** instead of implying a green check covers everything  
- Practised **AI-assisted engineering with guardrails**: plan first, verify independently, record which tool did what  

<br>

### **Try It**

The code, the demo pull request, and Bob's full review evidence (the adversarial tests, the claims file and both receipts) are all public:

- [Counterexample on GitHub](https://github.com/sadishihab/counterexample)
- [The reviewed pull request](https://github.com/sadishihab/counterexample-demo-checkout/pull/1)
- [Review evidence](https://github.com/sadishihab/counterexample-demo-checkout/tree/main/review-evidence)

If your team is shipping AI-generated code and wants this kind of verification in its own pipeline, or you are building AI agents that need to be right rather than merely confident, I am happy to talk: [book a 30 minute call](https://calendly.com/sadi-shihab/30min) or find me on [LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi).

<br>

### **References**
- [Counterexample on GitHub](https://github.com/sadishihab/counterexample)
- [Demo repository and review evidence](https://github.com/sadishihab/counterexample-demo-checkout)
- [IBM Bob](https://bob.ibm.com)
- [Mutation testing (Wikipedia)](https://en.wikipedia.org/wiki/Mutation_testing)
- [Python `ast` module](https://docs.python.org/3/library/ast.html)
- [Python `pathlib` module](https://docs.python.org/3/library/pathlib.html)
- [GitHub Actions documentation](https://docs.github.com/en/actions)
<br>

### **License**
This project is open-source and available under the MIT License
