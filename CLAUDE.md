# Global Claude Code config — Joshua

This file loads on every Claude Code Desktop session start, regardless of working directory or project. Keep it short — the actual conventions live in linked files. This is just the bootstrap.

## Multi-session workspace convention

Joshua runs multiple Claude Code Desktop chats in parallel. Each chat is a coordinator-class agent with its own workspace. Before doing meaningful work in any session:

1. **Identify your chat name.** It's the title shown in the Claude Code Desktop Recents pane (e.g. "MVP Workup - Protocol", "Agent N7 - Prototype Build"). If you don't know it, ask Joshua. Slugify by lowercasing + replacing spaces with `-`.

2. **Read your own workspace** at `~/.claude/sessions/<your-slug>/`:
   - `PROFILE.md` — your scope, what you own, what you don't
   - `ACTIVE.md` — what you were working on (left from last run)
   - `JOURNAL.md` — append-only log of significant events
   - `HANDOFF.md` — context from your last pause

3. **Read what other sessions are doing** by scanning `~/.claude/sessions/*/ACTIVE.md`. Don't duplicate work or touch the same repo / worker / file as another session without coordinating.

   **Reconcile stale references to your own work in other sessions' files.** If a sibling session's INBOX or ACTIVE.md references work you've now shipped, clean up the reference (delete the inbox file, update their ACTIVE.md note line) so future sessions don't act on stale state. Stay narrow: only touch references TO YOUR SHIPPED WORK; don't touch unrelated state in sibling sessions.

4. **If the workspace doesn't exist yet**, you're a brand-new chat. Ask Joshua for the chat name, create the workspace dir, populate with starting PROFILE / ACTIVE / JOURNAL / HANDOFF stubs based on the chat's intended scope. Convention spec: `~/.claude/memory/session_workspace_convention.md` (and the same file mirrored in any project memory that loads it).

5. **When claiming a Notion ToDo item** (https://www.notion.so/356f359483b2818c9d94fb891f702547), prepend `[<your-slug>]` to the bullet. When done, replace with `[x]`. The ToDo page is for Joshua's asks; don't put your own session status there.

   **Mirror your `ACTIVE.md` to the dedicated Claude Sessions page** at https://www.notion.so/35bf359483b281a5b350c3290dc124dc whenever your `ACTIVE.md` changes meaningfully — start of work, finish of work, blocked, idle. Find your block keyed by your slug; replace its `**Last seen:** <UTC ISO timestamp>` line and the 1-3 lines of plain-English status that follow. If your block doesn't exist yet, append one at the end. This is how Joshua sees who's doing what across parallel chats without browsing the filesystem.

6. **Always pull dates from `date -u +"%Y-%m-%dT%H:%MZ"`** when timestamping anything (commits, JOURNAL entries, CHANGELOG headers). Claude is unreliable at telling time; pull, don't estimate.

## Pointers

- **Notion ToDo (asks-only list, Claude-maintained):** https://www.notion.so/356f359483b2818c9d94fb891f702547
- **Session workspaces:** `~/.claude/sessions/<chat-slug>/`
- **Convention spec:** `~/.claude/memory/session_workspace_convention.md`
- **Project-scoped memory:** lives at `~/.claude/projects/<project-slug>/memory/MEMORY.md` and loads only when Claude operates in that project's working tree.

## Cross-cutting hard rules

- **Don't deploy to production without explicit approval in chat.** Especially `wrangler deploy`, npm publish, PyPI publish, force pushes, repo-visibility flips.
- **Don't merge `security-hardening-*` branches without Joshua's review.** Sessions leave them as side branches deliberately. The branch's final commit message will say "pending Josh review" or similar.
- **Don't paste secrets into chat.** If a secret is needed, generate it server-side, save it to a temp file Joshua can copy to 1Password, never echo it back.
- **Don't `wrangler delete` a worker without confirming `wrangler whoami` first.** The 2026-05-08 admin-dashboard incident was caused by an account-context confusion.
- **`axis-comments` worker has no git history yet.** Don't commit changes there until it's `git init`-ed and pointed at a destination repo.
