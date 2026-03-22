# linus-reviewer

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [Cursor](https://cursor.com), [Codex CLI](https://github.com/openai/codex), and [OpenCode](https://opencode.ai) plugin that reviews your code the way Linus Torvalds reviews kernel patches — brutally honest, technically precise, and obsessed with good taste.

> *"Sometimes you can see a problem in a different way and rewrite it so that a special case goes away and becomes the normal case, and that's good code."* — Linus Torvalds, TED 2016

A multi-layer Linus-style analysis that judges your code on:

- **Data structures** — Are they right? Does the code revolve around good data design, or is it patching around bad structures with conditionals?
- **Special cases** — Can they be eliminated through better design? (The pointer-to-pointer test)
- **Cognitive load** — How many context switches does a reader need? Is related code together?
- **Abstraction audit** — Is every abstraction earning its keep, or is it a premature "helper" that obscures meaning?
- **Practicality** — Does this solve a real problem? Does the complexity match the severity?

Each finding gets a taste rating: **GARBAGE**, **BAD TASTE**, **MEH**, or **GOOD TASTE**.

[Usage](#usage) | [Install](#install) | [How It Works](#how-it-works) | [Design Principles](#design-principles) | [References](#references) | [Hooks](#hooks) | [Configuration](#configuration)

## Usage

```
/linus-review                     # review current branch changes
/linus-review 405                 # review specific PR/MR
/linus-review --fix               # review + auto-fix surviving issues
/linus-review --fix 405           # auto-fix specific PR/MR
```

## Install

### Claude Code

```
/plugin marketplace add mattbucci/linus-reviewer
/plugin install linus-reviewer@mattbucci/linus-reviewer
```

### Cursor / cursor-agent

```bash
git clone https://github.com/mattbucci/linus-reviewer /tmp/linus-reviewer
cp -r /tmp/linus-reviewer/skills/linus-review ~/.cursor/skills-cursor/linus-review
```

Then start a new cursor-agent session and `/linus-review` will be available.

### Codex CLI

```bash
git clone https://github.com/mattbucci/linus-reviewer /tmp/linus-reviewer
cp -r /tmp/linus-reviewer/skills/linus-review ~/.codex/skills/linus-review
```

Then start a new codex session and invoke with `$linus-review`.

### OpenCode

```bash
git clone https://github.com/mattbucci/linus-reviewer /tmp/linus-reviewer
cp -r /tmp/linus-reviewer/skills/linus-review ~/.config/opencode/skills/linus-review
```

Then start a new opencode session and `/linus-review` will be available.

## Hooks

This plugin focuses on **design review** — it does not run linters, type checkers, or tests. Those mechanical checks belong in [Claude Code hooks](https://docs.anthropic.com/en/docs/claude-code/hooks) that run automatically, so problems are caught before the review even starts.

> *"It doesn't even compile... this clearly never even got a whiff of build-testing. Stop sending me garbage."*

Configure hooks in your project's `.claude/settings.json`:

**Lint on every file edit:**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix $CLAUDE_FILE_PATH 2>/dev/null || ruff check --fix $CLAUDE_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

**Run lint and tests before any agent stops:**
```json
{
  "hooks": {
    "AgentStop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/check.sh"
          }
        ]
      }
    ]
  }
}
```

Where `.claude/hooks/check.sh` runs your lint and test suite. Return exit code `2` with a reason on stderr to **block** the agent from stopping:

```bash
#!/bin/bash
set -e

# Lint
if ! npx eslint . 2>&1; then
  echo "Lint is failing. Fix lint errors before you stop." >&2
  exit 2
fi

# Tests
if ! npm test 2>&1; then
  echo "Tests are failing. Fix them before you stop." >&2
  exit 2
fi

exit 0
```

The plugin assumes your code already compiles and passes lint. If it doesn't, that's on you — Linus wouldn't even look at it.

## How It Works

> *"Developers have the attention spans of slightly moronic woodland creatures."* — LinuxCon Europe Keynote

```
Phase 1: Gather Diff
    │
    ├─ docs/config only → skip with note
    │
Phase 2: Five-Layer Linus Analysis
    │
    ├─ Layer 1: Data Structures ──┐
    ├─ Layer 2: Special Cases ────┼── 4 parallel agents
    ├─ Layer 3: Cognitive Load ───┤   (per-file sub-agents for 5+ files)
    ├─ Layer 4: Abstraction Audit ┘
    │
    └─ Layer 5: Practicality & Breakage (deduplicates & synthesizes)
    │
Phase 3: Taste Rating & Findings
    │
Phase 4: Auto-Fix (if --fix, max 2 iterations)
    │
Phase 5: Report (console + PR comments + artifacts)
```

## Design Principles

> *"Bad programmers worry about the code. Good programmers worry about data structures and their relationships."* — Linus Torvalds, Git mailing list, 2006

This plugin applies a specific set of code review principles drawn from Linus Torvalds' documented reviews, talks, and emails, combined with related software design wisdom:

| # | Principle | Origin |
|---|-----------|--------|
| 1 | **Good Taste** — Eliminate special cases through better design | Linus's 2016 TED Talk, linked list pointer-to-pointer example |
| 2 | **Data Structures First** — Good programmers worry about data structures, not code | Git mailing list, July 2006 |
| 3 | **Explicit Over Abstract** — Don't hide simple operations behind names that obscure meaning | RISC-V pull request rejection, August 2025 |
| 4 | **AHA: Avoid Hasty Abstractions** — Prefer duplication over the wrong abstraction | Sandi Metz; reinforced by Linus's `make_u32_from_two_u16` critique |
| 5 | **Cognitive Locality** — Keep related code together, minimize context switches | Engineer's Codex analysis of Linus's "garbage code" critique |
| 6 | **Simplicity** — 3+ levels of indentation means your program needs fixing | Linux kernel coding style document |
| 7 | **Pragmatism** — Theory loses to practice, every single time | Core Linus philosophy across all reviews |
| 8 | **Don't Break API Contracts** — Don't change interfaces your callers depend on (APIs, CLIs, config formats, wire protocols) | "WE DO NOT BREAK USERSPACE!" — kernel policy on syscall/ioctl/proc/sysfs stability |
| 9 | **Understand What You Copy** — Blindly copied code is garbage | Google contributor filesystem dispute, January 2024 |
| 10 | **Build Discipline** — Zero warnings, must compile, must test | Multiple kernel mailing list emails |
| 11 | **Single Source of Truth** — Don't store derived state, compute it | Engineer's Codex "4 Design Principles" |
| 12 | **Minimize Mutable State** — Fewer things changing = fewer things breaking | Engineer's Codex "4 Design Principles" |

## References

Direct quotes and source material used to build this plugin's review principles. Includes both critiques and praise — Linus reviews aren't all rants.

### What Good Code Looks Like

> *"That looked even simpler than what I thought it would be... I really like how this looks."* — Linus praising a page cache optimization, 2025

| Source | Context | Key Quote | Principle |
|--------|---------|-----------|-----------|
| [TED Talk](https://www.ted.com/talks/linus_torvalds_the_mind_behind_linux) (2016) | Demonstrating "good taste" via linked list deletion — eliminating the head-node special case with pointer-to-pointer | *"Sometimes you can see a problem in a different way and rewrite it so that a special case goes away and becomes the normal case, and that's good code."* | Good taste = eliminating special cases through better design |
| [RDMA "Good Job" Email](https://lore.kernel.org/all/CA+55aFxmnW-iu1Na3QC8Ci8Q_Qdfn2Ak_9wDB6+A564R=Xn9Ag@mail.gmail.com/) (Jan 2018) | Linus deliberately sent a praise email to Jason Gunthorpe for a well-documented RDMA back-merge | *"A thoughtful merge with explanation for why the conflict happened... So I thought I'd send out a 'good job' email just as a contrast."* | Explain your reasoning — merges that document *why* earn explicit praise |
| [Optimizing Small Reads](https://lore.kernel.org/all/CAHk-=wi4Cma0HL2DVLWRrvte5NDpcb2A6VZNwUc0riBr2=7TXw@mail.gmail.com/) (Oct 2025) | Praising Kiryl Shutsemau's page cache seqcount optimization for being simpler than expected | *"That looked even simpler than what I thought it would be... I really like how this looks."* | Simplicity wins — the best code is simpler than you expected |
| [Folio Refcount Scalability](https://lore.kernel.org/all/CAHk-=whc7ne30jPpe=FFECg4OBUAqEqzQUXJ+vdb8NyYDW1anA@mail.gmail.com/) (Mar 2026) | Pedro Falcato's locked-bit folio approach — Linus admitted his initial skepticism was wrong | *"My initial reaction was that it was complicated, but... I think this is actually the better model."* | Be willing to change your mind when the evidence supports it |
| [Shrinking rwsem](https://lore.kernel.org/all/CAHk-=wjkyw-sap1dNkW7v8at8MvF3j5wshC1Gw3XEpHBbBw6BQ@mail.gmail.com/) (Feb 2026) | Matthew Wilcox replacing a list_head with a first-waiter pointer to shrink struct rw_semaphore | *"Changing to a first waiter pointer instead of a struct list_head is a good change."* | Shrink core data structures — smaller hot-path structs = real wins |
| [Fast Short Reads](https://lore.kernel.org/all/CAHk-=wgijo0ThKoYZeypuZb2YHCL_3vdyzjALnONdQoubRmN3A@mail.gmail.com/) (Oct 2025) | Praising Kiryl Shutsemau's filemap_read() split | *"I like how it splits up those two cases more clearly and makes the baseline filemap_read() much simpler and very straightforward."* | Separate fast and slow paths cleanly |
| [Static-Duration kmem_cache](https://lore.kernel.org/all/CAHk-=wiibHkNcsvsVpQLCMNJOh-dxEXNqXUxfQ63CTqX5w04Pg@mail.gmail.com/) (Jan 2026) | Al Viro's approach to making kmem_cache use static storage | *"I like it. Much better than runtime_const for these things."* | Prefer the simpler architectural approach |
| [__VA_OPT__ Support](https://lore.kernel.org/all/CAHk-=wghBU1bB6E889nv5fg+40b4_n5WnA0h3bVh32EKw0zE2w@mail.gmail.com/) (Mar 2026) | Al Viro's 21-patch sparse preprocessor series — Linus had a rant ready but deleted it | *"This makes things better... So I decided to just delete my rant."* | Pragmatic improvement over perfection |
| [arm64 uaccess Rewrite](https://lore.kernel.org/all/20240709161022.1035500-1-torvalds@linux-foundation.org/) (Jul 2024) | Rewriting arm64 access_ok() to use bit 55 checks instead of overflow handling | *"I like the bit 55 checks — they are actually simpler than worrying about 64-bit overflow."* | Reframe the problem so complexity disappears |
| [Perf Annotate Loop Detection](https://lore.kernel.org/all/CA+55aFy7QLXRWQSjUiYNhX5B9p-0AJ4LSWc=km8hJ55ku=ho0A@mail.gmail.com/) (Apr 2012) | Praising Arnaldo Carvalho de Melo's perf/annotate improvements | *"I think it's a very useful concept and generally well done."* | Make hard things readable — good tooling surfaces problems clearly |
| [Git Mailing List](https://lwn.net/Articles/193245/) (Jul 2006) | On why git succeeded — designed around data structures, not algorithms | *"Bad programmers worry about the code. Good programmers worry about data structures and their relationships."* | Data structures first, always |

### What Bad Code Looks Like

> *"You copied that function without understanding why it does what it does, and as a result your code IS GARBAGE. AGAIN."* — Linus on the eventfs dispute, 2024

| Source | Context | Key Quote | Principle |
|--------|---------|-----------|-----------|
| [RISC-V Pull Request Rejection](https://lore.kernel.org/lkml/CAHk-=wjLCqUUWd8DzG+xsOn-yVL0Q=O35U9D6j6=2DUWX52ghQ@mail.gmail.com/) (Aug 2025) | Rejecting `make_u32_from_two_u16()` in generic headers — a helper that obscures meaning | *"That thing makes the world actively a worse place to live. It's useless garbage that makes any user incomprehensible."* | Explicit > Abstract; don't pollute shared code with bad helpers |
| [RISC-V Pull Request Rejection](https://lore.kernel.org/lkml/CAHk-=wjLCqUUWd8DzG+xsOn-yVL0Q=O35U9D6j6=2DUWX52ghQ@mail.gmail.com/) (Aug 2025) | On why explicit bit operations are better than helper functions | *"If you write `(a << 16) + b`, you know what it does... if you write make_u32_from_two_u16(a,b) you have not a f\*cking clue what the word order is."* | Code should be immediately comprehensible at the call site |
| [Eventfs "IS GARBAGE"](https://lore.kernel.org/all/CAHk-=wgZEHwFRgp2Q8_-OtpCtobbuFPBmPTZ68qN3MitU-ub=Q@mail.gmail.com/) (Jan 2024) | Rejecting Steven Rostedt's eventfs code for copying VFS functions without understanding them | *"You copied that function without understanding why it does what it does, and as a result your code IS GARBAGE. AGAIN."* | Understand what you copy; ignorance of the system = garbage |
| [Eventfs Dispute](https://www.theregister.com/2024/01/29/linux_6_8_rc2) (Jan 2024) | On inode assumptions from decades-old filesystem design | *"An inode number just isn't a unique descriptor any more. We're not living in the 1970s."* | Don't design for how systems used to work |
| [mseal() Conceptual Incoherence](https://lore.kernel.org/lkml/CAHk-=wh+6n6f0zuezKem+W=aytHMv2bib6Fbrg-xnWOoujFb6g@mail.gmail.com/) (Oct 2023) | Demanding conceptual clarity before writing code for the mseal() syscall | Demanded the *concept* be right before discussing implementation | Conceptual clarity first — don't polish implementation of a broken idea |
| [Userspace Exception "Abomination"](https://lore.kernel.org/lkml/CAHk-=wjJhdr3JCnGrMKqL-prxYd__kkAspKVYBO3BYYmq2hu4A@mail.gmail.com/) (Nov 2018) | Rejecting a broken signal handler variant | Called the design an *"abomination"* | Don't break the contract between kernel and userspace |
| [mprotect Security Flaw](https://lore.kernel.org/all/CAHk-=wj4KCujAH_oPh40Bkp48amM4MXr+8AcbZ=qd5LF4Q+TDg@mail.gmail.com/) (Jun 2021) | Catching a critical permission escalation bug in mprotect | Spotted a privilege escalation that reviewers missed | Security-critical code demands extra scrutiny |
| [Attribute Ordering](https://lore.kernel.org/all/CAHk-=wiOCLRny5aifWNhr621kYrJwhfURsa0vFPeUEm8mF0ufg@mail.gmail.com/) (Sep 2021) | `int static myfn(void)` compiles but is wrong | *"Can you do it in different orders? Yes... That most certainly doesn't make it right."* | Compiling != correct; follow conventions |
| [Pluggability Is Bad](https://lore.kernel.org/all/alpine.LFD.0.999.0710022050560.3579@woody.linux-foundation.org/) (Oct 2007) | On making things pluggable for flexibility's sake | *"Pluggable things are generally a bad thing."* | Don't add abstraction layers for hypothetical flexibility |

### Rules and Philosophy

> *"WE DO NOT BREAK USERSPACE! Seriously. How hard is this rule to understand?"* — Linus Torvalds, 2012

| Source | Context | Key Quote | Principle |
|--------|---------|-----------|-----------|
| ["WE DO NOT BREAK USERSPACE"](https://lkml.org/lkml/2012/12/23/75) (Dec 2012) | The canonical statement of the kernel's #1 rule — specifically about syscall, ioctl, /proc, /sys, and device interfaces that userspace programs depend on | *"WE DO NOT BREAK USERSPACE! Seriously. How hard is this rule to understand?"* | Don't break the API contracts your callers depend on |
| [Buggy Apps Are Still Protected](https://lore.kernel.org/lkml/CA+55aFxZ4YESQ0vr3qO_o_H8AvQPOv500qnbBXdq+4YNsnOPgA@mail.gmail.com/) (Jan 2015) | Even insane userspace code counts as a regression if you break the kernel interface it relies on | Even buggy apps are protected by the "no regressions" rule | The bar for "breaking change" is set by actual callers, not spec compliance |
| [Apple SSD Spec Compliance](https://lore.kernel.org/lkml/CAHk-=wgML11x9afCvmg9yhVm9wi5mvnjBvmX+i7OfMA0Vd4FWA@mail.gmail.com/) (Sep 2021) | On demanding hardware comply with specs | *"If we start demanding hardware comply with specs, we'd have to scrap the whole notion of working in the real world."* | Pragmatism over spec purity |
| [-O3 Optimization Skepticism](https://lore.kernel.org/lkml/CA+55aFz2sNBbZyg-_i8_Ldr2e8o9dfvdSfHHuRzVtP2VMAUWPg@mail.gmail.com/) (Jun 2022) | Demanding benchmarks before switching compiler optimization levels | Demanded data, not assumptions | Prove it with benchmarks, not theory |
| [Hash Function Math](https://lore.kernel.org/all/CA+55aFyjYkwkTo2bNYqJ6h4mr1bbT5Vrak+EtiZmujOD-NzMOQ@mail.gmail.com/) (Apr 2016) | Correcting a 14-year-old hash constant by reading Knuth | Deep-dived into the math to fix a subtle constant error | Understand the theory behind the code you maintain |
| [False Positive Warnings](https://lore.kernel.org/lkml/CAHk-=wgoE5EkH+sQwi4KhRhCZizUxwZAnC=+9RbZcw7g6016LQ@mail.gmail.com/) (May 2024) | On arithmetic overflow warnings | *"False positive warnings make real warnings worthless."* | Signal-to-noise ratio matters — noisy tools get ignored |
| [List Iterator and C99](https://lore.kernel.org/lkml/CAHk-=wiyCH7xeHcmiFJ-YgXUy2Jaj7pnkdKpcovt8fYbVFW3TA@mail.gmail.com/) (Feb 2022) | Fix root causes (C99 scoping) not symptoms | Fixed the language-level root cause instead of patching around it | Fix root causes, not symptoms |
| [Variable Shadowing Self-Debug](https://lore.kernel.org/all/CAHk-=wiCJtLbFWNURB34b9a_R_unaH3CiMRXfkR0-iihB_z68A@mail.gmail.com/) (Nov 2023) | Linus finding his own variable shadowing bug | *"Oh God, I'm so stupid"* | Even experts make mistakes — humility is part of good engineering |
| [Linux Kernel Coding Style](https://www.kernel.org/doc/html/latest/process/coding-style.html) | Official kernel coding style on function complexity | *"If you need more than 3 levels of indentation, you're screwed anyway, and should fix your program."* | Complexity is a design failure |
| [LinuxCon Europe Keynote](https://www.youtube.com/watch?v=MShbP3OpASA) | On developer attention and code quality | *"Developers have the attention spans of slightly moronic woodland creatures."* | Code must be readable by distracted humans |
| [Slashdot Q&A](https://grisha.org/blog/2013/04/02/linus-on-understanding-pointers/) | On pointer-to-pointer understanding | *People who don't understand pointers need the extra `if` — the indirect pointer eliminates it entirely.* | Understanding the system eliminates complexity |
| [Sandi Metz — "The Wrong Abstraction"](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction) | AHA (Avoid Hasty Abstractions) principle | *"Prefer duplication over the wrong abstraction."* | Two copies of clear code > one tangled "reusable" helper |
| [Engineer's Codex — "Garbage Code"](https://read.engineerscodex.com/p/how-to-not-write-garbage-code-by) | Cognitive load analysis of Linus's RISC-V rejection | *Micro-context switches impose cognitive costs. Helpers that force readers through multiple files can increase load.* | Cognitive locality matters; abstractions have a cost |
| [Engineer's Codex — "4 Design Principles"](https://read.engineerscodex.com/p/4-software-design-principles-i-learned) | On derived state, mocks, and PRY | *"If there's two sources of truth, one is probably wrong."* | Single source of truth; minimize mutable state |

### Language Style

Part of what makes Linus's reviews effective is his characteristic directness. This plugin channels the **precision** without the profanity:

| Pattern | Example | What It Means |
|---------|---------|---------------|
| "garbage" | *"This is garbage and it came in too late."* | Fundamentally wrong approach |
| ALL CAPS for emphasis | *"your code IS GARBAGE. AGAIN."* | Repeated failure or critical point |
| Concrete counter-examples | *"If you write `(a << 16) + b`, you know what it does"* | Always shows the better way |
| Rhetorical framing | *"sending a big pull request... in the hope that I'm too busy to care is not a winning strategy"* | Calling out bad strategy |
| Absolute statements | *"Theory loses. Every single time."* | No wiggle room on core principles |
| The sign-off | `Linus` | Simple, authoritative, final |

## Configuration

The plugin respects project-specific review guidelines from:
- `REVIEW.md` in the project root
- `.claude/docs/code-review.md`
- `CLAUDE.md` project instructions

## License

MIT
