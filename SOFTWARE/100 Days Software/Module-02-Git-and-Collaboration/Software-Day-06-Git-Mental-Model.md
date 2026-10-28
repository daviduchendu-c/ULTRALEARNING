# Day 6: Git Mental Model

*100 Days of Software Engineering, Module 2: Git and Collaboration*

**Time:** about 3 hours
**Level:** beginner. You should be comfortable in a terminal (Day 4). No Git experience is needed.

**By the end of this day, you will have:**

- A clear picture of Git's four areas (working tree, staging area, local repository and remote) and the commands that move content between them
- The ability to read a commit hash, a `git log` graph and a `git diff`, and say what each one means
- A repo with a `.gitignore`, and the habit of undoing a mistake with `git restore`
- A small Git-tracked project pushed to GitHub with a history you can explain commit by commit

**What you need:**

- Git 2.23 or newer (this lesson was tested with Git 2.55.0, from `git --version`). Version 2.23 added `git switch` and `git restore`.
- A terminal: Ubuntu on WSL 2 (Day 3), Git Bash, or PowerShell. The commands below work in all three.
- A code editor that opens from the terminal (VS Code from Day 3, or `nano`)
- A free GitHub account

**Prerequisites:**

- Day 4: Linux CLI I (`cd`, `ls`, `cat`, and `man`)
- Day 3: Developer Environment (PATH, absolute and relative paths)

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [The working tree](#the-working-tree)
- [Staging](#staging)
- [Commits](#commits)
- [Hashes](#hashes)
- [History](#history)
- [Diff](#diff)
- [Restore](#restore)
- [.gitignore](#gitignore)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: a Git-tracked greeting tool with a readable history](#project-a-git-tracked-greeting-tool-with-a-readable-history)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Every team needs to answer three questions about its code: what changed, who changed it, and how to get back to the version that worked. Git answers all three with one data structure, and it does so on your own machine, offline, before anyone else sees your work. Most developer and DevOps job postings expect daily Git use, and a take-home coding task is usually delivered as a Git repository.

Git feels hard because its commands are verbs (`add`, `commit`, `reset`, `checkout`, `merge`) and each one quietly touches a different part of the system. Learn the parts first and the verbs become predictable. Today you learn the areas Git keeps, the core ideas behind every command, and the habit of checking what Git sees before you change anything.

[Wikipedia: Version control](https://en.wikipedia.org/wiki/Version_control) gives the general idea. [Wikipedia: Git](https://en.wikipedia.org/wiki/Git) covers the tool's history, including that Linus Torvalds wrote the first version in 2005 for the Linux kernel project.

---

## Glossary

- **Repository (repo):** a project folder plus the hidden `.git` folder that stores its history.
- **Working tree:** the files you see and edit in the project folder.
- **Staging area (index):** the list of changes that will go into the next commit.
- **Commit:** a saved snapshot of the staged files, with a message, an author, a time and a parent commit.
- **Hash:** a fixed-length string that identifies a commit, file or folder by its content.
- **Branch:** a movable name that points at one commit.
- **HEAD:** Git's name for the commit you are currently on.
- **History:** the chain of commits reached by following each commit's parent.
- **Diff:** a line-by-line comparison between two versions of content.
- **Restore:** copying content from the staging area or an older commit back into the working tree.
- **.gitignore:** a file that lists paths Git should not track.

---

## The working tree

Picture a workbench. Parts sit on it in various states of assembly. The bench is the working tree. Changes on the bench are real, but Git has not recorded them yet, so they are not in the project's history.

Git keeps four areas, and you move content between them with explicit commands: the working tree, the staging area, the local repository (the `.git` folder), and the remote repository on GitHub (Day 8). Only the working tree is visible in File Explorer or your editor. The other three live inside `.git` or on the server.

```
 edit files      git add        git commit        git push
 ──────────►  Working   ──►  Staging   ──►  Local     ──►  Remote
              tree          area          repo (.git)      (GitHub)
```

Create a repo and inspect it:

```bash
mkdir git-practice
cd git-practice
git init -b main
echo "first draft" > notes.txt
git status
```

`git status` reports `No commits yet` and lists `notes.txt` as untracked. Git can see the file, but it has not been asked to track it.

The common mistake is treating the working tree as a backup. Edits that exist only on the bench are lost if a command discards them, and no command in Git will bring back an untracked file. Commit early, and run `git status` after any surprise.



---

## Staging

Staging is choosing what goes into the next commit. It is like packing a delivery box. Only the items you pick go in, and you can take some out before you seal the box.

`git add` copies the current state of a file into the staging area. The staging area holds exactly what the next commit will contain. Changes that are not staged stay in the working tree and are not committed.

```bash
echo "second draft" >> notes.txt
echo "unrelated" > scratch.txt
git add notes.txt
git status --short
```

Output:

```
M  notes.txt
?? scratch.txt
```

The first column is the staging area, and the second column is the working tree. `M` in the first column means a staged change to a tracked file. `??` means an untracked file. `scratch.txt` stays out of the commit until you add it.

The common mistake is `git add .` in a folder that holds secrets, binaries or build output. It stages everything beneath the current folder. Check `git status` first, and let `.gitignore` (covered below) handle files that should never be tracked. A second mistake is expecting `git commit -m` to include unstaged edits. It does not, unless you pass `-a`, which stages all tracked files and skips new ones.

In practice, the staging area is how a developer builds a clean pull request. Each commit contains a deliberate choice of what to stage, and reviewers read the result.

---

## Commits

A commit is a labelled snapshot in a photo album. It records the full content of every tracked file (stored efficiently, so unchanged files are not copied again), the author and time, a message, and one or more parent commits. The parent links are what turn commits into a history.

```bash
git add notes.txt
git commit -m "Add notes file with first draft"
```

Git prints a summary such as `[main (root-commit) 5e1d437] Add notes file with first draft`. The phrase `root-commit` means this commit has no parent, so it is the first one. Your hash will differ.

Write the first line of a message as a short summary, under about 72 characters, that explains the change. "fix" tells a reader nothing. "Stop the parser crashing on empty input" tells them what changed. Small commits with clear messages are easy to review, easy to revert, and easy to search in `git log`.

Commits stay local until you push them, and they are permanent in the sense that a later commit does not erase an earlier one. If you commit a password and delete it in the next commit, the password is still in the history. Day 7 covers how to rewrite history, and why that is a team decision.

---

## Hashes

A hash is a fingerprint. A hash function takes any input and returns a fixed-length string. The same input always gives the same output, and a small change in the input gives a completely different output. [Wikipedia: Hash function](https://en.wikipedia.org/wiki/Hash_function) explains the general idea.

Git names every commit, file version and folder snapshot by a hash. Git uses SHA-1 by default (40 hexadecimal characters), and newer repositories can use SHA-256. SHA-1 has known weaknesses against deliberate collisions, and Git includes countermeasures for those, so treat the hash as a reliable identity for everyday work.

```bash
git rev-parse HEAD
git rev-parse --short HEAD
git log -1 --format='%H %h %s'
```

The full hash is 40 characters. The short form is the first seven, and Git lengthens it when it would be ambiguous in your repository. Two clones that agree on a commit's hash agree on its content and its whole history behind it, because a commit's hash covers its parent.

The common mistake is copying a short hash from chat and assuming it is unique. In scripts and CI, use the full hash. Hashes also appear in container image digests, package lock files, and the commit SHA shown on each pull request in GitHub.

---

## History

History is the chain of commits, read backward from `HEAD`. Each commit points at its parent, so `git log` walks the chain from the newest commit to the oldest.

```bash
git log --oneline
git log --oneline --graph --decorate
```

`--oneline` prints one line per commit. `--graph --decorate` draws branches as text and labels the branch names. After a second branch splits off, the output looks like this:

```
* 4b7e1a2 (HEAD -> main) Tighten greeting wording
* 9c03d18 Add the second draft
| * 2f6d8b0 (feature/shout) Add shout option
|/
* 5e1d437 Add notes file with first draft
```

Read it from the bottom up. The root commit is oldest, two branches split from it, and `HEAD` points at `main`. Use `git show --stat <hash>` to see the files a commit touched, and `git log --follow -- notes.txt` to follow one file across renames.

The common mistake is treating `git log` as a record of what was saved. It shows what was committed. Unsaved work does not appear, and `reset`, `rebase` and `commit --amend` can rewrite the chain. A team needs an agreed rule for when history may be rewritten.
---

## Diff

A diff lists the lines that changed between two versions, like a copy editor's red strikethroughs and green insertions. Code review tools, including GitHub's pull request view, are built on the same output.

```bash
git diff                       # working tree versus staging area
git diff --staged              # staging area versus HEAD (the next commit)
git diff HEAD~1 HEAD           # the last commit versus the one before it
git diff main feature/shout    # two branches
```

An unstaged change looks like this:

```diff
--- a/notes.txt
+++ b/notes.txt
@@ -1,2 +1,3 @@
 first draft
 second draft
+third draft
```

Lines starting with `-` were removed, lines starting with `+` were added, and lines with a leading space are unchanged context. The `@@` header gives the line numbers. `git diff --stat` summarises the change per file. Binary files, such as images, show only that they changed.

The common mistake is running `git commit` after looking at `git diff` alone. The unstaged diff shows what you could commit, and the staged diff shows what you will commit. Check `git diff --staged` before every commit. Reviewers judge a pull request by its diff, so a diff that mixes a rename, a fix and a formatting change is hard to approve.

---

## Restore

`git restore` copies content from one place to another, and it touches files only. It replaced the file-level jobs of `git checkout` in Git 2.23, and it never moves a branch.

Without flags, it copies from the staging area into the working tree, which discards unstaged edits. With `--staged`, it copies from `HEAD` into the staging area, which unstages a file and keeps its edits. With `--source`, it copies from any commit.

```bash
echo "a bad idea" >> notes.txt
git restore notes.txt                    # discard the unstaged edit
git add notes.txt
git restore --staged notes.txt           # unstage, keep the edit in the working tree
git restore notes.txt                    # discard it for good
git restore --source HEAD~1 notes.txt    # take the older version from the last commit
```

`git restore` discards unstaged edits permanently, because Git has no copy of them. Treat it with the same care as `rm`, and run `git status` before and after so you can see which area changed.

Two related commands undo commits instead of files. `git reset` moves a branch pointer backward, and `git revert` adds a new commit that undoes an old one. Day 7 covers both, because the right choice depends on whether the commit has been shared.

The common mistake is running `git restore .` in a folder with hours of unsaved work. Check `git status` first, and copy the folder if the work matters.

---

## .gitignore

Some files should never be tracked: compiled output, dependency folders such as `node_modules/`, virtual environments such as `.venv/`, editor settings, operating system files, and anything holding a password or an API key. A `.gitignore` file tells Git which paths to skip.

It works like a "do not photograph" sign at a museum. The room is still there, and the sign stops certain things from being recorded. Ignored files stop appearing in `git status` and stop being added by `git add .`.

Each line is a pattern. A pattern without a slash matches at any depth, a trailing slash matches folders only, and a leading `!` re-includes something an earlier line excluded.

```gitignore
# Python
__pycache__/
*.pyc
.venv/

# Editors and operating systems
.vscode/
Thumbs.db
.DS_Store

# Secrets and generated output
.env
*.log
build/
```

Test a pattern with `git check-ignore -v build.log`, which prints the matching line. If it prints nothing, the file is not ignored.

The main limitation is that `.gitignore` only affects untracked files. If you committed `.env` yesterday, adding it to `.gitignore` today does nothing for the file already in history. Stop tracking it with `git rm --cached .env`, commit, and rotate the key, because the old value remains in the history. Another mistake is copying a template without reading it. A line that ignores `*.jar` in a template can hide a file your project needs, and the failure appears only on a teammate's machine. Run `git status` after a fresh clone to check.

---

## Hands-on exercises (about 60 minutes)

Work in `git-practice`. Run each command and read the output before going on.

Check the setup first (about 5 minutes):

```bash
git --version
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

On Windows you may see `LF will be replaced by CRLF the next time Git touches it`. This warning is about line endings and does not mean your content is wrong.

Exercise 1 builds a two-commit history (about 10 minutes).

1. Run `mkdir git-practice && cd git-practice && git init -b main`.
2. Create `notes.txt` with one line, stage it, and commit with the message "Add first note".
3. Add a second line, run `git diff`, read it, then stage and commit with the message "Expand first note".
4. Create `draft.md` and leave it unstaged.
5. Run `git log --oneline --graph --decorate`.

Done when `git log --oneline` shows exactly two commits and `git status --short` shows only `?? draft.md`.

Exercise 2 practices staging (about 10 minutes).

1. Add a third line to `notes.txt` and a line to `draft.md`.
2. Run `git add notes.txt`, then `git status --short`. Confirm `M ` for `notes.txt` and `??` for `draft.md`.
3. Run `git diff --staged`, then `git diff`. The first shows the third line; the second shows nothing for `notes.txt`.
4. Commit with the message "Add third note line".

Done when you can say in one sentence why `draft.md` was not in the commit.

Exercise 3 covers `.gitignore` (about 10 minutes).

1. Create `secret.env` containing `API_KEY=not-a-real-key` and `build.log` containing any text.
2. Run `git status --short`. Both appear as untracked.
3. Create `.gitignore` containing `*.env` and `*.log`. Run `git status --short` again.
4. Run `git check-ignore -v build.log` and save the output.
5. Commit `.gitignore` with the message "Ignore env and log files".

Done when `git ls-files` lists `.gitignore` and `notes.txt`, and neither ignored file.

Exercise 4 covers restore (about 15 minutes).

1. Append `BAD LINE` to `notes.txt` and save.
2. Run `git restore notes.txt`, then open the file to confirm the line is gone.
3. Append `BAD LINE` again, run `git add notes.txt`, then `git restore --staged notes.txt`. Run `git status` and note which column changed.
4. Discard the edit with `git restore notes.txt`.
5. Run `git log --oneline`, copy the hash of the first commit, and run `git restore --source <that-hash> notes.txt`. Read the file, then run `git restore notes.txt` to return to the latest version.

Done when you can explain in your own words the difference between `git restore` and `git restore --staged`.

Exercise 5 reads history (about 10 minutes).

1. Run `git log --stat` and find the commit with the largest change.
2. Run `git show --stat HEAD` and `git diff HEAD~1 HEAD --stat`. The file lists should match.
3. Run `git log --oneline -- notes.txt`.

Done when you can name the hash, message and file for your largest commit.

---

## Project: a Git-tracked greeting tool with a readable history

The code is deliberately tiny. The evidence you commit is the history: small commits, a clean ignore file and a written record of the undo drill.

Suggested repo name: `100-days-software-day-06-git-mental-model`.

The repo layout:

```
100-days-software-day-06-git-mental-model/
├── .gitignore
├── README.md
├── greet.py
├── notes/
│   └── mental-model.md
└── history/
    └── log.txt
```

Starter code for `greet.py`:

```python
import sys


def greet(name: str) -> str:
    return f"Hello, {name}!"


if __name__ == "__main__":
    print(greet(sys.argv[1] if len(sys.argv) > 1 else "world"))
```

Step-by-step tasks:

1. Create the folder, run `git init -b main`, add `greet.py`, and commit with the message "Add greeting function".
2. Add a `.gitignore` that excludes `__pycache__/`, `*.pyc`, `.venv/` and `.env`. Commit it with a message that says why.
3. Add a `--loud` option that prints the greeting in uppercase. Check it with `git diff --staged` before committing, and commit it on its own.
4. Create `notes/mental-model.md` with one short paragraph each on the working tree, staging, commits and history, in your own words. Commit it.
5. Make an unwanted change, undo it with `git restore`, and record the commands under a heading "Undo drill" in the notes file. Commit the notes.
6. Create the `history` folder, run `git log --oneline --graph --decorate > history/log.txt`, and commit it.
7. Write `README.md` with the purpose, the run command `python greet.py Ada`, and this checklist:
   - [ ] `python greet.py` prints `Hello, world!`
   - [ ] `python greet.py --loud Ada` prints `HELLO, ADA!`
   - [ ] Each commit message says what changed
   - [ ] No `__pycache__/` folder is tracked
8. Create an empty repository on GitHub with the same name, without a README, license or `.gitignore`. Run `git remote add origin <the HTTPS URL GitHub shows>` and `git push -u origin main`. Day 8 explains each piece.

Done when:

- `git log --oneline` shows at least five commits, each with a message that describes its change
- A fresh `git clone` of the GitHub repo runs `python greet.py Ada` without errors
- `history/log.txt` matches your local `git log --oneline --graph --decorate` output
- The "Undo drill" section names at least two `git restore` commands and what each did
- The GitHub repo shows your commits on `main`, and the README renders on the repo page

Evidence to commit: `greet.py`, `.gitignore`, `README.md`, `notes/mental-model.md`, `history/log.txt`, and the commits that produced them.

This project is light on code because the lesson is about the history. A reviewer who reads your log can see how you worked, and that is a Git skill you can point to in an interview.

---

## Check your understanding

1. You edit three files, stage one, and run `git commit -m "Fix bug"`. Which changes are in the commit, and what two commands would you run first to confirm it?
2. Two developers both see commit `5e1d437` in their logs. What does that tell you about their copies of the commit's content and history?
3. Why does `git diff` show different changes from `git diff --staged`? Describe a commit that goes wrong if you only check one of them.
4. Yesterday you committed an API key in `.env`, and today you added `.env` to `.gitignore`. Why is the key still exposed, and what should you do?
5. A teammate ran `git restore .` and lost work. What did the command discard, and what would you check to find out whether anything is recoverable?
6. Using the idea that history is a chain of parent commits, explain why commits titled "fix stuff" make an incident review slower.

### Answers

1. Only the staged file is committed. The other two stay in the working tree. Run `git status` to see the staged list, and `git diff --staged` to read the exact content before you commit.
2. The hash is computed from the commit's content, its parent, its author, its committer and its message. Two developers with the same hash have the same content and the same history behind it.
3. The unstaged diff compares the working tree with the staging area, so it shows what could be added. The staged diff compares the staging area with `HEAD`, so it shows what the next commit contains. If you check only the unstaged diff, you may commit changes that you never reviewed because they were not staged.
4. The key sits in an earlier commit, and `.gitignore` only affects untracked files. Anyone who clones the repo can read the old commit. Rotate or revoke the key, stop tracking the file with `git rm --cached`, and decide with the team whether the history must be rewritten.
5. `git restore .` discards unstaged edits to tracked files, and it can overwrite staged content depending on flags. Git has no copy of those edits, so check whether the work exists in an editor's undo history, a backup, or a copy of the folder.
6. An incident review walks the parent chain and reads each commit to find where a behavior changed. If each message says "fix stuff", the reviewer must open every diff to learn what it did, which takes much longer.

---

## Sources to read, watch and practice

*Read*

- Git documentation, Pro Git book, chapter "Getting Started" and the section on recording changes, at git-scm.com/book/en/v2
- Git documentation, man pages for `git-restore`, `git-diff`, `git-status` and `gitignore`, at git-scm.com/docs
- Atlassian Git tutorials at atlassian.com/git/tutorials. Search the site for "What is version control" and "Git tutorial".
- [Wikipedia: Version control](https://en.wikipedia.org/wiki/Version_control)
- [Wikipedia: Git](https://en.wikipedia.org/wiki/Git)
- [Wikipedia: Hash function](https://en.wikipedia.org/wiki/Hash_function)

*Tutorials*

- GeeksforGeeks, "Git Tutorial", at geeksforgeeks.org. Search the site for the title and read the sections on commits and staging.
- W3Schools, "Git Tutorial", at w3schools.com. Search for the Git section and follow the commands in order.

*Watch*

- freeCodeCamp.org, "Git and GitHub for Beginners - Crash Course" on YouTube. Run each command as the video shows it, up to the section on staging.

*Practice*

- Learn Git Branching at learngitbranching.js.org. Complete the first two levels in the browser. It draws commits and branches as you type.

---

## Tomorrow

Day 7 works with the history you read today. You will create branches, merge them, resolve a merge conflict, rebase for a cleaner history, and recover work after a mistake. Keep the `git-practice` folder, because Day 7 begins from it.
