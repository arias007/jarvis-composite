# The Four Seams

A composite only exists where the parts can talk. This file tracks what each seam needs, what it has today, and what blocks it.

Status values: **open** (nothing exists) · **partial** (one direction works) · **solved** (works today, should be reused, not rebuilt).

---

## Seam 1 — Keyboard → Agent

**Status: partial**

### What exists

wmkeyboard accepts an inbound text-injection broadcast with a real delivery receipt:

```
am broadcast -a com.wasimaster.wmkeyboard.action.TYPE_TEXT \
  --es text '...' -p com.wasimaster.wmkeyboard
```

The intent is scoped to the package, and the result reports `data="typed"` when the text actually landed in the field. This is a usable one-way channel *into* the input path.

### What is missing

- **No outbound channel.** The keyboard has no way to say "the user invoked the agent" — no broadcast, no bound service, no content provider.
- **No mode negotiation.** While the agent is composing a reply, the keyboard has no concept of "not mine to interpret." Any keystroke is a keystroke.
- **No clipboard discipline.** wmkeyboard monitors the clipboard to build OTP chips. An agent that writes the clipboard fights it. The known workaround is to inject text directly rather than copy-paste, but that is a convention, not a protocol.
- **No result surface.** The agent has no place to show progress that is not a notification or a separate app.

### What the seam needs

1. An invocation intent the keyboard fires with a payload (text, selection, current app, timestamp).
2. An agent-side receiver that acknowledges and starts a task.
3. A return path so the keyboard can render "working / done / needs you" inline, without switching apps.
4. A shared rule about who owns the clipboard at any moment.

---

## Seam 2 — Agent → Device

**Status: solved**

Mika has this. Do not rebuild it.

- Virtual display with its own focus (Android 14+)
- Accessibility view tree with OCR and vision fallbacks
- Shell helper over the phone's own wireless debugging, with their own ADB implementation
- Per-app mute while working, restore on handback
- Task lifecycle: apps after task end, main-screen spill detection, takeover for login and payments

Integration point: Mika exposes `/api/v1/tasks` with Bearer auth, plus an MCP server mode. Either is sufficient to drive it from outside.

---

## Seam 3 — Agent → Sandbox

**Status: solved, twice**

Both Mika and Minis ship PRoot + Alpine on arm64. Duplication, but a proven pattern.

Notable detail from Mika: the sandbox is network-off by default, and commands inside it can call back into the phone tools (`mika phone <op> '<json>'`) over a per-run socket with a one-time token. That bridge design is worth copying if this is ever rebuilt.

---

## Seam 4 — Agent ↔ Memory

**Status: open**

The hardest seam, and the one with a licence problem attached.

Obsidian's plugin API is the natural contract, but Obsidian is closed source — the API can be *used*, not *reimplemented from its code*.

Three options, ordered by how much new code they require:

1. **External service (least code).** Keep Obsidian as the memory store. Reach it through its plugin API from outside the app. Works today.
2. **Model the shape on open foundations (most code).** Markdown vault on disk, plugin host built over CodeMirror 6, API designed from the same ideas rather than from the source.
3. **Files and nothing more (least capability).** Any Markdown vault, any editor, no plugin host. Maximum decoupling.

### What the seam actually needs to carry

- Read and write notes by path
- Query by tag, link, frontmatter
- Deliver "current context" to the agent without the agent crawling the whole vault
- Let a plugin extend the agent's tools without forking the agent

That last one is the interesting requirement, and none of the four currently support it.

---

## Cross-cutting problems no single seam owns

1. **Identity.** Four apps, four accounts-of-self. Whose memory is whose.
2. **Background survival.** Four separate fights with the OS. They will not agree on policy.
3. **Permissions.** The keyboard needs input access; the agent needs accessibility and shell; the vault needs file access. No shared threat model.
4. **Release cadence.** Four upstreams moving independently. Any contract defined here has to survive being broken by someone else's minor version.

These are the reasons this repository is a written proposal and not a build.
