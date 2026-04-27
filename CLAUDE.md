# Personal Claude Code Conventions

These conventions apply to every session, not just specific projects.

## Stream timeout prevention

Anthropic is aware of an ongoing "Stream idle timeout — partial response received"
issue (GitHub issue #47841). The Write tool is by far the most common trigger.
Follow these rules to avoid stalls and, more importantly, to avoid retry loops
that burn quota.

### Tool-call style
1. Parallel tool calls are fine for short, independent operations — reads,
   searches, quick bash commands. Use them; they rarely cause stalls.
2. Don't bundle multiple heavy operations into a single parallel batch. If a
   step involves a large Write or a long-running command, run it on its own
   rather than alongside other heavy work.
3. The Write tool on larger files is by far the most common stall trigger,
   but any single tool call that produces a lot of output can occasionally
   stall — keep individual calls focused.

### Write tool rules
4. Never use the Write tool for a file longer than ~150 lines in a single call.
   For anything longer, use Bash with a heredoc, appended in small chunks. For
   example:

       cat << 'EOF' > path/to/file
       ...first chunk...
       EOF
       cat << 'EOF' >> path/to/file
       ...next chunk...
       EOF

5. First failure = switch, don't retry. If a Write call stalls or returns a
   stream-idle / partial-response error even once, do NOT retry Write.
   Immediately fall back to Bash heredoc (or Edit for modifying an existing
   file). Never loop on a failing Write — that's the quota-burning failure mode.
6. For creating brand-new files where the content is non-trivial in size or
   contains tricky characters, default to heredoc from the start rather than
   trying Write first.

### Reads & searches
7. When reading large files, use the offset and limit parameters rather than
   reading the whole file.
8. Avoid commands that dump hundreds of lines of output (broad grep, full
   recursive ls, unfiltered find). Narrow with paths, file types, or head.

### Session hygiene
9. If the conversation gets long (20+ tool calls) and responses start to feel
   sluggish, suggest starting a fresh session. A long context window makes
   stalls more likely.

### Subagents and the Agent tool
10. Subagents hit the same stream-idle issue and have the same Write-tool failure
   mode. When spawning an agent (Agent tool / subagent), include a brief
   reminder of these rules in the agent's prompt — at minimum:
   "Follow the stream-timeout-prevention rules: cap Write at ~150 lines,
   prefer Bash heredoc for larger files, and never retry a stalled Write —
   switch to heredoc immediately."
11. Prefer giving subagents narrowly scoped tasks (research, single-file edits)
    rather than broad multi-file generation tasks, since broad tasks are more
    likely to involve large Writes.
