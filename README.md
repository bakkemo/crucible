# Crucible

**An AI agent harness with a memory you own, permissions you set, and undo
that actually undoes.** One executable, no install: a rich terminal UI, a
browser co-driver mirroring the same session, a persistent knowledge base
that packs itself into every prompt by relevance, per-turn git checkpoints
(`/retract` rewinds the files *and* the conversation together), unattended
task lists whose ticks are **proven by contract** — the harness checks the
work, the model doesn't get to grade itself — and a constraint solver the
model can call as a tool.

> **Beta 0.8.183 — download:**
> [**macOS** (Apple silicon, signed & notarized)](https://raw.githubusercontent.com/bakkemo/crucible/master/releases/crucible-beta-0.8.183.zip)
> · [**Windows** (x64)](https://raw.githubusercontent.com/bakkemo/crucible/master/releases/crucible-beta-windows-0.8.183.zip)
> · or from the [latest release](https://github.com/bakkemo/crucible/releases/latest).
> Unzip, read `READ-ME-FIRST.txt`, run. No API key needed to try it:
> keyless sessions run against a deterministic offline stand-in, and the
> banner tells you honestly which one you're on.
> **Looking for beta testers** — [what would help most](#beta-testers-wanted).

## Why another harness

Look at what a harness actually *does*, semantically:

| Harness pattern | What it actually is |
|---|---|
| try model A, else model B | **ordered choice with failure** |
| best-of-n sampling | **collect all solutions** of a nondeterministic computation |
| retry on transient error | **re-attempt a failing branch** |
| validate structured output against a schema | **unification** against a pattern with holes |
| fan out sub-tasks, join the results | **dataflow variables** |
| react to a streaming response | **partial data**, refined incrementally |
| speculative execution of plans | **choice + rollback** |

Every row is a concept functional logic programming had first-class forty
years before harnesses existed — failure and choice are Prolog, dataflow
variables are Oz, streams as incrementally-bound structure are Concurrent
Prolog. Written in a mainstream language anyway, a harness becomes a swamp
of `async`/`await`, retry libraries, and hand-rolled state machines — none
of it essential complexity.

So Crucible is built the other way: on **Obverse** https://bakkemo.github.io/obverse-playground/, a Verse-inspired
functional logic language designed alongside it, where those semantics are
the language. The thesis in one line: *stop encoding these semantics in a
language that lacks them.* Fallbacks, races, retries, schema validation,
speculative plans — in Crucible these are language constructs doing what
they say, not libraries fighting the language. That is what makes the
features below cheap enough to actually exist: real rollback, contract-
proven task lists, a solver in the tool manifest.

You never need to see Obverse to use Crucible — you type English and answer
approval prompts. It's there for the day you want to write a skill of your
own.

## The terminal

A real session — a fact worth remembering is offered to the knowledge base,
the harness asks first, the accounting row prices the turn:

![the rich terminal UI: a task typed, a memory_put approval, the answer](media/crucible-solo.gif)

## The terminal and the browser, one session

The same session mirrored live in a browser (`--serve`) while a task list
runs unattended: each checklist item is executed, then its **contract** is
checked against the sandbox — ticks land as *proven*, not claimed. The web
panel's rail shows per-item verdicts, checkpoints, and cost.

![the rich TUI beside the live web mirror, working through a task list](media/crucible-tasks.gif)

## Local models with KV-cache surgery

Crucible can drive [alekk89's KV-surgical fork of `llama.cpp`](https://github.com/alekk89/llama.cpp-kv-surgical-fork)
(`experimental/kv-surgery-dflash`) instead of an ordinary local server. The
fork holds a model's KV cache — its working memory of the conversation so
far — as something that can be edited in place: an old exchange evicted, a
hole compacted, the live tail extended, all without re-reading the whole
prompt. Crucible is the half that decides what belongs in that cache each
turn — which exchanges are still relevant, which are spent and can be
reclaimed, and, optionally, which get replaced by a short summary instead of
a bare marker. On a single machine the summary usually arrives after its
exchange has already been evicted; a late summary is spliced in over its
marker at the next make-room step, with no cache rebuild, and eviction
prefers ranges whose summary has already landed. A second server for the
summariser still shows the pairing at its best.

Setup is three lines once the server is running:

```
llama-server -m <model.gguf> --alias qwen2.5-7b --port 8081 --jinja --parallel 2 --slot-save-path ./slots
export CRUCIBLE_LOCAL_URLS=http://127.0.0.1:8081
/model qwen2.5-7b
```

Measured on a 7B model with an 8,192-cell window: a 20-file read chain, each
file naming the next, ran to completion and answered correctly at turn 21
with **zero cache rebuilds and 55 evictions**. The same window under
whole-transcript compaction — the mainstream approach, kept in Crucible as a
named control arm — re-summarised its way through the same chain and lost the
thread before finishing. Both are single measurements on one workload on one
model, not a benchmark claim. The benchmark itself ships in the program:
`crucible --bench-kv` runs the chain (and a competing-facts variant) against a
plain chat endpoint and against the surgical slot and prints a table —
`crucible --bench-kv --help` lists the knobs.

The macOS handout zip ships a prebuilt `llama-server-kv` and a
`start-kv-server.sh` launcher, built from the fork at commit `978dc96`. Windows builds the server itself from the
fork's own CUDA recipe — no prebuilt Windows binary yet.

`QUICKSTART-KV.pdf`, in the zip, walks the whole setup end to end for someone who has never run a local model
server before.

## What's in the zip

| file | what it is |
| --- | --- |
| `crucible` / `crucible.exe` | the whole program, one file |
| `READ-ME-FIRST.txt` | 90 seconds of setup, honestly told |
| `start-crucible.sh` / `.cmd` | one-line setup + launch |
| `QUICKSTART-BETA.pdf` | a 20-minute guided tour |
| `crucible-manual.pdf` | the full reference, with index |
| `task-list-tutorial.pdf` | one subsystem taught properly, front to back |
| `QUICKSTART-KV.pdf` | the local-model setup, with KV-cache surgery, end to end |
| `llama-server-kv` / `start-kv-server.sh` | macOS only: the KV-surgical `llama.cpp` server, prebuilt and signed, and its launcher (with its `LICENSE` and `NOTICE`) |

Platform notes: the **macOS** build is Developer-ID signed and the zip is
notarized by Apple — no warnings; at most the standard one-time "downloaded
from the internet" confirmation, which `start-crucible.sh` clears. The
**Windows** build is young and unsigned (SmartScreen will ask once — "More
info → Run anyway"); the macOS build has months of daily use behind it,
the Windows one is where beta reports matter most.

## Beta testers wanted

I'm looking for **around fifty people** to run Crucible on real work for a
couple of weeks and tell me what happened. No sign-up, no commitment:
download it, use it, report.

What would help most:

- **Local models.** Point it at Ollama (`OLLAMA_HOST`) or at LM Studio,
  llama.cpp or vLLM (`CRUCIBLE_LOCAL_URLS`), pick the model in `/model`,
  and tell me which models hold up through tool calls and which go strange.
  That territory is barely measured; every report moves it.
- **Windows.** The build is fresh and has had no daily use. A plain "it
  launched and ran a task" is a data point. Anything that broke is a
  better one.
- **Confusion.** A command that didn't do what its name suggested, a manual
  page that didn't answer the question, a banner line you had to guess at.
  Those count as bugs here.
- **The unattended path.** A task list with contracts, left to run: did the
  *proven* and *claimed* verdicts match what you found in the sandbox
  afterwards?

Send it as an [issue](https://github.com/bakkemo/crucible/issues) or to
**crucibleharness@gmail.com** — a sentence about what you did and what you
saw is plenty; screenshots welcome. Keyless sessions bill nothing, so
kicking the tyres costs nothing either.

Everything in the GIFs above is real: both were recorded from live sessions
of the shipped binary — the terminal frames through Crucible's own
`--debug-tty` screen readout, the browser frames photographed from the
actual page while it ran.

*Source release to follow.*
