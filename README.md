# Mutation-Proven Tests Prompt

Ask an agent for tests and it will raise coverage and call it done. Coverage only says a line ran. A test that calls
a function and asserts nothing covers it just as well as one that would catch a bug.

Mutation-Proven Tests builds or strengthens a repo's test suite and proves the tests work. It breaks the code on
purpose, one small change at a time (a `<` turned into `<=`, a call removed), and counts only the tests that notice.
The number it reports is the kill rate for each module, before any test was written and after the last one. Every
change the tests still miss is killed by a new test, shown to be harmless, or put to you as a decision.

The prompt is [prompt.md](prompt.md). Nothing to fill in: the agent finds what it needs in your repo and asks you the
rest.

## Use it

1. Create an empty folder beside your repo for the run's tooling and evidence: `../my-app-mutation`, where `my-app`
   is your repo's folder name.
2. Start your agent in the repo with the setup below.
3. Paste all of [prompt.md](prompt.md), appendices included, as your first message. Optionally add a line after it
   naming modules to do first, modules to leave alone, or a time budget.
4. Answer its questions and approve its plan. It stops twice more: after the first module, and before the end. It
   commits on your current branch, never pushes on its own, and checks CI after you push.

If a run stops (a crash or a usage limit), start it the same way and it resumes from the progress log.

Most of a run is waiting on mutation runs, which execute your tests once for every change they try. A small library
takes hours; a large app takes days unless you let it sample the lower-risk modules, and it will tell you which
numbers are sampled.

## Setup

Last checked November 2026. Run it locally: it needs your test suite, and whatever the tests need, on your machine.

| | Claude Code | OpenAI Codex |
|---|---|---|
| Model | Opus 5.5 (Fable 5.2 for the most thorough run) | GPT-6.1 Sol |
| Effort | `xhigh` | Extra High, including Plan mode |
| Plan phase | Plan mode | `/plan` |
| Long run | Approve with "Yes, and use auto mode" | `/permissions`, then Approve for me |

```bash
claude --model opus --effort xhigh --permission-mode plan --add-dir ../my-app-mutation --disallowedTools "Bash(git push *)"
```

```bash
codex -m gpt-6.1-sol -c model_reasoning_effort=xhigh -c plan_mode_reasoning_effort=xhigh -c sandbox_workspace_write.network_access=true --add-dir ../my-app-mutation
```

If you've run Zero-Drift Upgrade on this repo, also pass `--add-dir ../my-app-upgrade`, so the agent can reuse its
mock and seed data.

The Claude command blocks `git push`, which auto mode would otherwise allow. In a desktop app, make the same choices
from its menus. A full run outgrows Claude Pro and ChatGPT Plus limits; Claude Max, ChatGPT Pro or an API key suits it.

## What you need

- A clean checkout whose dependencies install and whose tests run locally. A repo with no tests works too: its
  baseline is zero.
- The mutation tool for your stack, or your go-ahead to install it. The prompt has appendices for .NET, Node and
  TypeScript, Python, Java and Kotlin, Go and Rust, and names tools to try for seven more stacks. Where no tool
  fits, it makes the changes by hand, from a fixed list, and labels those numbers as manual. Where a tool lists
  its mutants but can't run them honestly on your suite, the tool lists them and a script it commits runs them.
- Whatever your tests depend on, running locally: a database container, a mock. The agent won't run broken code
  against a real service, so tests it can't make local are left out and listed.
- A machine with little else to do. A mutation run takes every core it is given.

## What you get

- In the repo: test commits for each module; a commit for each fixed flaky test; any small change to production code
  that a test needed and you approved; the mutation tool's pinned version and config, unless you'd rather keep them
  out; and a "Mutation testing" section in the README with the command that reproduces the numbers.
- In the mutation folder: the plan, a progress log, `targets.md` (every module, its risk and how it was run),
  `setup.md` (what was fixed in place before the baseline, and the proof that the setup notices a weak test),
  `flaky.md`, `survivors.jsonl` (every change the tests missed, and how it ended), `inventory.md` (bugs found along
  the way, none fixed), the tool's raw reports, the checkpoint pages and a README on re-running it all. At the end
  the agent deletes what it can rebuild, unless you ask it to keep everything.
- A final report: the kill rate for each module before and after, what was accepted and why, and what didn't run.

## How the number works

A mutant is one small change to your code. It is killed when a test that passes on the real code fails on the
changed code.

**kill rate = killed ÷ (killed + survived + no coverage + timed out + undetermined)**

A few choices keep the number from flattering anyone:

- A timeout isn't a kill. Most tools count it as one, but a clock also notices a busy machine. Each timed-out
  mutant is run again alone, and counts against the rate unless a test fails on it. For the same reason a test's
  own time budget failing never counts: mutated code runs slower.
- Changes that can't alter behavior (equivalent mutants) and ones you agreed aren't worth a test stay in the
  denominator, in their own columns. Calling a survivor harmless never raises the rate, so 100% is rare, and the
  finish line is that every survivor has been dealt with.
- Before and after are measured on the same mutants, with the same tool settings, and the settings are fixed
  before the first number is taken. Excluding code never helps: the rate with exclusions counted is printed too.
- Before any number counts, the agent strips the assertions out of a working test in a scratch copy and checks that
  the tool now reports those changes as missed. A setup that can't see a weak test proves nothing.
- A test that fails some of the time looks like a kill. Flaky tests are found by repeated runs and fixed or set
  aside first, and every new kill is confirmed three times.
- A module that was sampled says so next to its number. Its final figure comes from a second sample the agent
  never saw while writing tests, measured before and after like the first.
- A test that only writes down what the code does today (because nothing says what it should do) is marked, and
  the rate without those tests' kills is printed beside the headline.

## Works with the other three

- [Audit-to-A](https://github.com/MorinelloA/audit-improvement-prompt): its layer map, critical flows, testing
  findings and coverage report decide which modules go first. Re-run Audit-to-A afterwards to close those findings.
- [Zero-Drift Upgrade](https://github.com/MorinelloA/zero-drift-upgrade-prompt): run this first and the upgrade's
  tests oracle has a suite worth trusting, with its flaky tests already found.
- [UI/UX Audit and Fix](https://github.com/MorinelloA/ui-ux-audit-and-fix-prompt): its layout spec is an end-to-end
  suite, which mutation runs leave out.

The agent reads all three and writes into none.

## What it doesn't cover

Fixing bugs (it logs them, and asks whether a test should hold today's behavior in place), refactoring, performance,
UI layout, version upgrades and new features. It doesn't mutate against end-to-end, browser or device suites, and it
doesn't chase a coverage figure.

## License

[MIT](LICENSE)
