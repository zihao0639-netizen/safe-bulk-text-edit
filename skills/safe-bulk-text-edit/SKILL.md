---
name: safe-bulk-text-edit
description: 'The standard posture for bulk-editing text across many files, one rule at N sites. Fixes four observed failures: the script dies on a quote SyntaxError while an earlier rename in the same command landed, so disk and ledger drift apart; the edit stops halfway; the anchor prefix is too short and hits a same-named function; the verification command silently returns empty, so not-verified reads as fine. Seven rules: the heredoc delimiter must differ in quote type from the body and never appear inside it; locate by line prefix, not whole-line match; assert uniqueness per site, validate all, write once for atomicity; never mix file operations with a scripted edit in one command; verify with grep -E, never python -c inside bash -c; any checker, gate or guard must pass a fail-closed self-test; any script that writes must read the file back. Ships a dry-run flow, rollback matrix and checklist. Use before a multi-file loop, or when two sides of an edit stop agreeing.'
license: MIT
compatibility: 'Needs macOS or Linux with a POSIX shell, git, python3 and BSD or GNU grep on PATH. Runs in Claude Code, Claude.ai, Codex, Cursor and any SKILL.md-compatible agent with shell and file-write access.'
metadata:
  author: zihao0639-netizen
  version: '1.0.0'
---

# Safe bulk text editing (many files, many sites)

> Source: learned the hard way. Four falls on the **same class of problem** in a single evening, each one leaving the two sides inconsistent.
> Applies to: python heredoc / scripts that rewrite text across several files. **Does not apply** to a small single-file change — plain Edit is cheaper there.

---

## 🔴 The most expensive fall: the script never ran, and you thought it did

This is why the skill exists. Everything else is a variant of it.

```sh
# ❌ written like this before
python3 - <<'PY' ... PY && git mv A B
```

What actually happened:

1. The python inside the heredoc hit a **quote clash** and raised `SyntaxError` — **not one line executed**;
2. but `git mv A B` sat on the **line after the heredoc terminator** — that is a **new command**, **not covered by the preceding `&&`**;
3. so `git mv` **took effect anyway**;
4. the terminal showed mv's output and you reasonably concluded "done";
5. result: **the disk changed, the ledger did not.** It surfaces much later, by which time the intermediate state has polluted a pile of other things.

★ **A shell `&&` chain breaks on the line after the heredoc terminator.**
★ **`NameError`, a mistyped variable and a bad indent hide the same way — `&&` dresses all of them up as "looks like it succeeded".**

✅ The only correct posture: **split it into two commands — edit, confirm success, then mv / rename.**

---

## Four observed failures (self-check against this)

| # | Failure | Surface symptom | What is really happening |
|---|---|---|---|
| ① | **Script dies on a quote SyntaxError, nothing executes** | You think it is done | Not one character changed; the mv/rename in the same command already landed ⇒ disk and ledger drift apart |
| ② | **The edit stops halfway** | Some files look right | An exception mid-run; earlier writes already hit disk ⇒ inconsistent state left behind |
| ③ | **Prefix too short, matches another function** | The edit is done, errors start elsewhere | Measured: `    return {"判": "⚠️" if rows else "✅", "项": rows,` exists in **six checker functions** |
| ④ | **The verification command silently returns empty** | "That key does not exist", "already fixed" | Under BSD grep both `\|` and `\s` **silently return 0 lines** ⇒ not-verified is read as no-problem |

---

## 🔴 Seven postures (each one carries a **price**, not a "best practice")

### 1. Long strings: **the delimiter and the body must use different quote types**

| Form | Result |
|---|---|
| `"a double quote " inside a double-quoted string breaks it"` | ❌ `SyntaxError` |
| `'''...three single quotes written again inside the body...'''` | ❌ closes early, still `SyntaxError` |
| ✅ delimiter `'''` + **every quotation inside written as `「」`** | stable |

★ **The delimiter itself must never appear in the body.** To write "three single quotes", describe them — do not type the three characters.
★ **Non-ASCII content makes the file route mandatory**: always `Write` a script file and run it. Never wrap `python -c` in `bash -c`.

### 2. Locate with a **line prefix, `startswith`** — never whole-line equality

```python
hits = [i for i, l in enumerate(L) if l.startswith("| **23** |")]
```

Whole-line matching forces you to **copy the entire line character by character** (curly quotes, ideographic spaces) ⇒ one character off and it fails.
A prefix only needs **the distinctive head**.

### 3. **Assert uniqueness** at every site; **validate everything first, write once at the end** (atomicity)

```python
assert len(hits) == 1, ("line 23", hits)      # not unique ⇒ raise immediately
...
# every change is computed; one single open(...,"w").write() at the end
```

★ **A mid-run failure then means zero characters changed** — no half state survives.
★ **A prefix that is too short collides** ⇒ **scope to the function or section first**, then search inside it:

```python
j = [k for k,l in enumerate(L) if l.startswith("def check_a3")][0]
nxt = [k for k in range(j+1, len(L)) if L[k].startswith("def ")][0]
seg = [k for k in range(j, nxt) if L[k].startswith("    return ...")]   # search inside the function only
```

### 4. 🔴 **Never put a file or directory operation in the same command as a scripted edit**

✅ **Split into two commands**: **edit → confirm success → then mv / rename**.
(Reason: "the most expensive fall" above.)

### 5. The verification command **must itself be reliable**

| Trap | Reality | Fix |
|---|---|---|
| `grep 'a\|b'` | **Under BSD grep, `\|` alternation silently returns 0 lines** and looks like "no match" | 🔴 **Always `grep -E 'a\|b'`** |
| `grep '^[a-z_]+\s='` | 🔴 **BSD grep does not support `\s`** ⇒ **it also returns 0 lines, silently** (measured; looks like "that key does not exist") | Use `[[:space:]]` / `[ ]*`; **for a small file just `cat` it** — do not grep |
| `cmd \| tail -5; echo $?` | `$?` is **tail's** exit code | `cmd > out.txt 2>&1; echo $?` |
| `bash -c "python3 -c \"…`backtick`…\""` | 🔴 **Backticks inside the double quote layer are treated by the shell as command substitution** ⇒ every value written into the file **comes out empty** (the terminal spits `command not found`, **yet the script keeps running and prints OK**) | **Always `Write` a script file and run that** — never `python -c` inside `bash -c`; non-ASCII content makes this mandatory |
| `A && grep X && B` | grep returning 1 **short-circuits everything after it** | Separate with `;`, or run it on its own |

### 6. Changing a **checker / gate / guard** ⇒ a **fail-closed self-test is mandatory**

Blank the input (`ROOT` pointing at a directory that does not exist) ⇒ **the exit code must be non-zero**.
★ Exit 0 = **fail-open** ("cannot verify" counted as "all passed") ⇒ do not ship it.
★ **State exactly which path the self-test covered**: if you only exercised path A, say path A — **do not stop at "the self-test passed"**.

### 7. 🔴 **Any script that writes a file must carry a read-back check**

Read it back **the moment it is written**; **do not trust the OK you printed yourself**:

```python
save(F, new)
print("read-back:", "✅ written" if "marker" in load(F) else "❌ not written")
```

★ **Price (measured)**: `save()` left undefined ⇒ `NameError` ⇒ **the file was never written**, while the chained `&& git commit` **ran anyway** ⇒ **a silent empty commit** — **you thought you changed it, you did not**.
★ Same family: `NameError` / mistyped variable / bad indent — **every one of them masked by an `&&` chain into "looks like it succeeded"**.

---

## Standard posture: the full flow of one bulk edit

```
1. Anchor        git status --short + git stash / record the current HEAD
                    ↓ no anchor, no rollback
2. Script file   Write a .py file; never python -c inside bash -c
                    ↓ non-ASCII content always goes through a file
3. Dry run       print "would change X to Y at file:line" only, write nothing
                    ↓ assert len(hits)==1 at every site; a too-short prefix means scope to the function first
4. Count check   does the dry-run site count match what you expect? stop on any shortfall
                    ↓ a mismatch means do not proceed
5. Write once    a single write after everything is computed (atomic: mid-run failure = zero changes)
                    ↓
6. Read back     load and confirm the marker is there; if not, roll back at once
                    ↓
7. Separate command  mv / rename / commit in a different command
                    ↓ never in the same one as the edit
8. Closing list  review git diff character by character (not just the stat line)
```

★ **Step 3, the dry run, is the whole value of this flow**: it moves "got it wrong" from after the write to before the first keystroke.

---

## How to verify the change really landed

In this order, **do not skip**:

1. **Read-back** (rule 7) — built into the script, read immediately after writing; never trust your own OK.
2. **Independent command review** — a different command, a different method:
   ```sh
   grep -rn "marker" <target-dir>      # on BSD always add -E for multiple patterns
   ```
   ★ Query with a **different mechanism than the one that made the change**, so a single bug cannot spoil both the edit and the check.
3. **Count sites, do not check existence** — "is there a match" lies to you (silent 0 lines); **count how many there should be**.
4. **Small file? just `cat` it** — skip grep, your eyes are the most reliable judge.
5. **If the target is a checker / gate / guard** — run the fail-closed self-test (rule 6); the exit code must be non-zero.
6. **Walk the closing checklist** item by item (below).

🔴 **A verification command that silently returns empty means not verified — not no problem.**

---

## How to roll back

| Stage | Rollback |
|---|---|
| **Before starting** | Build an anchor: `git stash push -m "before bulk edit"` or record the current HEAD; **no anchor, no rollback** |
| Dry-run stage | Nothing was written; `git checkout -- .`, or just discard the script |
| **After writing, before committing** | `git checkout -- <changed file>` — restore path by path; do not `git checkout -- .` and clobber someone else's work |
| Stash anchor used | `git stash pop`; **run `git stash list` first and confirm the top entry is the one you want** |
| **Already committed** | `git revert <commit>` (keeps history); **never `reset --hard`** — that throws work away |
| **Already mv'd / renamed** | Reverse `git mv` back, and **fix the ledger in the same pass** — disk and ledger roll back together |
| Worst case (no version control) | Copy the tree outside the workspace first; **no copy means do not start a bulk edit** |

★ **Re-run verification after any rollback** to confirm you are back at the pre-edit state.
★ **Disk rolled back and ledger not = fall ① all over again.** Look at both sides together.

---

## ✅ Closing checklist (walk it every time)

- [ ] `git status --short` — **only the intended files** touched? (this also catches other people's changes dragged in)
- [ ] `git diff` — review **every hunk character by character**, not the stat line
- [ ] Is the **same sentence scattered across sites**? Fixed A, missed B? ("patched one spot, missed the whole patch")
- [ ] Did the **trio** (body / ledger / index) **all get updated**?
- [ ] Numeric conclusions: **can they change?** Hard-coded values go stale ⇒ either mark them "measured at the time" or leave them out
- [ ] **Leave same-named files in the archive area alone** (historical snapshots)
- [ ] Checker / gate / guard: **did the fail-closed self-test run**, was the exit code non-zero?
- [ ] Is the rollback anchor still in place? Confirm you can get back

---

## One-line summary

> **No quote clashes · prefix long enough · assert uniqueness · validate before writing · file operations in a separate command · verify with `-E` · read back after writing · anchor before you start.**
