# Day 7: Branches and Merging

*100 Days of Software Engineering, Module 2: Git and Collaboration*

**Time:** about 3 hours
**Level:** beginner to intermediate. You should have completed Day 6 and be comfortable with `git add`, `git commit`, `git status` and `git log`. This is for anyone who has been afraid of "merge conflict" messages.

**By the end of this day, you will have:**

- A working model of branches as movable pointers, and the ability to create, switch and delete them
- The ability to merge a branch, read a merge conflict, and resolve it on purpose
- A clear view of when `rebase` makes history cleaner, and when it is the wrong tool
- A set of recovery commands (`reflog`, `reset`, `revert`, `stash`) and the habit of using them before panicking
- A GitHub-ready repo whose history shows branches, one conflict and one recovery

**What you need:**

- Git 2.23 or newer (tested with Git 2.55.0, from `git --version`)
- The `git-practice` repo from Day 6, or a new folder you can use for this lesson
- A terminal (Ubuntu on WSL 2, Git Bash, or PowerShell) and an editor that opens from the terminal
- A free GitHub account for the project

**Prerequisites:**

- Day 6: Git Mental Model (working tree, staging, commits, hashes, `git log`, `git diff`, `git restore`, `.gitignore`)
- Day 4: Linux CLI I (`cd`, `ls`, `cat`)

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Branches](#branches)
- [Merge](#merge)
- [Conflicts](#conflicts)
- [Rebase concept](#rebase-concept)
- [Clean history](#clean-history)
- [Recovery](#recovery)
- [Hands-on exercises](#hands-on-exercises)
- [Project: a todo CLI with a branch conflict and a recovery drill](#project-a-todo-cli-with-a-branch-conflict-and-a-recovery-drill)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Two people fix two bugs in the same file on the same afternoon. Branches let each piece of work move independently, and merges bring it back together. Most of the friction new engineers feel comes from three misunderstandings: that a branch is a copy of the folder, that a conflict means something went wrong, and that a rebase rewrites history for no reason. This day corrects all three. The recovery section covers the undo commands people reach for under pressure, so you should know what each one does before you need it.

[Wikipedia: Merge (version control)](https://en.wikipedia.org/wiki/Merge_(version_control)) describes the merge operation in general terms. [Wikipedia: Git](https://en.wikipedia.org/wiki/Git) covers the tool that made these ideas common.

---

## Glossary

- **Branch:** a name that points at one commit. Making a commit on a branch moves the name forward.
- **Fast-forward:** a merge where the target branch is an ancestor of the source, so Git only moves the pointer.
- **Merge commit:** a commit with two parents, created when two lines of history join.
- **Conflict:** a merge or rebase that Git cannot complete automatically because both sides changed the same lines.
- **Rebase:** replaying your commits on top of a different base commit, which gives them new hashes.
- **Squash:** combining several commits into one.
- **Reflog:** a local log of where `HEAD` and each branch pointed, and when. It is the main recovery tool.
- **Stash:** a temporary shelf for uncommitted changes.
- **Force push:** pushing a branch when the remote does not contain your commits, which replaces the remote history. Day 8 covers remotes.

---

## Branches

A branch is a label on a commit. Creating one does not copy your files. Picture sticky notes placed on a timeline. Each note marks a commit, and when you add a new commit to the branch, the note moves forward to the new one. Many notes can point at the same commit, which is why branches are cheap in Git.

Creating and switching branches takes a moment because Git only writes a small file, `.git/refs/heads/<name>`, containing one hash.

```bash
git branch                       # list local branches, with * for the current one
git switch -c feature/todo-list  # create a branch and switch to it
git switch main                  # switch back
git log --oneline --graph --decorate --all
```

`git switch -c` is the modern form of `git checkout -b`. Older tutorials use `checkout`, and it still works, but `git switch` only switches branches, which makes it safer to type. Git 2.23 added `switch` alongside `restore`.

The main pitfall is working on the wrong branch. Commits go to whichever branch `HEAD` points at, so check `git branch` before your first commit. The second pitfall is deleting a branch that still holds unmerged commits. `git branch -d` refuses to delete such a branch, and `git branch -D` deletes it anyway, which is why the first command is the safe default.

Teams prefix branch names, such as `feature/`, `fix/` or `chore/`, so the list of branches shows what each one is for. Branches stay on your laptop until you push them, which Day 8 covers.

---

## Merge

A merge combines the commits of one branch into another. The simplest merge is a fast-forward, which happens when the target has no new commits. Git moves the pointer and creates no new commit.

```bash
git switch main
git merge feature/todo-list
```

If `main` has new commits since the branch was created, Git must combine two lines of history. It finds their common ancestor, computes the changes on each side relative to it, and produces a merge commit with two parents. The default output looks like this:

```
Merge made by the 'ort' strategy.
 todo.py | 4 ++--
 1 file changed, 2 insertions(+), 2 deletions(-)
```

The `--no-ff` flag forces a merge commit even when a fast-forward is possible. Teams use it to keep a visible record that a set of commits was a single feature:

```
*   2ea2126 Merge feature/b
|\
| * a992e82 Add b file
|/
* 2652cdf Feature A adds a line
```

The branch structure stays visible in `git log --graph`. Without `--no-ff`, the same commits appear in a straight line, and the feature boundary disappears. Either choice is valid, but a team should pick one.

`git merge --ff-only` is a safer option. It refuses to create a merge commit, so it fails when history has diverged. That failure tells you to rebase first, which is the subject of a later section.

The common mistake is merging the wrong direction. `git merge X` brings X into your current branch, so if you run it on `feature/x` by mistake, the feature branch receives `main`'s changes. Check `git branch` before every merge.

---

## Conflicts

A merge conflict is Git saying "I cannot decide this one for you". It happens when two branches changed the same lines, or when one branch changed a line that the other deleted. The conflict is not an error in the repository. Nothing is lost, and both versions are still in the commits.

Suppose `main` and `feature/shout` both changed the same line of `greet.py`:

```bash
git switch main
git merge feature/shout
```

Git stops and prints:

```
Auto-merging greet.py
CONFLICT (content): Merge conflict in greet.py
Automatic merge failed; fix conflicts and then commit the result.
```

The file now contains conflict markers:

```
<<<<<<< HEAD
    return f"Hello, {name}!"
=======
    return f"HELLO, {name.upper()}!"
>>>>>>> feature/shout
```

The part between `<<<<<<< HEAD` and `=======` is your current branch. The part between `=======` and `>>>>>>> feature/shout` is the incoming branch. You edit the file to the result you want, which may be one side, both sides combined, or something new. Then you remove the markers and stage the file:

```bash
git add greet.py
git status            # confirm nothing is still unmerged
git commit            # opens the editor with a default merge message
```

`git status` lists unmerged paths as `UU`. A file with conflicts is not ready until you `git add` it. Running `git commit` with the markers still in place commits them, which is a common way to ship broken code.

If you are unsure, `git merge --abort` returns to the state before the merge. Git only sees text positions. Two changes can merge cleanly and still be wrong, because the two sides disagreed about meaning without touching the same line. Run the tests after every merge, even a clean one.

---

## Rebase concept

Rebase takes a branch's commits and replays them on top of a different base. The result looks as if you started your work from the new base, with straight history and no merge commit. Picture a set of tracing paper sheets. Rebase lifts each sheet, moves the stack to a new starting point, and redraws the sheets one by one on the new point.

```bash
git switch feature/todo-list
git rebase main
```

Git finds the commits on `feature/todo-list` that are not on `main`, checks out `main`, and applies each commit in turn. Each replayed commit receives a new hash, because its parent changed. The content of the changes stays the same unless a conflict forces you to resolve it.

Before rebase, with letters standing in for hashes, the feature branch forked from an older commit:

```
* M (main) Add m
| * D2 (feature/d) Tweak d
| * D1 Add d1
|/
* X Merge feature/b into main
```

After `git rebase main`, the same two commits sit on top of `main`. This is the real output from a test run, with its hashes:

```
* ecfec35 (HEAD -> feature/d) Tweak d
* e494da2 Add d1
* 7b0989a (main) Add m
* 18a279a Merge feature/b into main
```

The rebased branch is now a straight line on top of `main`. A later fast-forward merge is possible, and the history reads in order.

Rebase rewrites commits, so it changes their hashes. Anyone who has based work on the old commits will have trouble when you push the rewritten ones. Rebase your own local branches freely. Do not rebase a branch that other people are using, or a shared `main`, unless your team has agreed to it.

Rebase can conflict once per replayed commit. A ten commit branch can produce ten rounds of conflicts, and each one needs `git rebase --continue` after you resolve it. If that becomes painful, `git rebase --abort` returns the branch to where it was. A merge of the same branch has one round of conflicts instead, which is sometimes the better trade.
---

## Clean history

A clean history is one a reader can follow. Each commit makes one logical change, each message explains that change, and the branch structure matches the real work. Clean history is not a goal in itself. It exists so that `git log`, `git blame` and `git bisect` are useful years later.

Three tools produce clean history. The first is `git commit --amend`, which replaces the last commit with a new one. Use it for a typo in a message or a file you forgot to stage, before you push.

```bash
git add forgotten.txt
git commit --amend --no-edit
```

The second tool is interactive rebase, which edits a range of commits. Open the list with `git rebase -i HEAD~3`. Git opens an editor with lines like this:

```
pick 7d1c3a0 Add parser
pick 3b9e2f1 Fix parser typo
pick 0a4c8e2 Add tests for parser
```

Change `pick` to `squash` (or `s`) on the second line to combine it into the first. Change it to `reword` (or `r`) to edit a message, and `drop` to remove a commit. Save and close the editor. Git replays the list and may ask you to write one combined message. The result is one clean commit called "Add parser". Interactive rebase is a local tool, and it rewrites hashes, so use it before you push.

The third tool is choosing the merge style. Fast-forward merges keep history linear. `--no-ff` merges keep the feature boundary visible. Pick one per team and write it down in the README or CONTRIBUTING file.

Clean history can erase useful information. If you squash a debugging session into one commit, the reasoning that led to a fix may disappear. A useful balance is to squash noisy commits such as "wip" and "oops", and keep commits that represent separate decisions.

Some teams ask for a rebase before merging so the history matches the final change. Others keep the true record of how the change was made. Both are reasonable once the team has agreed.

---

## Recovery

Recovery is about the commands that undo a mistake, and about knowing which one is safe. Git almost never deletes a commit immediately. A commit that no branch points at stays in the repository until garbage collection removes it, and by default the reflog keeps unreachable entries for 30 days. The reflog is the record that makes recovery practical.

```bash
git reflog
```

The reflog records each time `HEAD` moved, including checkouts, merges, rebases and resets, with a number such as `HEAD@{3}`. A recovery looks like this:

```bash
git reflog -n 5
git branch rescued-work HEAD@{2}     # create a branch at the old commit
git reset --hard HEAD@{2}            # or move the current branch there
```

The `reset` command comes in three forms. `--soft` moves the branch pointer and keeps the staging area and working tree, which is useful to redo a commit. `--mixed`, the default, moves the pointer and unstages changes but keeps the working tree. `--hard` moves the pointer and overwrites the working tree, which discards uncommitted changes. `--hard` is the most dangerous form, so run `git status` before it and be sure you can afford to lose the working changes.

`git revert` undoes a commit by adding a new commit that reverses its changes. It is the right choice on a branch that others have pulled, because it does not rewrite history.

```bash
git revert HEAD             # create a commit that undoes the last one
git revert --no-commit HEAD # stage the undo without committing
```

`git stash` saves uncommitted work on a shelf so you can switch branches. `git stash push -m "wip on login"` stores tracked changes, `git stash list` shows the shelf, and `git stash pop` applies the newest entry and removes it. Use `git stash apply` if you want to keep the entry.

If you deleted a branch by mistake, the reflog still contains its last commit. Find that commit with `git reflog`, then recreate the branch at it with `git branch <name> <hash>`.

The common mistake is using `git reset --hard` on a branch that has been pushed, or force pushing with `git push --force`, which overwrites the remote history for everyone. Use `git push --force-with-lease` when you must force push, because it refuses if the remote has commits you have not seen. Even then, coordinate with the team first. The safer habit is to create a backup branch before any destructive command, such as `git branch backup-before-reset`.

Recovery has a hard limit. The reflog is local, and it expires. A clone on another machine does not have your reflog. Work that was never committed or stashed cannot be recovered by Git at all.

---

## Hands-on exercises

Work in the `git-practice` repo from Day 6. If you do not have it, create a new one with `mkdir git-practice && cd git-practice && git init -b main`, add a `notes.txt` file, and commit it.

Exercise 1 creates a branch and fast-forwards it.

1. Run `git switch -c feature/intro` and add a line to `notes.txt`. Commit it with the message "Add intro line".
2. Run `git switch main`, then `git merge feature/intro`. Read the output and confirm it says `Fast-forward`.
3. Run `git log --oneline --graph --decorate`. There should be no merge commit.

Done when the output says `Fast-forward` and `main` points at the same commit as `feature/intro`.

Exercise 2 creates a merge commit with `--no-ff`.

1. Run `git switch -c feature/todo` and add a new file `todo.txt` with one task. Commit it.
2. Run `git switch main`, add a line to `notes.txt`, and commit it so that `main` has a new commit.
3. Run `git merge --no-ff feature/todo -m "Merge feature/todo"`.
4. Run `git log --oneline --graph --all`. Confirm the merge commit has two parents, and that the graph shows the branch.

Done when the graph shows a branch and a merge commit labeled "Merge feature/todo".

Exercise 3 produces and resolves a conflict.

1. Run `git switch -c feature/caps`, change the first line of `notes.txt` to uppercase, and commit.
2. Run `git switch main`, change the same first line to a different wording, and commit.
3. Run `git merge feature/caps`. Confirm that Git reports a conflict.
4. Open `notes.txt` and read the conflict markers. Edit the file to keep the text you want, and delete all the marker lines.
5. Run `git add notes.txt`, then `git commit` and save the default message.
6. Run `git log --oneline --graph -n 5` and identify the merge commit.

Done when `grep -c "<<<<" notes.txt` prints `0` and `git status` reports a clean working tree.

Exercise 4 uses rebase.

1. Run `git switch -c feature/rebase-me` and make two small commits, each with a different new file.
2. Run `git switch main` and make one commit on `main`.
3. Run `git switch feature/rebase-me` and then `git rebase main`.
4. Run `git log --oneline --graph --all -n 6`. The feature commits should sit on top of the `main` commit, with no merge commit.
5. Run `git switch main` and then `git merge --ff-only feature/rebase-me`.

Done when the `--ff-only` merge succeeds, which proves the history is linear.

Exercise 5 practices recovery.

1. Run `git switch -c feature/scratch`, create `scratch.txt`, and commit it with "Add scratch". Then run `git switch main` and `git branch -D feature/scratch`.
2. Run `git reflog -n 10` and find the "Add scratch" commit, which is the last commit the deleted branch pointed at.
3. Recreate the branch with `git branch feature/scratch <hash>` and confirm it appears in `git branch`.
4. Make an uncommitted change to `notes.txt`. Run `git stash push -m "exercise 5 wip"`, then `git stash list` and `git stash pop`.
5. Run `git revert HEAD` to undo the last commit, and read the new commit message Git suggests.

Done when you have recreated a deleted branch from the reflog, and can describe in one sentence the difference between `git reset --hard` and `git revert`.

---

## Project: a todo CLI with a branch conflict and a recovery drill

The project is a tiny Python todo tool. The code is small so the Git work is visible: two branches that conflict on purpose, a rebase, a recovery, and a commit history you can narrate.

Suggested repo name: `100-days-software-day-07-branches-and-merging`.

The repo layout:

```
100-days-software-day-07-branches-and-merging/
├── .gitignore
├── README.md
├── todo.py
├── docs/
│   └── branch-log.md
└── tests/
    ├── __init__.py
    └── test_todo.py
```

The empty `tests/__init__.py` file tells `python -m unittest` to look inside `tests/`. Without it, the command finds zero tests and still reports `OK`, which is a quiet failure worth knowing about.

Starter code for `todo.py`:

```python
import sys
from pathlib import Path

TODO_FILE = Path("todos.txt")


def add(task: str) -> None:
    with TODO_FILE.open("a", encoding="utf-8") as f:
        f.write(task + "\n")


def list_tasks():
    if not TODO_FILE.exists():
        return []
    return TODO_FILE.read_text(encoding="utf-8").splitlines()


if __name__ == "__main__":
    if len(sys.argv) > 2 and sys.argv[1] == "add":
        add(" ".join(sys.argv[2:]))
    else:
        for number, task in enumerate(list_tasks(), start=1):
            print(f"{number}. {task}")
```

Starter code for `tests/test_todo.py`:

```python
import unittest
from pathlib import Path

import todo


class TodoTests(unittest.TestCase):
    def setUp(self):
        self.original = todo.TODO_FILE
        todo.TODO_FILE = Path("test-todos.txt")

    def tearDown(self):
        if todo.TODO_FILE.exists():
            todo.TODO_FILE.unlink()
        todo.TODO_FILE = self.original

    def test_add_then_list(self):
        todo.add("write lesson")
        self.assertEqual(todo.list_tasks(), ["write lesson"])


if __name__ == "__main__":
    unittest.main()
```

Step-by-step tasks:

1. Initialize the repo with `git init -b main`, add the starter files, add `todos.txt` and `test-todos.txt` to `.gitignore`, and commit with a clear message.
2. Create `feature/done-marker` from `main`. Change the `print` line in `todo.py` so each task prints as `[ ]` before the number. Commit with the message "Show checkbox before each task".
3. Switch back to `main`. Create `feature/numbering` from there, and change the same `print` line so tasks start at 0 instead of 1. Commit with a message that says the behavior change.
4. Merge `feature/done-marker` into `main` with `--no-ff`. Merge `feature/numbering` into `main` and resolve the conflict on the `print` line by writing one correct version. Run `python -m unittest` before you commit the resolution.
5. Create `feature/tests` from `main`, add one test for `list_tasks` on an empty list, and commit. Then add a commit to `main` that changes the README. Rebase `feature/tests` onto `main` and fast-forward merge it.
6. Recovery drill: create `feature/scratch`, add a commit with a broken file, and delete the branch with `git branch -D feature/scratch`. Use `git reflog` to find the commit, recreate the branch, and confirm the file is back.
7. Write `docs/branch-log.md` with three short sections: the conflict you resolved and how, the rebase and why it was safe, and the recovery command you used and what it returned. Commit it.
8. Create an empty GitHub repository with the name above, then `git remote add origin <URL>` and `git push -u origin main`. Day 8 covers the details.

Done when:

- `git log --oneline --graph --all` shows two feature branches, one `--no-ff` merge, one conflict resolution and one rebased branch
- `python -m unittest` passes on `main`
- `docs/branch-log.md` names the commands for the conflict, the rebase and the recovery, with a one line result for each
- `git status` is clean and `todos.txt` is not tracked
- The GitHub repo shows the merged `main` with the full history

Evidence to commit: `todo.py`, `tests/test_todo.py`, `.gitignore`, `README.md`, `docs/branch-log.md`, and the commit history on GitHub.

This project is light on code and heavy on history, because the skill being tested is handling branches, not writing a todo tool. A reviewer who reads your graph should be able to see where two lines of work met, where you resolved them, and where you recovered from a mistake.

---

## Check your understanding

1. You create `feature/login`, make three commits, and switch back to `main`. Where does `main` point, and where do the three commits live?
2. A merge of `feature/x` into `main` produces a conflict in one function. What is in the file before you resolve it, and what must you do before the merge is complete?
3. Your feature branch is ten commits behind `main`, and you have not pushed it yet. Compare `git merge main` and `git rebase main` for this case, naming one risk of each.
4. A teammate says "rebasing is dangerous". Give the exact situation in which they are right, and the situation in which rebase is the safe choice.
5. You ran `git reset --hard HEAD~2` by mistake, losing two commits from your branch. Describe how you would find and recover them, and name the one piece of evidence that shows they still exist.
6. Why is `git revert` a better choice than `git reset --hard` for undoing a commit that has been pushed to a shared `main`?

### Answers

1. `main` still points at its own commit, and the three commits are on `feature/login`. Branches are labels on commits, so the commits exist in the repository whether or not a branch points at them directly.
2. The file contains conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) around both versions of the conflicting lines. Before the merge is complete, you must edit the file to the intended result, remove the markers, run `git add` on the file, and then commit.
3. `git merge main` creates a merge commit that joins the two histories, which keeps the record but adds a commit that is not part of the feature. `git rebase main` replays the ten feature commits on top of `main`, giving a straight history, but it changes their hashes and may produce conflicts once per commit. Because the branch is not pushed, rebasing is safe here.
4. Rebase is dangerous when others already have the old commits, because the rewritten commits have new hashes and their copies will diverge. It is safe on a local branch that nobody else has pulled, which is the common case for a feature branch you have not pushed.
5. Run `git reflog` to find the hash of the commit that was `HEAD` before the reset, then create a branch at it with `git branch recovered <hash>`, and cherry-pick or merge it back. The evidence is the reflog entry that shows the old `HEAD`, because the commits were not deleted from the object store.
6. `git revert` adds a new commit that undoes the changes, so existing history stays intact and everyone else's clones still fit. `git reset --hard` moves the branch pointer and makes the other developers' histories diverge from the remote the next time they push or pull.

---

## Sources to read, watch and practice

*Read*

- Git documentation, Pro Git book, chapter "Git Branching" and the section on rebasing, at git-scm.com/book/en/v2
- Git documentation, man pages for `git-merge`, `git-rebase`, `git-reflog` and `git-reset`, at git-scm.com/docs
- Atlassian Git tutorials at atlassian.com/git/tutorials. Search the site for "Merging vs. Rebasing" and "Undoing changes".
- [Wikipedia: Merge (version control)](https://en.wikipedia.org/wiki/Merge_(version_control))
- [Wikipedia: Version control](https://en.wikipedia.org/wiki/Version_control)

*Tutorials*

- GeeksforGeeks, "Git Branch" and "Git Merge" tutorials, at geeksforgeeks.org. Search the site for each title.
- W3Schools, "Git Branch" tutorial, at w3schools.com. Search for the Git section and follow the commands.

*Watch*

- freeCodeCamp.org, "Git and GitHub for Beginners - Crash Course" on YouTube. Watch the sections on branches and merging, and repeat each command in your repo.

*Practice*

- Learn Git Branching at learngitbranching.js.org. Complete the levels on branching, merging, and rebasing. The visual graph shows each rebase as it happens.

---

## Tomorrow

Day 8 moves your repo onto GitHub. You will connect a local repo to a remote, clone a repo, push and pull, open issues and pull requests, review code, and publish a release with a version tag. The branches and merges from today become the pull requests you will open tomorrow.

**Next:** [Day 8: GitHub Workflow](Software-Day-08-GitHub-Workflow.md)
