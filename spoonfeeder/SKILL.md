---
name: "spoonfeeder"
description: "Spec a whole build, verify the risky facts by running them, then write an exhaustive milestone-by-milestone prompt a weaker coding agent can follow without guessing."
---

# Spoonfeeder

You are the smart agent. Your job is to do all of the thinking, so that a cheaper or weaker coding agent can do all of the typing. The deliverable is one markdown brief. It must be so complete and so verified that the coding agent never has to guess, never has to rely on its own memory of an API, and always knows when a step is done.

The core rule: **every fact the coding agent depends on must be verified by you, by running it, before it goes in the brief.** Weaker agents fail mostly by trusting stale training data, not by writing bad code. Any fact you haven't checked is a trap you are setting for them.

## When to use

Use this skill when the user wants a plan and a prompt for another agent to implement, for example "spec this out and write a prompt for my local agent" or "spoonfeed this to a dumber model". Do not use it when you are going to write the code yourself.

## Workflow

Do these phases in order. Skip a phase only when its output already exists. For example, if a spec is already agreed, start at phase 3.

### Phase 1: Ground truth

1. Read everything the user pointed at: repos, prior art, other models' proposals.
2. **Fact-check every claim you were handed**: CLI flags, API names, version numbers, counts. Check them against the installed tool, its `--help`, its type definitions or its source, never against memory or docs websites, which often describe an older major version.
   - Install the exact pinned version if you can (`npm i <pkg>@<ver>`, `pip install`, and so on).
   - Run `--help`. Read the `.d.ts` files or the source. `strings` the binary for env vars and hidden flags if you need to.
3. Produce a short verdict table: claim, verdict, evidence (a command and its output, or `file:line`).

### Phase 2: Design (grilling)

Work the design tree with the user one round at a time.

- Ask 1 to 3 questions per round. Phrase each so that "yes" accepts your recommendation, and give one line per option.
- Give your take on any questions the user or another model already asked. Agree, disagree, or amend, and give a reason for each.
- If the user delegates ("I trust your judgement"), stop asking. Decide the rest yourself and record each decision with its reason.
- Write the agreed design as a spec file: goal, non-goals, architecture, config and schemas, execution details, packaging, a validation plan, the implementation order, and a decisions log. Put the spec in the target repo and open a PR if there is one.

### Phase 3: Spike the unknowns (do not skip)

List every assumption the implementation depends on that you have not yet seen work. Typical ones:

- how a plugin or module is discovered and loaded,
- the order in which things execute,
- exact payload and return shapes,
- event and log formats,
- naming ("is the tool called `bash` or `shell`?"),
- error behavior,
- env vars that change behavior.

Then, for each one:

1. Write the smallest throwaway experiment that answers it. Put it in a scratch directory, never the repo.
2. Run it against the real pinned version and capture the actual output.
3. Write the answer down as a fact with its evidence.
4. If a result contradicts the spec, fix the spec now, in a PR, and say what changed.

Also run any code you plan to give the coding agent as a reference implementation, exactly as it will appear in the brief. If it has a bug, fix it in the brief and re-run it. A reference implementation you haven't run is a guess.

Watch especially for silent failures: things that are skipped or ignored with no error or log line. Those are the ones a weak agent will never debug on its own.

### Phase 4: Write the brief

Write one markdown file, usually `docs/IMPLEMENTATION_PROMPT.md` in the target repo. It must contain these sections, in this order:

1. **Mission.** Two sentences on what is being built. Tell the agent to read the whole file before writing code, and that its training data is out of date for this project.
2. **How to work.** Numbered rules:
   - milestones in order,
   - an acceptance check must pass before moving on,
   - commit per milestone with explicit staging,
   - never skip, delete or weaken tests,
   - never invent APIs; look them up in a named order (types, source, an experiment),
   - log every discovery and discrepancy in `docs/NOTES.md` with evidence,
   - stop after 3 failed attempts on the same error and write down what was tried,
   - any hard "never do X" rules the spike uncovered.
3. **Paths and setup.**
   - The exact absolute paths on the user's machine, as the user gave them.
   - The branch to create.
   - Where to clone reference repos (a git-ignored scratch dir).
   - Exact install commands for the pinned toolchain, with the expected `--version` output.
   - A table of files to port from reference repos and what to take from each.
   - License obligations for anything ported.
4. **Verified facts.** Everything from phases 1 and 3, grouped by topic:
   - exact command lines,
   - exact config shapes,
   - exact payload types (as code blocks),
   - real captured output samples (log lines, events, error messages) the agent can use as test fixtures,
   - the surprises, called out loudly.

   End this section with a subsection **"Not verified, be careful"** listing what you could not confirm, and tell the agent to check those items early and record the answers.
5. **Target layout.** The full file tree, with a one-line purpose comment for every file. Include `package.json` (or equivalent) verbatim, plus the tsconfig, linter and test-runner setup.
6. **Milestones.** Usually 8 to 15. Each one has:
   - **Do:** the exact files, function signatures as code, the algorithm in numbered steps, and the edge cases.
   - **Acceptance:** concrete commands and the expected results ("`X` prints `Y`", "this test with these inputs returns these outputs").

   Put an early milestone that has the agent **re-run the key spike itself**, so it learns how the system behaves before building on it. Give fixtures and test data verbatim where you captured them. The final milestone is a pass that checks every spec statement against the code and opens a PR with a fixed body format.
7. **Gotchas.** A numbered list of every trap you found, each one line, to be re-read before every milestone.
8. **Definition of done.** A checkbox list.

Writing rules for the brief:

- **Be explicit to the point of redundancy.** Say "exactly", give literal strings, give default values in brackets, and name the error message text you expect.
- **Prefer code over prose** for interfaces, configs and algorithms.
- **Remove choices.** Pick the library, the layout and the names. A weaker agent given a choice will pick wrong or stall.
- **Every acceptance check must be runnable** and binary (pass or fail), never "looks good".
- **Security defaults go in the text**: secret redaction, no shell interpolation of untrusted input, and what must never be run with real credentials.
- **Don't pad.** Long is fine. Filler isn't. Every line should prevent a specific mistake.

### Phase 5: Verify and deliver

1. Re-read the brief adversarially, as the weaker agent would: where could it guess? Where could it misread a step? Fix those places.
2. Re-run any embedded reference code from the brief file itself (extract it and run it), not from your scratch copy.
3. Commit the brief to the target repo in a PR, and also attach the file to your reply so the user can hand it over directly.
4. Tell the user, in a few lines:
   - the one-line instruction to give the coding agent (for example "Read docs/IMPLEMENTATION_PROMPT.md fully, then do it milestone by milestone."),
   - the 2 or 3 biggest surprises the spike found,
   - anything you could not verify.
5. If a PR merges before your last commit lands, check what actually reached the default branch and open a follow-up PR for anything missing.

## Anti-patterns

- Writing the brief from memory or docs before running anything.
- "Use an appropriate library" or "handle errors sensibly". Name the library and give the error handling.
- Acceptance checks like "tests pass" with no named tests.
- Leaving open design questions inside the brief. Decide them, or list them under "Not verified" with what to do.
- Reference code you never executed.
- Telling the agent to use tools or paths on your machine instead of theirs.
