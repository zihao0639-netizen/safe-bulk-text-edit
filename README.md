# safe-bulk-text-edit

An agent skill for the moment you have to rewrite the same text across many files with a script — one rule, N landing points — and the edit has to land atomically, verifiably, and without leaving the disk and the index out of sync.

**Core insight:** the expensive failure is not a wrong edit. It is a script that never ran at all while an earlier `git mv` in the same command already landed, because a shell `&&` chain breaks at the line after the heredoc terminator — `SyntaxError`, `NameError` and a bad indent all get dressed up as "looks like it succeeded". So: prove the script ran before you let anything else in the command touch the tree; anchor on a line prefix instead of a copied whole line; assert uniqueness at every site and write once at the end so a mid-run failure changes nothing; keep file operations in a separate command; and read the file back, because the OK you printed yourself is not evidence.

## Install

```bash
npx skills add zihao0639-netizen/safe-bulk-text-edit
```

Or copy `skills/safe-bulk-text-edit/` into your agent's skills directory manually.

Works with Claude Code, Claude.ai, Codex, Cursor and any SKILL.md-compatible agent with shell access.

## What it covers

1. **The four observed failures** — script never executed but a rename did; edit stops halfway; prefix collides with a same-named function; verification command silently returns empty. Each with its surface symptom and what is actually happening
2. **Quote discipline** — the heredoc delimiter must use a different quote type from the body and must never appear inside it; non-ASCII content goes through a written script file, never `python -c` inside `bash -c`
3. **Anchoring without collisions** — locate by line prefix with `startswith`, scope to the enclosing function or section first, assert the match count is exactly 1
4. **Atomicity** — compute every change, validate every site, then a single write; a mid-run failure leaves zero characters changed
5. **Command separation** — file and directory operations never share a command with a scripted edit; edit, confirm, then move
6. **Reliable verification** — `grep -E` on BSD, `[[:space:]]` instead of `\s`, capture the real exit code instead of `tail`'s, and never put a `grep` in the middle of an `&&` chain
7. **Fail-closed and read-back** — blank the input and require a non-zero exit for any checker, gate or guard; read every written file back instead of trusting your own `print("OK")`
8. **Operations** — a dry-run-first workflow, a rollback matrix per stage (anchor / dry run / written / committed / renamed), and a closing checklist

## Requirements

- macOS or Linux
- A POSIX shell, `git`, `python3`, and BSD or GNU `grep` on PATH
- An agent that can run shell commands and write files

## Security notes

- Every example is local: it reads and writes files inside the repository the user already owns
- No credentials, no network calls, no environment variables touched
- The skill never runs `git reset --hard`, never force-pushes, and never discards uncommitted work as a shortcut
- Destructive steps (`mv`, rename, `git revert`) are always spelled out as separate, reviewable commands

## License

MIT — see [LICENSE](LICENSE).
