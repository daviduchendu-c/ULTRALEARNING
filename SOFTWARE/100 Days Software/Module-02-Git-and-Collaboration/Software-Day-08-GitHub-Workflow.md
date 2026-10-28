# Day 8: GitHub Workflow

*100 Days of Software Engineering, Module 2: Git and Collaboration*

**Time:** about 3 to 4 hours
**Level:** beginner to intermediate. You should have completed Days 6 and 7 and have a local repo with commits and branches. Some of the work is in a web browser, so you need a GitHub account.

**By the end of this day, you will have:**

- A local repo connected to a remote, with `clone`, `push`, `fetch` and `pull` working in both directions
- An issue, a pull request that closes the issue, and a review on that pull request
- A README that tells a stranger what the project does and how to run it
- A tagged release (`v0.1.0`) with release notes, published on GitHub

**What you need:**

- Git 2.23 or newer (tested with Git 2.55.0, from `git --version`)
- A GitHub account (free). Public repositories are free, and the free plan includes unlimited public repos.
- Authentication for Git over HTTPS. GitHub no longer accepts your account password for Git operations. Git for Windows includes Git Credential Manager, which asks for a sign-in the first time you push. Ubuntu users can use the same approach through `git-credential-manager` or an SSH key.
- Optional: the GitHub CLI (`gh`) from cli.github.com. The lesson uses the website and plain Git, so `gh` is not required.
- The todo CLI repo from Day 7, or a new folder for this lesson

**Prerequisites:**

- Day 6: Git Mental Model (commits, hashes, `git log`, `git diff`, `.gitignore`)
- Day 7: Branches and Merging (branches, merge, rebase, `git reflog`)

---

## In this guide you are going to cover the following

- [Introduction](#introduction)
- [Glossary](#glossary)
- [Remotes](#remotes)
- [Clone](#clone)
- [Push and pull](#push-and-pull)
- [Issues](#issues)
- [Pull requests](#pull-requests)
- [Code review](#code-review)
- [README](#readme)
- [Releases](#releases)
- [Hands-on exercises (about 60 minutes)](#hands-on-exercises-about-60-minutes)
- [Project: publish the todo CLI with issues, a reviewed pull request and a release](#project-publish-the-todo-cli-with-issues-a-reviewed-pull-request-and-a-release)
- [Check your understanding](#check-your-understanding)
- [Sources to read, watch and practice](#sources-to-read-watch-and-practice)
- [Tomorrow](#tomorrow)

---

## Introduction

Git stores history on your machine. GitHub is where that history is shared, discussed and released. A team uses it to decide what to work on (issues), propose a change (pull requests), check each other's work (code review), explain the project to a stranger (the README), and publish a version that people can install (releases). Many software teams run their daily work through these features, and a public GitHub profile with a well-kept project is a proof of skill that an employer can check.

Today you learn the mechanics that connect a local repo to the shared one, and the workflow that teams build on top of them. The Git commands are the same as on Day 6 and Day 7. The new skill is knowing which change belongs in which place: a bug report in an issue, the code change in a branch, the explanation in the README, and the version in a tag.

[GitHub Docs](https://docs.github.com) is the official reference for every feature in this lesson. [Wikipedia: Code review](https://en.wikipedia.org/wiki/Code_review) explains why teams review changes at all.

---

## Glossary

- **Remote:** a named link to another copy of a repository, usually on a server. The default name is `origin`.
- **Clone:** a full copy of a repository, including its history, made from a remote.
- **Fetch:** download new commits and branches from a remote without changing your branches.
- **Pull:** fetch, then merge or rebase the remote branch into your current branch.
- **Push:** send your local commits to a remote branch.
- **Upstream:** the remote branch that a local branch tracks, set with `git push -u`.
- **Issue:** a tracked task, bug report or question, stored in the repository's issue list.
- **Pull request (PR):** a request to merge one branch into another, with discussion and review attached.
- **Review:** a reader's comments and decision (approve, comment, or request changes) on a pull request.
- **Tag:** a fixed name for one commit, often used to mark a version.
- **Release:** a GitHub page for a tag that holds release notes and optional downloadable files.
- **Semantic versioning:** a version scheme of `MAJOR.MINOR.PATCH`, where a major change can break users. [Wikipedia: Software versioning](https://en.wikipedia.org/wiki/Software_versioning) describes the general practice.

---

## Remotes

A remote is a bookmark that points at another repository. Your local repo can have several remotes, but the convention is one called `origin` that points at the copy everyone shares. Git stores the names and URLs in `.git/config`.

```bash
git remote -v
git remote add origin https://github.com/<your-user>/100-days-software-day-08-github-workflow.git
git remote -v
```

The first `git remote -v` prints nothing for a fresh repo. After `git remote add`, it prints a `fetch` and a `push` line for each remote. The two URLs are usually the same. They differ only when a project's contributors have different permissions for reading and writing.

The URL can be HTTPS, which works with a token-based sign-in, or SSH, which uses a key pair. HTTPS is simpler at first. Both forms appear on the repository's green "Code" button on GitHub, and you can switch between them with `git remote set-url origin <new-url>`.

The common mistake is pointing `origin` at the wrong repository, such as a fork you did not intend to push to. Check `git remote -v` before the first push. A second mistake is assuming a remote is a backup. It is a shared copy, and a force push there can erase other people's commits. Day 7 explains why.

---

## Clone

Cloning creates a new local folder that holds the full repository, including every branch and every commit, and it sets `origin` for you. It is the first thing you do when you join a project.

```bash
git clone https://github.com/<your-user>/100-days-software-day-08-github-workflow.git
cd 100-days-software-day-08-github-workflow
git status -sb
```

`git status -sb` shows the current branch and its tracking state. A fresh clone prints `## main...origin/main`, which means the local `main` tracks `origin/main` and has nothing new to push or pull.

Clone copies the history, so an old, long history can take time to download. A shallow clone with `git clone --depth 1 <url>` downloads only the latest commit. It suits CI runners and quick reads, but `git log` shows only the commits that were downloaded, so you do not get the full history.

The common mistake is cloning into a folder that already has the project, which creates a second copy with a different remote state. Clone into a new folder, and keep your local work in one place. A second mistake is editing files in the `.git` folder. Only Git should touch it.

---

## Push and pull

Push sends your commits to the remote. Pull brings the remote's commits into your branch. Both are about synchronising two copies of the same history, and both depend on the shared branch names being the same on both sides.

Push a new local branch and set its upstream the first time:

```bash
git switch -c feature/add-done-marker
git commit --allow-empty -m "Start done marker work"
git push -u origin feature/add-done-marker
```

The `-u` flag (short for `--set-upstream`) records that the local branch tracks `origin/feature/add-done-marker`. After that, `git push` and `git pull` work without arguments. Without upstream tracking, Git stops and prints the exact command to run, which is a useful hint.

Fetch and pull are separate on purpose. `git fetch` downloads remote commits and updates the remote-tracking branches such as `origin/main`, but it does not change your branches. `git pull` does a fetch and then integrates the result. Many teams run `git pull --rebase` to keep history straight, as in Day 7, and set it as the default with `git config --global pull.rebase true`.

Here is a real case, tested with a second clone of the same remote. Your local `main` and the remote `main` each have one new commit:

```
## main...origin/main [ahead 1, behind 1]
```

Running `git pull --rebase` replays your commit on top of the remote's, and then `git push` sends the result. Had you run a plain `git pull` without the rebase, Git would have created a merge commit, which is also valid. The `[ahead 1, behind 1]` line is the signal that both sides changed.

The common mistake is a rejected push: `! [rejected] main -> main (fetch first)`. It means the remote has commits you do not have. Run `git fetch`, inspect with `git log --oneline main..origin/main`, integrate with `git pull --rebase`, and push again. Never answer a rejection with `git push --force` on a shared branch. Force push replaces the remote history and removes your teammate's commits from the shared copy.

Tags are pushed separately, with `git push origin <tag>` or `git push --tags`. Plain `git push` does not send them.

---

## Issues

An issue is a shared description of a piece of work. It may be a bug, a feature request, a question or a task. An issue is not code, and it does not have to be finished in one branch. Its job is to answer "what problem are we solving, and how do we know it is done?"

A good issue has a title that names the behavior, a description with steps to reproduce or a clear goal, and a definition of done as a short checklist. Labels such as `bug` or `enhancement` let a team filter the list. An assignee shows who is responsible.

```markdown
Title: `todo add` ignores tasks that contain quotes

Steps:
1. Run `python todo.py add "buy "milk""`
2. Run `python todo.py`

Expected: the task appears with its text intact.
Actual: the task is truncated at the second quote.

Done when: a unit test covers quotes and the test passes on main.
```

Connect code to issues in the pull request. Writing `Closes #3` (or `Fixes #3`) in the pull request description links the PR to issue number 3. When the PR merges into the default branch, GitHub closes the issue automatically. This keeps the issue list accurate without anyone clicking a button.

A poorly written issue wastes everyone's time. An issue that says "make it better" gets no useful response. Write the steps and the expected result, and link to the code that is affected.

---

## Pull requests

A pull request asks the project to merge one branch into another. It is a conversation attached to a set of commits. The PR shows the diff, the commit list, the discussion, the status of automated checks, and the reviews.

To open one, push a branch, then open the repository on GitHub. The page usually offers a "Compare & pull request" button for a branch you just pushed. Set the base branch to `main`, write a title and description, and create the PR. Some teams open a draft PR while the work is unfinished, which tells reviewers that the code is not ready for a full review.

A good PR description answers three questions: what changed, why, and how to check it. Include the issue link, the commands to run, and anything you want reviewers to look at closely.

```markdown
## What
Adds a checkbox before each task in `todo.py`.

## Why
Closes #1. Users asked to see completion state.

## How to check
- `python todo.py add "write lesson"`
- `python todo.py` shows `[ ] 1. write lesson`
- `python -m unittest` passes
```

Keep the PR small. Reviewers spend less time and find more problems when a change fits in one sitting, usually a few hundred lines or less. If a feature is large, split it into a series of PRs that each leave the project working.

A pull request is only as good as its tests and checks. A green check mark means the automated checks passed, not that the change is correct. Read the diff yourself before you ask someone else to.

---

## Code review

Code review is a second person reading a change before it reaches the main branch. It catches bugs, spreads knowledge about the codebase, and keeps style consistent. Reviews also teach people the codebase, which a document rarely does as well.

On GitHub, a reviewer opens the PR's "Files changed" tab, clicks the line number they want to discuss, and writes a comment. A reviewer can also propose an exact change with a "suggested change" block, which the author can accept with one click. When they finish, they choose one of three outcomes: comment only, approve, or request changes. An approval from the right reviewer is usually required before merging, and a team can enforce that with branch protection rules.

A suggested change is written as a fenced block labelled `suggestion` inside the review comment, like this:

````markdown
```suggestion
    print(f"[ ] {number}. {task}")
```
````

Review is a two-way skill. As an author, respond to each comment with a change, a reason, or a question. Mark resolved threads only after you have addressed them. As a reviewer, comment on what matters: correctness, edge cases, naming that will confuse the next reader, and missing tests. Keep style nits to a minimum, and automate them with a linter when you can.

People skim large diffs. A reviewer who reads 800 lines in one sitting misses things. A second limitation is that you cannot approve your own pull request on GitHub, which means solo projects need another habit, such as reviewing your own PR in the browser the next morning with fresh eyes, or asking a peer to review.

---

## README

The README is the first file most visitors open. It tells them what the project does, why it exists, how to install and run it, and how to contribute. GitHub renders `README.md` on the repository's front page. The file uses Markdown, a lightweight text format that GitHub renders into headings, lists and code blocks.

A useful README answers five questions in order. What is this? Who is it for? How do I run it in under five minutes? What does a typical command look like? How do I contribute or report a problem? A short section on the license and on the project's status completes the set.

```markdown
# todo

A command-line todo list written in Python. It stores tasks in a plain text file.

## Run it

    python todo.py add "buy milk"
    python todo.py

## Test it

    python -m unittest

## Status

Version 0.1.0. Tasks cannot yet be marked done (see issue #1).
```

Test the README by following it. Clone the repo into a new folder and run every command in the README exactly as written. If a step is missing, a stranger will get stuck at that step, and the README is the reason the project looks unfinished.

A README goes stale quickly. A command that changed in version 2 but not in the README sends readers to an error. Update the README in the same pull request as the code change, and make it part of your definition of done.
---

## Releases

A release is a named version of the project that people can rely on. In Git, the simplest form is a tag, a fixed name for one commit. On GitHub, a release is a page built on a tag. It holds notes that describe what changed and, optionally, files people can download.

Use semantic versioning when you choose the number. Increase the `MAJOR` number for changes that break users, the `MINOR` number for new features that keep old behavior working, and the `PATCH` number for bug fixes. The first public version is often `0.1.0`, which signals that the project is early and may change.

Create an annotated tag with a message, then push it:

```bash
git tag -a v0.1.0 -m "First release: add, list and tests"
git push origin v0.1.0
git tag --list
git show v0.1.0 --stat
```

An annotated tag stores the tagger, the date and the message, which makes it suitable for releases. A lightweight tag (`git tag v0.1.0` without `-a`) stores only a name. Use annotated tags for releases, and push them explicitly, because plain `git push` does not send tags.

On GitHub, open the repository's Releases section and choose the option to draft a new release. Pick the existing tag `v0.1.0`, give the release a title, and write notes. Keep the notes short and specific: a list of user-visible changes with issue or PR numbers. Publish the release when the notes are right. A release can be edited later, but tags should not be moved, because people and build systems depend on them.

Versions only work when they are honest. A `PATCH` release that breaks behavior breaks trust, and a release with no notes forces every reader to read the log. Keep a `CHANGELOG.md` that records each version, and link the release notes to it.
---

## Hands-on exercises (about 60 minutes)

Exercise 1 practices remotes, clone, push and pull without an account, using a local bare repository as the remote (about 15 minutes). A bare repository holds history and has no working files, which is how servers store repos.

1. From the folder that contains `git-practice`, create a bare repo: `git init --bare remote-practice.git`.
2. Inside `git-practice`, run `git remote add origin ../remote-practice.git`, then `git remote add origin ../remote-practice.git`, then `git push -u origin main`.
3. Clone it into a second folder: `git clone ../remote-practice.git ../second-copy`. Run `cd ../second-copy` and `git log --oneline -n 3`.
4. In the second copy, make a commit and push it. Go back to `git-practice` and run `git fetch`, then `git status -sb`. It should say `behind 1`.
5. Run `git pull --ff-only` in `git-practice`. Confirm the new commit is in `git log`.

Done when `git status -sb` in `git-practice` shows `## main...origin/main` with no ahead or behind count.

Exercise 2 creates a GitHub repo and pushes to it (about 15 minutes).

1. On GitHub, create a new public repository named `100-days-software-day-08-github-workflow`. Do not add a README, license or `.gitignore`, since your local repo already has content.
2. Copy the HTTPS URL from the Code button, then run `git remote add origin <url>` in your Day 7 todo repo, or in a new repo you create for this lesson.
3. Run `git push -u origin main`. Authenticate when Git Credential Manager asks.
4. Refresh the repo page and confirm your commits are listed.

Done when the repo page shows your commit history, and `git status -sb` shows `main...origin/main` with no differences.

Exercise 3 creates an issue, a branch and a pull request (about 15 minutes).

1. Open an issue titled "Show a checkbox before each task", with a short description and a definition of done.
2. Create a branch with `git switch -c feature/checkbox-marker`, change the `print` line in `todo.py` to show `[ ]` before each task, and commit with the message "Show checkbox before each task".
3. Push with `git push -u origin feature/checkbox-marker`.
4. Open a pull request into `main`. In the description, write `Closes #1` (use the number of your issue).
5. Read the diff on GitHub's Files changed tab. Leave one comment on your own PR asking yourself a question about an edge case, such as what happens with an empty list.

Done when the PR shows the closing reference and the diff, and the issue links to the PR.

Exercise 4 covers review and merging (about 10 minutes).

1. Reply to your own comment with the answer, and change the code if the answer requires it. Push the change to the same branch.
2. Merge the PR on GitHub with the default merge button, then confirm the issue closes automatically.
3. Locally, run `git switch main`, then `git pull`, then `git log --oneline -n 4`.

Done when the issue shows as closed, and your local `main` contains the merged commit.

Exercise 5 creates a release (about 10 minutes).

1. Run `git tag -a v0.1.0 -m "First release"` on `main`, and push it with `git push origin v0.1.0`.
2. On GitHub, open Releases, draft a new release from tag `v0.1.0`, and write two bullet points of notes.
3. Publish the release, then open the release page and check the tag name, date and notes.

Done when the release page shows `v0.1.0` with your notes.

---

## Project: publish the todo CLI with issues, a reviewed pull request and a release

The project takes the todo CLI from Day 7 and publishes it the way a team would. You will open issues, use a pull request to fix one, write the README, add a continuous integration check, and publish version `0.1.0`. The code is the same small program. The evidence is the workflow.

Suggested repo name: `100-days-software-day-08-github-workflow`.

The repo layout:

```
100-days-software-day-08-github-workflow/
├── .github/
│   └── workflows/
│       └── ci.yml
├── .gitignore
├── CHANGELOG.md
├── README.md
├── todo.py
└── tests/
    ├── __init__.py
    └── test_todo.py
```

Starter code for `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python -m unittest
```

GitHub Actions runs this file on every push and pull request. It is free for public repositories. The version numbers after `@` are the major versions that were current when this lesson was written. Check [GitHub Docs](https://docs.github.com) for the current recommended versions before you copy them.

Step-by-step tasks:

1. Copy `todo.py`, `tests/__init__.py` and `tests/test_todo.py` from Day 7 into the new folder, add the `.gitignore` from Day 7 (including `__pycache__/` and `todos.txt`), and create an empty `CHANGELOG.md`. Commit with a clear message.
2. Create the GitHub repo, push `main`, and confirm the commit list on GitHub.
3. Add `ci.yml` in a branch named `ci/add-workflow`. Commit, push, and open a pull request. Wait for the CI check and confirm it passes on the Actions tab. Merge the PR.
4. Open issue 1: "Mark tasks as done". Open issue 2: "Document the commands in the README". These are the two issues you will fix.
5. Create `feature/readme`, write `README.md` with the five sections from this lesson (what it is, run it, test it, status, and license), and reference issue 2 in the PR with `Closes #2`. Open the PR, check the Files changed tab, and merge after your own review comment is answered.
6. Create `feature/mark-done`. Add a `done` command to `todo.py` that marks a task by number, or a simpler version that prints a message if the number is invalid. Add one test. Open a PR with `Closes #1`. Use the review checklist below, and merge after the check passes.
7. Update `CHANGELOG.md` with a `0.1.0` section listing the features. Commit it on `main` through a PR.
8. Tag `v0.1.0` on `main`, push the tag, and publish a GitHub Release from it with notes linked to the changelog.

README checklist:

- [ ] The README says what the project does in one sentence
- [ ] `python todo.py add "buy milk"` and `python todo.py` are shown and work
- [ ] `python -m unittest` is listed under "Test it" and passes
- [ ] The status section names the current version
- [ ] A fresh clone can follow every command without help

PR review checklist (use it on your own PRs, since you cannot approve your own work):

- [ ] The description links to the issue with `Closes #N`
- [ ] The diff contains only the change the issue describes
- [ ] The new test fails before the change and passes after it
- [ ] The CI check is green

Done when:

- The repo shows at least two issues, one closed by a merged PR and one closed by another merged PR
- The CI badge or the Actions tab shows a passing run on `main`
- `README.md` passes the fresh-clone check
- The release `v0.1.0` exists on GitHub, with notes that match `CHANGELOG.md`
- `git tag --list` on your machine shows `v0.1.0`

Evidence to commit and publish: the repo itself on GitHub, including `.github/workflows/ci.yml`, `README.md`, `CHANGELOG.md`, the closed issues, the merged PRs, and the `v0.1.0` release page. Link the repo and the release page in your portfolio.

The code takes an hour to write. The workflow is the part a reviewer looks at: a clear issue, a focused change, a documented result, and a version someone can install.

---

## Check your understanding

1. You ran `git push` and got `rejected (fetch first)`. Explain what the rejection means and the safe sequence of commands to fix it, without using a force push.
2. What is the difference between `git fetch` and `git pull`, and when would you run `git fetch` on its own?
3. A pull request has a green CI check and no comments. Why is that not enough evidence that the change is correct, and what would you check before approving it?
4. You need a fix on `main` to reach users right now. Why is an annotated tag and a release more useful than telling users to "use the latest commit"?
5. A teammate asks you to write a README for a project with no users yet. Which sections would you write first, and what would make you update the README in the same PR as a code change?
6. Why should the code change and the issue stay linked, and what does `Closes #12` in a PR description do when the PR merges?

### Answers

1. The remote has commits that your local branch does not. Run `git fetch` to download them, inspect them with `git log --oneline main..origin/main`, integrate them with `git pull --rebase` (or a merge), and then run `git push`. A force push would replace the remote's commits and remove other people's work from the shared branch.
2. Fetch downloads remote commits and updates the remote-tracking branches such as `origin/main`, but it does not change your local branches. Pull does a fetch and then integrates the result. Run `git fetch` on its own when you want to see what changed before you decide how to combine it.
3. A green check shows that the automated tests passed, not that the change matches the issue or handles edge cases. Before approving, read the diff, check that it contains only the described change, confirm the new test fails without the change, and run the code on the examples in the README.
4. An annotated tag gives a fixed, named point in history with a message and a date. Users and build systems can refer to `v0.1.0` and get the same code every time. "The latest commit" changes with each push, so a user who installed it last week cannot tell what version they have.
5. Start with what the project does, how to run it, and how to test it, because those three answer the first questions a stranger asks. Update the README in the same PR whenever a command, a flag or a required setup step changes, because a README that is out of date sends new users to errors.
6. The link lets anyone trace from the bug or feature to the code and the review, and it keeps the issue list accurate. When a PR with `Closes #12` merges into the default branch, GitHub closes issue 12 automatically.

---

## Sources to read, watch and practice

*Read*

- GitHub Docs (docs.github.com). Search for "About remote repositories", "About pull requests", "About issues", "About releases", and "Managing branch protection rules".
- Git documentation, Pro Git book, chapter "GitHub", at git-scm.com/book/en/v2. The chapter covers contributing to projects on GitHub.
- Git documentation, man pages for `git-remote`, `git-push`, `git-fetch` and `git-pull`, at git-scm.com/docs
- [Wikipedia: Code review](https://en.wikipedia.org/wiki/Code_review)
- [Wikipedia: Software versioning](https://en.wikipedia.org/wiki/Software_versioning)
- Semantic Versioning specification, at semver.org

*Tutorials*

- GeeksforGeeks, "Git Remote" and "Git Clone" tutorials, at geeksforgeeks.org. Search the site for each title.
- W3Schools, "Git Push" and "Git Pull" tutorials, at w3schools.com, in the Git section.

*Watch*

- GitHub's official YouTube channel. Search the channel for "pull request" and "GitHub Actions" for short walkthroughs from the vendor.
- freeCodeCamp.org, "Git and GitHub for Beginners - Crash Course" on YouTube. Watch the sections on remotes and pull requests, and run each step in your repo.

*Practice*

- GitHub Skills at skills.github.com. Complete a course on pull requests or on the basics of GitHub, which runs in your own account.

---

## Tomorrow

Day 9 starts Module 3, Python (Deep Foundation), with the Python runtime and syntax. The Git and GitHub habits from this module carry over directly, because every later day builds on a repo with a README, issues, pull requests and tags. Keep the todo CLI repo, because later days return to it.
