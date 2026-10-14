# Agent Note: A resumed log protects the system message its surface gains

Status: implemented

English | [中文](2026-10-15-resumed-log-system-prompt-head.zh.md)

## Problem

A stored v4 session whose history was written before the format recorded system messages is resumed by the current loop. On resume the loop replays the stored events as its seed, and `SystemPromptProjection.systemNodes()` finds no surviving `system/message` on that surface, so the step's first system-prompt commit is an `append` (`packages/core/agent-loop/src/agent.ts`) that lands after the replayed user, assistant, and tool nodes. The native V4 relationship pass refused any `system/message` that arrives once the surface already holds nodes, and required the protected system head to be the surface's first node, so it rejected the whole artifact: `SessionFormatError: system/message requires a protected first surface head`, surfaced as `stored log is corrupt`. The writer had produced a log its own reader refuses, and every later append kept it refused. A three-turn COMSOL work session written by the older writer (948 events, `~/.dsh/sessions/--F-COMSOL64--/session-1b68db67-…`) failed exactly this way on 2026-10-14; the session layer (`dsh-session` `foldSurface`) accepted the same log, so only the native V4 relationship pass rejected it. Nothing about the stored artifact is malformed: the surface's system node is simply not its first node.

## Decision

`packages/session/session-format-v3-to-v4/src/relationships.ts` accepts a `system/message` that joins a surface already holding history. The first `system/message` to join the surface becomes the protected node, whether it is appended anywhere or introduced by a replacement while no protected node exists; `protectedHead` records that node and the existing protection rules apply unchanged — only a `system/message` covering exactly that node may replace it, and compaction may never shadow it. Surface order is otherwise untouched, so a mid-surface system node keeps its recorded position and the model-visible history stays as written. Retrieval needs no change: `SystemPromptProjection.systemNodes()` reads surviving surface nodes in surface order, so the resumed loop finds the appended node by seq and replaces it when the rendered prompt next differs. The V2-to-V3 migration keeps its own refusal, because that pass synthesizes a head while migrating and refuses a V2 surface preceding the first step rather than reordering chronology; native V4 reads have no chronology to change. `SESSION_FORMAT_VERSION` stays 4: no stored field, envelope, or schema changes.

## Alternatives considered

**Refuse the log and fix the writer instead.** Rejected: an append-only log cannot place the system prompt at surface position 0 once earlier history exists, so refusing the append leaves either a model-visible system prompt with no logged event (breaking model-visible ⟺ logged) or a resumed session that silently runs without one.

**Accept the append but keep refusing a replacement that introduces the first system node.** Rejected: two paths would then produce a system node with different protection, and a replacement that introduces one leaves the surface carrying an unprotected system node; establishing the protected node on the first system message from either path keeps one invariant.

**Rewrite or migrate the stored log instead of widening the read.** Rejected: a read pass must not rewrite stored artifacts, and the artifact is otherwise valid, so a heading fix belongs in validation rather than in a migration that would have to renumber history.

**Also relax the protection rules, so compaction may shadow the node.** Rejected: protection is what keeps a session's system prompt recoverable across compaction; only the position requirement was wrong, not the protection.

## Consequences

Reading is strictly wider: every log that loaded before still loads, and a log whose surface gains its system message after earlier nodes now loads with that message protected from compaction and from any non-system replacement. A resumed session written by the older writer can be read and continued, and its next prompt change replaces the protected node by seq, so the surface keeps exactly one protected prompt node. The cost is that a stored artifact may now present the session's system prompt anywhere in surface order, including between earlier conversation nodes; consumers that assumed the system message leads the derived messages must read surface order instead. The protection guarantee is unchanged in strength: the node compaction and non-system replacements may not shadow is now whichever system message the surface first gained.

## Testing

`packages/session/session-format-v3-to-v4/tests/relationships.spec.ts` moves the previously rejected case — user input followed by a system message — into an accepted one: the log reopens, its derived messages are user then system, a `compaction/prune` shadowing that node throws `/protected system head/`, a non-system replacement of it throws the same, a system replacement over exactly it is accepted, and a replacement that introduces the surface's only system message is accepted. The remaining malformed-relationship cases still throw, so the widening did not blunt the range, reference, and protection checks. `packages/session/session-persistence-jsonl/tests/current-event-admission.spec.ts` stores a log whose surface is user input followed by the system message and asserts it opens for read and write, that the append after it leaves the existing generation bytes untouched, and that a reopen yields every event. `packages/session/session-persistence-jsonl/tests/native-developer-admission.spec.ts` keeps refusing a developer replacement of the protected system node.
