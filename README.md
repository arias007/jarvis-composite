# Jarvis — One Assistant, Four Abilities, One Memory

> **Status: specification, not software.** Nothing here is built yet. This repository exists to write the target down precisely enough that building it later is engineering rather than invention — and to record what the current tools get wrong, from daily use, so that every complaint becomes an acceptance test instead of a shipped bug.

---

## The claim

The assistant I want is one thing that is simultaneously:

| Ability | Today's best available implementation | How good it actually is |
|---|---|---|
| **Memory** | Obsidian | Local Markdown, a real plugin API, link semantics that make a vault navigable. Closed source, and the plugin host cannot be borrowed. |
| **Hands** | Mika | Virtual display, accessibility with OCR and vision fallback, a shell helper over its own ADB implementation. Genuinely the hardest part, genuinely solved. |
| **Workspace** | Minis / OpenMinis | A real Linux userland on the phone — shell, browser, files, packages. |
| **Entry point** | wmkeyboard | The one surface always in front of you. Speak to the agent from the keyboard, in any app, without leaving what you were typing. |

None of these is the thing. Each is one plane of the thing, implemented separately, by people solving one problem each.

**The position, stated without hedging:** these four are the least-bad implementations available, and nothing else is even close to being an option. That is not a compliment. It is a description of a gap that is going to be filled — by someone, and the only question is whether it gets filled well.

**The claim this repository makes is a time-window claim, not a technical one:** before neural interfaces, glasses and general-purpose robotics become ordinary, an assistant that is always reachable, remembers everything, and can actually act is the most valuable thing a person can have on their phone. Everything below is either evidence for that or an honest account of the work.

---

## The correction

One application, four abilities, one memory. Not four apps glued together — one runtime that happens to be good at all four things, and gets better because they share state.

I was wrong to argue against convergence in the first draft of this file. Let me state the corrected position plainly, because it matters.

**Convergence is inevitable and if nobody does it, I will.** These four are the best available implementations of four parts of one idea. The parts are what I need; the current implementations are incidental. If merging them is the only way to get a Jarvis, then merging them is the plan — not a fallback.

**What the current situation actually is:** the four are the least-bad option right now, and nobody else is even close to being an option. That is not a compliment to them. It is a description of a gap.

**What I am not doing:** treating "it's hard" as an argument. It is not an argument. The only arguments that count are "it doesn't get me closer to the thing" and "there is a cheaper path to the same thing." Neither of those is true here.

So: the target is one thing that is a keyboard, a memory, a workspace and a pair of hands at once. Everything below is about *how* to get there without the result being worse than the four parts.

---

## Architecture: one runtime, four planes

The engineering objection is real but it is about **scheduling**, not about **ownership**. The fix is one process with hard internal boundaries, not four processes with a network between them.

```
┌──────────────────────────────────────────────────────────┐
│  Jarvis                                                   │
│                                                           │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐  │
│  │ INPUT PLANE │  │  MIND PLANE  │  │ ACTION PLANE    │  │
│  │             │  │              │  │                 │  │
│  │ IME service │  │ Agent loop   │  │ Virtual display │  │
│  │ candidates  │  │ Context      │  │ Accessibility   │  │
│  │ voice       │  │ Memory       │  │ Shell helper    │  │
│  │             │  │ Tools        │  │ Browser engine  │  │
│  └──────┬──────┘  └──────┬───────┘  └────────┬────────┘  │
│         │                │                    │           │
│  ┌──────┴────────────────┴────────────────────┴────────┐  │
│  │              SUBSTRATE  (shared, always on)          │  │
│  │  vault on disk · event bus · permission model        │  │
│  │  job scheduler · crash isolation per plane           │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

**What the boundaries buy** — this is the substantive part, and it is why this is not just "one big app":

- **A plane can die without taking the keyboard.** The IME service is a separate Android service bound to the input path; it holds no agent state, so an agent crash never shows the user a dead keyboard. This is a scheduling problem with a scheduling solution.
- **The keyboard never waits on the agent.** Input handling and agent execution run on different threads with a queue between them. The keyboard's job is keystrokes; when the user invokes the agent, it hands off an intent and returns immediately.
- **Memory is the substrate, not a feature of any plane.** One vault, one index, addressed by path. The agent reads it, the keyboard suggests from it, the browser files into it. That is the "quality leap" that a composite cannot produce, because a composite has four partial views of you.
- **The scheduler is where the value is.** Knowing that the user is mid-sentence in WeChat, that an agent task is 40% done on another display, and that a note was just edited — and deciding what gets CPU and when — is a thing only an integrated runtime can do.

That last point is the whole argument for convergence, and it is the one thing the four-currently-separate cannot do at any level of engineering effort.

---

## What each piece already provides (verified, not assumed)

This section is written from reading the source. It is a survey of **what to take**, not a list of reasons to keep them separate.

### Mika — the hands

- Android app, Kotlin, Apache-2.0. ~2.9k stars at the time of writing.
- **Virtual display**: tasks operate apps on a separate display, sized like the main screen. Since Android 14 that display has its own focus, so the user keeps typing on the main screen. The target app is muted while the agent works, and unmuted when handed back.
- **Built-in phone assistant**: a shell-privileged helper process started over the phone's own wireless debugging. They implemented the ADB protocol themselves, including RSA-2048 pairing keys and a TLS handshake, in `core/adb/`.
- **Accessibility** for reading the view tree and injecting gestures, with a whole hierarchy of fallbacks (view tree → OCR → vision model) when the tree is not usable.
- **Linux sandbox**: PRoot + Alpine 3.22, shipped inside the APK, unpacked on first launch. Not network-enabled by default.
- **Built-in browser**: WebView on a private virtual display, driven over the Chrome DevTools Protocol. Their own tab pool, their own download handling, their own page-read scripts.
- **Actor model**: `:core` is pure Kotlin with no Android dependency, so the agent loop, tools, context compression, verification and memory are all testable on a desktop JVM. `:app` is the Android host.
- **A real HTTP API**: `/api/v1/tasks` with Bearer auth — `POST /tasks`, `GET /tasks/{id}`, `wait`, `answer`, `steer`, `cancel`. Also an MCP server mode (USB or LAN).

The most impressive part is not any single feature. It is the accumulated detail: what happens to an app that plays audio when the display is torn down; how to move a task back to the main screen when an app opens itself there from a background service; how to corrupt nothing when the process is killed mid-task. That is years of work compressed into a codebase.

### Minis — the workspace

- The environment this document is being written in: a Linux userland on the phone, with a real shell, a browser, a file system, and the ability to install packages.
- Where file manipulation, scripted automation, and long-form work actually happen.

### wmkeyboard — the entry point

- Android IME, Kotlin, MIT. ~680 stars.
- Rime engine for Chinese input, with the full machinery (schemes, dictionaries, user layers) that implies.
- **A broadcast interface for text injection** (`com.wasimaster.wmkeyboard.action.TYPE_TEXT`), which already makes it drivable by other apps. This is the seam the composite needs.
- Clip entities, AI continuation, per-language layout rules — the keyboard is already an intelligent surface, not a key grid.

### Obsidian — the memory

- Closed source. This matters and is discussed below.
- Local Markdown vault, plugin host with a real API, graph and link semantics that make a knowledge base navigable.
- What it gives the composite is the *shape* of memory: plain files you own, addressed by path, extended by code you can write.

---

## The four seams

A composite is only real if the parts can talk. Here is where they already do, and where they do not yet.

### 1. Keyboard → Agent

**Status: possible today, not wired.**

wmkeyboard already accepts a text-injection broadcast with a delivery receipt. An agent can put text on the keyboard without touching the clipboard — which matters, because wmkeyboard reads the clipboard and turns OTP codes into chips, so writing to it conflicts.

What is missing is the other direction: the keyboard does not know an agent exists. Pressing and holding a key does not summon one. There is no protocol for "this user is speaking to the agent, do not treat this as ordinary typing."

### 2. Agent → Device

**Status: solved, by Mika.**

This is the part everyone else gets wrong, and Mika gets right. Solved enough that it should be treated as infrastructure rather than reimplemented.

### 3. Agent → Sandbox

**Status: solved twice, independently.**

Mika ships PRoot + Alpine. Minis *is* PRoot + Alpine. That is duplicated effort, but it also means the pattern is proven.

### 4. Agent ↔ Memory

**Status: the real gap.**

Obsidian's plugin API is the closest thing to a standard here, and it is closed source. The honest options:

- **Borrow the shape, not the code**: a Markdown vault on disk, a plugin API modeled on the same ideas, implemented from scratch on something open (CodeMirror 6 is the obvious base for the editor).
- **Treat Obsidian as an external service**: keep it as the memory store, talk to it through its plugin API from outside. This works today and requires no new editor.
- **Declare memory as files and nothing more**: any Markdown vault, any editor. Least coupling, least capability.

I lean toward the second for personal use and the first if this ever needs to be distributable.

---

## The licences, stated plainly

What can actually be taken as code, and what can only be taken as ideas:

| Project | Licence | What it means |
|---|---|---|
| Mika | Apache-2.0 | Code can be taken, modified, redistributed, closed. Attribution required. |
| wmkeyboard | MIT | Code can be taken with no meaningful constraint. |
| **Minis / OpenMinis** | **GPL-3.0** | Code can be taken, but the derived work must be released under GPL. |
| **Obsidian** | **Closed source** | **No code can be taken.** Ideas only. |

Two consequences, and they are the hard constraints on this project:

**Obsidian is the one gap that must be filled from scratch.** The vault can be Markdown on disk — that part is trivial. The plugin host cannot be borrowed at all. The replacement is CodeMirror 6 plus a plugin API designed from the same ideas. That is real work, and it is the largest single item in the build, because a plugin ecosystem is not a file format, it is a set of promises about what extension is allowed to do.

**GPL-3.0 decides what this can become.** If the Minis/OpenMinis sandbox layer is taken as code, the whole thing is GPL. For personal use this is irrelevant. For distribution it decides the licence of everything upstream of it.

The resolution that keeps the option open: **the substrate and the planes are written clean; the sandbox is taken as an external binary or reimplemented over PRoot + Alpine directly**, which are both open and carry no such constraint. That costs some work now and preserves the choice later.

---

## The plan

There are two dates, and they do different jobs.

**Now, to March 2027 — assemble, don't build.** The four run side by side. No new runtime. The purpose of this period is not to ship anything; it is to record, from daily use, exactly which behaviours are wrong. That record is the specification. Every complaint written down now becomes an acceptance test later, and a complaint nobody wrote down becomes a bug that ships.

**1 March 2027 — the check.** If something exists that is simultaneously keyboard, memory, hands and workspace, the work is unnecessary. If not, it starts.

**The build, when it starts — order matters:**

1. **Substrate first.** Vault on disk, event bus, job scheduler, permission model. This is the part that makes the result better than four apps rather than worse. Building planes before the substrate produces exactly the tangle the four already are.
2. **Input plane second.** The IME is the surface the user touches constantly, so it is the fastest place to discover whether the shared substrate actually helps. It is also the smallest plane.
3. **Mind plane third.** Agent loop, context, tools. Port the ideas from Mika's `:core` — it is already pure Kotlin with no Android dependency, which is the correct shape and worth copying.
4. **Action plane last.** Virtual display, accessibility, shell helper, browser. This is the least rewarding to write and the most rewarding to take, licence permitting.

**On "implement each one myself so none is inferior":** this is the right instinct and the wrong plan, and the distinction is worth stating. Reimplementing all four to parity is four years. Reimplementing them *to the shape the substrate needs* is much less, because the substrate removes work each one currently does alone — the keyboard stops needing its own settings sync, the agent stops needing its own file store, the browser stops needing its own download manager. **The quality leap comes from the deletion, not the addition.** That is the actual answer to "and also make it a quality leap."

---

## What "stronger" will look like, and why it may not close this

My honest expectation is that by March 2027 something considerably stronger will exist. Betting against that would be stupid; the field moves too fast for the opposite position to be credible.

But "stronger" is not the same shape as "unified", and the difference is the whole point of the deadline. So the check has to be specific. Four tests, in the order they are likely to be passed:

**Test 1 — Does anything ship an input-method surface?** This is the cheapest test and the most likely to pass early. An assistant that lives in the keyboard is a small feature for anyone who already has an assistant. Expect this one to be satisfied first, and expect it to mean almost nothing on its own: a keyboard that can summon a chatbot is not an input plane, and it certainly is not a shared substrate.

**Test 2 — Does anything operate real apps on the user's own device, with real logged-in accounts?** Mika already does. Others will. Also not sufficient — plenty of things can drive a phone and remember nothing.

**Test 3 — Does anything own the memory as plain files the user can read, edit, diff and version?** This is where most projects quietly fail the whole test. The commercial incentive is the opposite: a memory you can export is a memory you can leave with. So this is the test that distinguishes a personal agent from a subscription — and the one that a funded competitor has the strongest reason to fail.

**Test 4 — Do all four happen in one runtime, with one scheduler that knows all four states?** This is the only test that actually matters and the least likely to pass. Every item above can be true of four separate products. Test 4 asks whether one scheduler knows that the user is mid-sentence, that a task is 40% done on another display, and that a note was just edited — and resolves the three against each other. Nobody is close, and nobody's roadmap is shaped to get there, because getting there requires each team to admit their product is one plane.

**If Test 4 passes before March 2027, the work is unnecessary and this repository closes.** Passing Tests 1–3 does not close it — that would be the four-app tangle again, only with better apps.

**And if my prediction is right and something much stronger does appear:** it will almost certainly be stronger *inside one plane* — a better agent, a better keyboard, a better vault. Stronger components are not the thing. Four excellent planes, separately, is exactly the situation this repository exists to describe.

---

## What this repository is not

- Not a fork of any of the four.
- Not a request for anyone to build it.
- Not a promise that it will work.
- Not a claim that these four are the best of their kind. They are the four that are simultaneously good, open, and already installed — which is a different and much weaker statement.
- Not a claim that the field is standing still. It is not. The bet is narrower than that: **convergence on all four axes at once is the slow one, and it is the one that matters.**

It is a written-down bet, with a date on it.

---

## References

- Mika — https://github.com/wangzan101/Mika (Apache-2.0, Kotlin)
- OpenMinis — https://github.com/OpenMinis/OpenMinis (GPL-3.0, Swift + Kotlin)
- wmkeyboard — https://github.com/wasi-master/wmkeyboard (MIT, Kotlin)
- Obsidian — https://obsidian.md (closed source)

*Written 2026-11-01. Revisit 2027-03-01.*
