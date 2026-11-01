# Jarvis — A Composite Personal Agent, Built From Four Tools That Already Exist

> **Status: a proposal, not a product.** Nothing here is built. This repository exists to write the idea down, keep it honest, and check back later. If the four pieces have not merged into one thing by **March 2027**, I will build the composite myself.

---

## The claim

Four open-source projects, each already good at one thing, together cover almost the whole surface of what a personal AI agent should be:

| Piece | What it does well |
|---|---|
| **Obsidian** | Your knowledge lives locally, in plain Markdown, extended by plugins you own |
| **Mika** | Real device control — a hidden virtual display, apps operated the way you operate them, a built-in terminal and browser |
| **Minis** | A Linux sandbox on the phone with a terminal, a browser, and a file system that behaves like a real machine |
| **wmkeyboard** | The one surface that is always in front of you. Speak to your agent from the keyboard, in any app, without leaving what you were typing |

Alone, each is a fragment. **Global shortcuts that ignore the app you are in. Local-first memory. Real device control. A real shell.** Together they describe one thing: a personal agent that is always reachable, remembers everything, and can actually do things.

**The bet: before brain-computer interfaces, smart glasses and general-purpose robots become ordinary, this composite is the most capable and most needed thing a person can build for themselves.**

That is the entire thesis. Everything below is either evidence for it or an honest account of what breaks.

---

## Why a composite and not one app

The instinct is to merge all four into a single application. I think that instinct is wrong, and here is the structural reason:

**Their engineering constraints contradict each other.**

- A keyboard must be resident, tiny, and never stall. It is in the input path of every keystroke on the phone.
- An agent must run long, hold large contexts, and survive being backgrounded.
- A notes app must open instantly and never lose a character.
- A virtual display needs system-level privileges and owns a whole display context.

Fuse them into one process and each one's requirement sabotages the others. The keyboard waits on the agent. The notes app dies when the agent's service is reclaimed. A crash in the virtual display takes your input method with it.

**So the composite is not a merge. It is a composition:** four processes, one shared contract, and a thin orchestration layer that speaks to all four. That is the thing worth building, and it is small.

---

## What each piece already provides (verified, not assumed)

This section is written from reading the source, not from marketing pages.

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

## What is actually missing

Honesty section. A composite of four tools is not a product, and pretending otherwise wastes everyone's time.

1. **No shared identity or permissions model.** Four apps, four sets of permissions, four background-execution policies, four ways to get killed by the OS.
2. **No shared context.** The keyboard does not know what the agent is working on. The notes app does not know what you just asked. Each one holds a fragment.
3. **No shared memory of failure.** Each app learns your behavior separately, and forgets it separately.
4. **Background survival is a per-app fight.** Every one of these four has its own battery-optimization exemption story, and they conflict.
5. **Cost of integration is not zero.** Every seam above is a protocol someone has to define, version, and keep working across four release cadences.

---

## The licences, stated plainly

Anybody who builds on this needs to see this table:

| Project | Licence | What it means |
|---|---|---|
| Mika | Apache-2.0 | Permissive. Can be modified and redistributed, including in closed form, with attribution. |
| wmkeyboard | MIT | Maximally permissive. |
| OpenMinis | GPL-3.0 | Copyleft. Modifications must be released under GPL. |
| Obsidian | Closed source | **Cannot be reused as code.** Only its ideas are available. |

For personal use, none of this matters. For distribution, the GPL third and the closed-source fourth constrain the design before a line is written.

---

## The plan

**Now (before March 2027):** use the four together as they are. No new code. Learn where the seams hurt by living in them.

**March 2027:** check again. If a genuine unified personal agent has appeared — one thing that is simultaneously shortcut, memory, hands and workspace — then the composite was an interim answer and should be retired.

**If it has not appeared:** build the composite. The scope, as it stands today, is one thin orchestration layer:

- a contract that lets the keyboard summon the agent and hand it intent
- a bridge that lets the agent read and write the vault through the existing plugin API
- a dispatcher that turns a spoken request into a task, and a task result back into something the keyboard can show

Roughly: a few thousand lines, not a new agent. The hard parts — device control, sandbox, browser, input method — are already built by other people and should stay theirs.

---

## What this repository is not

- Not a fork of any of the four.
- Not a request for anyone to build it.
- Not a promise that it will work.
- Not a claim that these four are the best of their kind. They are the four that are simultaneously good, open, and already installed.

It is a written-down bet, with a date on it.

---

## References

- Mika — https://github.com/wangzan101/Mika (Apache-2.0, Kotlin)
- OpenMinis — https://github.com/OpenMinis/OpenMinis (GPL-3.0, Swift + Kotlin)
- wmkeyboard — https://github.com/wasi-master/wmkeyboard (MIT, Kotlin)
- Obsidian — https://obsidian.md (closed source)

*Written 2026-11-01. Revisit 2027-03-01.*
