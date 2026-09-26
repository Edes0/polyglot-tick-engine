# Working with AI agents

[← back to the index](../README.md)

I build this with AI coding agents. The typing is the cheap part. The engineering is in making their
output trustworthy: deciding what they may do, checking what they claim, and noticing when a guardrail
has quietly stopped guarding.

> Designed and built solo over ~10 months by Andreas Sjögren — I make the decisions and review every merge; AI coding agents implement under gates I built (see Working with agents).

<!-- budget:inshort max=60 -->
> **In short.** **Problem:** an agent is confident, fast, and does not remember last week. **Decision:**
> repo-resident knowledge in four kinds of file, every agent audit checked against the code before
> anyone acts on it, and gates that must be seen to fail before they are trusted. **Outcome:** audits
> that were wrong got caught before they became refactors.
<!-- /budget -->

---

## Who does what

I write the specs and make the design decisions. Agents implement against those specs, under a
workflow of gates I built — a spec is grilled before it is planned, a plan before it is implemented,
and a diff is reviewed before it merges. I review every merge.

The project has had rules for AI assistants since its second day. The early months ran on Cursor;
Claude Code took over from May 2026, and most of the project's commits date from after that switch. None of that changes whose decisions these are — the rest of this write-up is those
decisions.

## Knowledge has four homes

Agents start every session knowing nothing about last week. Early on, the answer was one gotchas file
that every agent read and wrote. It grew to **94 entries, 626 lines, 169.6 KB** — and when I
classified the entries, 77.7% were facts about this codebase, but the rest were generic platform
facts, tooling quirks and process advice, all mixed together. Worse, the gate that was supposed to
route design decisions into records had fired on **0 of the 9 specs** created since it was wired,
because the file it looked in never declared where records live.

So agent-facing knowledge now has four homes, keyed by kind and lifecycle: decisions in ADRs,
vocabulary in a glossary, codebase facts in the gotchas file with a generated index, and tooling
quirks split by whether they are specific to this repo. Two rules came with it, both learned the hard
way:

- **Prune by relocating and indexing, never by reading entries and deleting them.** Removing one entry
  that looked redundant once dropped an evaluation score from 1.00 to 0.00.
- **Measure a knowledge change over at least five runs, compared on medians.** A single run has been
  seen to reverse the sign of the effect.

And one that sounds obvious and was not: knowledge the project depends on has to live **in the
repository**. Subagents, orchestrated runs and agents working in separate worktrees load the repo,
not my personal notes — so a fact that only I and my own assistant knew was invisible to the agents
doing the work.

## An audit is a claim, not a finding

Agents produce audits quickly and confidently. Treating those as findings is how a codebase gets
refactored toward someone's misreading of it. Three times, checking against the code first saved the
work:

- A refactor sweep planned two "named bugfixes". Both were **audit misreads** of code that was
  correct. They were documented instead of fixed.
- A spec to remove an asynchronous reservation table was **blocked at implementation**, because the
  caller-graph audit it depended on refuted its own premise.
- A documentation sweep found eight places where the docs contradicted the code, and resolved every
  one **against the code**. Two of the sweep's own findings were wrong and were dropped — one had
  inferred an off-by-one from a doc comment when the operator in the code said otherwise.

The ownership guard in [the trust boundary](09-trust-boundary.md#a-guard-that-was-fail-open-in-six-places)
is the sharpest case: an audit had the drift exactly backwards, and trusting it would have "fixed" the
two correct copies to match the six loose ones.

## Guardrails go quiet without telling you

A gate that has stopped firing looks exactly like a gate with nothing to catch. Four examples:

- **Three gates had gone inert.** Two end-of-task gates and a guard matched tool names with a pattern
  that could not match the renamed editor integration — verified by testing the pattern directly — so
  they had silently stopped applying. The fix widens the match; it is in review as I write this.
- **A gate that could block forever.** The test gate on finishing a task had no ceiling, so a
  permanently red suite blocked every stop. It now bails open after five cycles.
- **A test that cannot fail proves nothing.** A regression test salvaged from an abandoned branch was
  only trusted after being made to go red on purpose: cut the loop one tick short, watch it fail.
- **A merge that skipped the one check that mattered.** A fog-of-war rendering fix was merged before
  its visual play-test, which is the only gate a visual change actually has. It was reverted the next
  day.

Parallel agents taught one more, the expensive way: two agents shared a worktree, and one's commit
swept in the other's staged files, because `git commit` commits the whole index, not a path. The rule
since then is to stage explicit paths and never `git add -A`.

## Keeping the map smaller than the territory

Agents navigate by the `AGENTS.md` files in the tree. By September there were 113 of them, and the
root file was 149 lines. One consolidation pass cut that to **51 files and an 86-line root**, and took
the link sweep from 16 broken links and 4 dangling code anchors to zero. Documentation an agent reads
on every task is a hot path; it gets the same attention as one.

---

**Next:** [The world and its look →](11-world-and-visual-identity.md)
