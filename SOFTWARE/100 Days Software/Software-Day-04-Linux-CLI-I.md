# Day 4: Linux CLI I

*100 Days of Software Engineering, Module: Foundations*

**Time:** about 3 to 4 hours
**You will build:** an organized project folder created entirely from the command line, plus a field guide to the Linux filesystem based on your own exploration, in a GitHub repo.
**You should know:** Day 3 (Windows Terminal, WSL 2 with Ubuntu, PATH, absolute and relative paths). Everything today happens in your **Ubuntu** tab, not PowerShell.

---

## 1. The big idea

Most computers you will deploy code to have no screen, no mouse and no File Explorer. They are Linux servers, containers and build machines that you reach through a terminal. On those machines, a handful of commands is how you look around, find a config file, read a log, copy a release into place and clean up after yourself. An engineer who can move calmly through a Linux filesystem during an outage at 2 a.m. is worth a lot.

Today you learn to move, create, copy, move, delete and read files. You learn how to get help on any command without a search engine, and you learn the layout of the Linux tree so you always know where to look.

---

## 2. The filesystem hierarchy

**Intuition.** A hospital puts the same departments on the same floors in every branch, so a doctor who transfers from another city can find the pharmacy without asking. The Filesystem Hierarchy Standard (FHS) does this for Linux. Config files go in one place, logs in another and programs in a third.

**Definition.** Linux has one tree that starts at `/`. Everything appears somewhere inside it, including other drives and hardware devices.

```
/
├── bin  -> usr/bin    essential commands (a symlink on modern Ubuntu)
├── boot               kernel and boot loader files (mostly unused in WSL)
├── dev                device files: /dev/null, disks, terminals
├── etc                system-wide configuration, as text files
├── home
│   └── ada            YOUR files live here (~)
├── lib  -> usr/lib    shared libraries
├── mnt
│   └── c              your Windows C: drive, in WSL
├── opt                optional, self-contained third-party software
├── proc               live information about processes and the kernel
├── root               the root user's home directory
├── sbin -> usr/sbin   system administration commands
├── tmp                temporary files, may be wiped on reboot
├── usr
│   ├── bin            most user commands (ls, cp, python3)
│   ├── lib            libraries for those commands
│   ├── local          software you install by hand
│   └── share          shared data, including man pages
└── var
    ├── cache          downloadable data that can be rebuilt
    ├── lib            program state, such as package databases
    └── log            log files
```

**Worked example.** Explore four of these directories:

```bash
ls /                      # list the top level
cat /etc/os-release       # which Ubuntu version is this?
ls /proc                  # numbered folders: one per running process (PIDs, Day 2)
cat /proc/meminfo         # memory facts, generated live by the kernel
```

The numbers in `/proc` are PIDs. `/proc` does not exist on disk. The kernel generates its contents each time you read them. That is the Unix idea that "everything is a file."

**Mistakes.** Saving work in `/tmp` and losing it after a restart. Editing files in `/etc` without a backup (copy the file first; `cp` is below). Assuming every distro is identical. Ubuntu merges `/bin` into `/usr/bin` with a symlink, but other systems keep them separate. The FHS is a convention that distros follow loosely.

When a production web server misbehaves, the first places an on-call engineer checks are its config under `/etc` (for example `/etc/nginx/`) and its logs under `/var/log`.

---

## 3. Where am I, and what is here?

### 3.1 `pwd`

**Intuition.** The "You are here" dot on a mall map.

**Definition.** `pwd` (print working directory) prints the absolute path of the shell's current directory. Relative paths are resolved from there. Your prompt usually shows it too, with `~` standing for your home.

```
ada@laptop:~$ pwd
/home/ada
ada@laptop:~$ cd /var/log
ada@laptop:/var/log$ pwd
/var/log
```

If you reach a folder through a symlink, `pwd -P` prints the real location. `cd /bin` followed by `pwd -P` shows `/usr/bin` on Ubuntu.

**Mistakes.** Running a command with a relative path from the wrong directory, such as `rm notes.txt` when you thought you were in `~/old`. Make `pwd` a reflex before anything destructive. Scripts fail the same way, because they run from wherever you launched them, not from where the script file lives. This is why CI configs set the working directory explicitly.

### 3.2 `ls`

**Intuition.** Opening a drawer and looking inside. The flags decide whether you glance or empty it onto the table.

**Definition.** `ls` lists the contents of a directory. With no argument, it lists the working directory.

| Flag | Meaning |
|---|---|
| `-l` | long format: permissions, owner, size, date |
| `-a` | show all, including hidden dotfiles, `.` and `..` |
| `-h` | human-readable sizes (K, M, G), with `-l` |
| `-t` | sort by modification time, newest first |
| `-r` | reverse the sort |
| `-R` | recurse into subdirectories |
| `-d` | list a directory itself, not its contents |

**Worked example.**

```
ada@laptop:~$ ls -lah
total 32K
drwxr-x--- 4 ada  ada  4.0K Oct 24 09:12 .
drwxr-xr-x 3 root root 4.0K Oct 23 18:40 ..
-rw-r--r-- 1 ada  ada  3.7K Oct 23 18:40 .bashrc
drwxr-xr-x 3 ada  ada  4.0K Oct 24 09:10 dev
-rw-r--r-- 1 ada  ada   220 Oct 24 09:12 notes.txt
```

Read one line from left to right. The first character is the type: `-` for a regular file, `d` for a directory, `l` for a symlink. Then come the permissions (`rw-r--r--`, covered on Day 5), the link count, the owner and group (`ada ada`), the size in bytes (`220`), the last-modified time, and the name. Run `ls -l /` and find the `bin -> usr/bin` symlink from section 2.

**Mistakes.** Deciding a file is missing because it is hidden. Add `-a`. A huge directory floods the screen. Pipe the output to a pager (Day 5) or narrow it with a glob such as `ls *.log`. Do not write scripts that parse `ls` output to get filenames. Names with spaces or newlines break them. Use `find` (Day 5) instead. On the job, `ls -lt /var/log` (newest logs first) is a standard first step in an incident.

### 3.3 `cd`

**Intuition.** Walking from room to room. You can give directions from where you stand, or give a full street address.

**Definition.** `cd` (change directory) changes the shell's working directory.

| Command | Goes to |
|---|---|
| `cd /etc` | an absolute path |
| `cd dev/day04` | a relative path, from where you are |
| `cd ..` | the parent directory |
| `cd ../..` | two levels up |
| `cd ~` or just `cd` | your home directory |
| `cd -` | the previous directory (and prints it) |

`cd` is a **shell builtin**. Run `type cd` and bash says so. It has to be. A separate program would run as a child process (Day 2), and it could change only its own directory. The shell's directory would stay put. That is the same reason a variable set in a child process never reaches its parent (Day 3).

**Worked example.** You are in `/home/ada/dev/day04/src` and want `/home/ada/dev/day03`.

- Absolute: `cd /home/ada/dev/day03`, or `cd ~/dev/day03`
- Relative: `cd ../../day03` (up to `day04`, up to `dev`, down into `day03`)

Type `cd ~/de` and press Tab. Bash completes `~/dev/`. Press Tab twice when several matches exist, to list them.

**Mistakes.** Spaces split arguments, so `cd My Projects` fails with "too many arguments." Use `cd "My Projects"` or `cd My\ Projects`, or let Tab add the backslash. Case matters: `cd Dev` is not `cd dev`. In WSL, `cd /mnt/c/Users/<you>` works, but Linux tools run slowly there (Day 3).

---

## 4. Create and copy

### 4.1 `mkdir` and `touch`

**Intuition.** `mkdir` builds a new drawer. `touch` slips an empty sheet of paper into a drawer, or re-stamps the date on a sheet that is already there.

**Definition.** `mkdir name` creates a directory. `-p` creates any missing parents, and does not complain if the directory exists. `touch name` creates an empty file if none exists. If the file exists, `touch` updates its modification time and leaves the contents alone.

```bash
mkdir -p ~/dev/day04/habit-tracker/src
cd ~/dev/day04
mkdir -p habit-tracker/{tests,docs}      # bash brace expansion: two folders at once
touch habit-tracker/README.md habit-tracker/src/app.py
touch habit-tracker/tests/.gitkeep habit-tracker/docs/.gitkeep
ls -aR habit-tracker
```

Bash expands `{tests,docs}` into two paths before `mkdir` runs. A `.gitkeep` is an empty placeholder. Git does not store empty directories, so teams add one to keep a folder in the repo.

**Mistakes.** `mkdir my project` creates two directories, `my` and `project`. Quote names with spaces. Without `-p`, `mkdir a/b/c` fails if `a/b` does not exist. `touch` never opens an editor and never empties a file. To edit, use `code file` or `nano file`. A space inside the braces stops the expansion, so `{tests, docs}` fails.

### 4.2 `cp` and `mv`

**Intuition.** `cp` is a photocopier, and the original stays where it is. `mv` picks the page up and puts it somewhere else. It can also write a new label on the page without moving it.

**Definition.** `cp SOURCE DEST` copies. `cp -r` copies a directory and everything inside it. `mv SOURCE DEST` moves or renames. The two operations are the same thing, because a name is just a directory entry. Useful flags for both: `-i` asks before overwriting, `-v` prints what happened, and `-n` never overwrites. `cp -a` copies recursively and keeps timestamps and permissions.

```bash
cp notes.txt notes-backup.txt          # copy, new name
cp notes.txt docs/                     # copy into a directory, same name
cp -r habit-tracker habit-tracker-old  # copy a whole directory
mv notes-backup.txt archive/           # move into a directory
mv "meeting notes.txt" meeting-notes.txt   # rename
mv *.jpg images/                       # move every .jpg (the shell expands the glob)
```

Two results surprise people. First, `cp -r photos backup` creates `backup` as a copy of `photos` if `backup` does not exist. If `backup` already exists, the same command creates `backup/photos` instead. Running the same command twice gives two results. Second, inside one filesystem, `mv` only rewrites a directory entry, so moving a 10 GB file takes no time. Moving from `~` to `/mnt/c` crosses filesystems. Then `mv` must copy every byte and delete the original.

**Mistakes.** Both `cp` and `mv` overwrite an existing destination silently. `mv draft.txt final.txt` destroys the old `final.txt` without a warning. Add `-i` when you are unsure. Forgetting `-r` gives `cp: -r not specified; omitting directory`. A typo in the destination directory renames your file to that typo instead of moving it. Deployment often relies on this. A release is copied into a fresh directory and then renamed into place, so users never see a half-copied release.

---

## 5. Delete carefully

**Intuition.** A paper shredder with no bin underneath. The Linux command line has no Recycle Bin.

**Definition.** `rm file` removes files. `rm -r dir` removes a directory and everything in it. `rm -i` asks about each file. `rm -f` forces removal and suppresses prompts and errors. `rmdir dir` removes only an empty directory, so it is a safe choice when you expect the folder to be empty.

A safe routine:

```bash
pwd                      # 1. confirm where you are
ls *.tmp                 # 2. preview exactly what the glob matches
rm -i *.tmp              # 3. delete, confirming each one
rmdir old-empty-folder   # fails safely if anything is still inside
```

A file named `-notes` looks like a flag. Use `rm -- -notes` (the `--` means "no more options") or `rm ./-notes`.

**Mistakes.** One extra space is a disaster. `rm * .txt` (with a space after `*`) deletes every file in the folder, then complains that `.txt` does not exist. `rm -rf` on the wrong path cannot be undone. GNU `rm` refuses `rm -rf /` by default, but `rm -rf /*` is not protected. Some people add `alias rm='rm -i'` to `~/.bashrc`. It helps, but it trains you to rely on a prompt that will not exist on a server. Version control (Day 6) and backups protect you. Production incidents often trace back to a cleanup command run in the wrong directory, which is why teams review cleanup scripts before they run them.

---

## 6. Read files

**Intuition.** `cat` dumps the whole letter onto the table at once. `less` hands it to you one page at a time, with search.

**Definition.** `cat` (concatenate) prints one or more files, one after another. `cat -n` numbers the lines. `cat -A` shows invisible characters: line ends appear as `$`, tabs as `^I`, and Windows carriage returns as `^M`. `less` is a **pager**. It opens a file for scrolling and searching without loading the whole thing on screen, so it works on files far larger than your terminal.

| Key | Action |
|---|---|
| Space / b | next / previous page |
| Up, Down, or j, k | one line |
| `g` / `G` | top / bottom of the file |
| `/word` then Enter | search forward for "word" |
| `n` / `N` | next / previous match |
| `q` | quit |
| `h` | help |

```bash
cat /etc/hostname                  # short file: cat is fine
cat -n /etc/os-release             # with line numbers
cat part1.txt part2.txt            # two files, one after the other
less /etc/passwd                   # one line per user account; type /ada to find yours
less /var/log/dpkg.log             # package installs, including yesterday's
less -N /var/log/apt/history.log   # what apt installed, with line numbers
```

In `/var/log/dpkg.log`, press `G` to jump to the newest entries, then type `?install` and Enter to search backward.

**Mistakes.** `cat` on a big log floods the terminal, so use `less`. `cat` on a binary file such as `/usr/bin/ls` prints garbage and can scramble the terminal. Type `reset` and press Enter to recover. Being stuck in `less` is the most common beginner panic. Press `q`. On the job, SREs read logs with `less` and its search, and `less +F` (follow mode) watches a log as it grows during an incident.

---

## 7. Get help without leaving the terminal

**Intuition.** A reference book bolted to every Linux machine, written for the exact versions of the tools installed there.

**Definition.** `man command` opens the manual page for a command in `less`. Pages are grouped into numbered sections: 1 for user commands, 5 for file formats, 7 for overviews, and 8 for administration commands. When a name exists in several sections, say which one you want. `man passwd` shows the command, and `man 5 passwd` shows the format of `/etc/passwd`. `man -k word` searches page descriptions. Most commands also accept `--help`, which prints a short summary. Builtins such as `cd` have no separate man page in many setups. Use `help cd` for those.

```
cp [OPTION]... [-T] SOURCE DEST
cp [OPTION]... SOURCE... DIRECTORY
```

Read the SYNOPSIS like this. Square brackets mean optional, and `...` means "one or more." The second line says to give one or more sources and a directory last, and copy all of them into it. Type `/-i` and Enter to jump to the description of `-i`.

```bash
man -k "copy files"     # which commands copy files?
man 5 passwd            # the file format, not the command
ls --help               # the quick version
help cd                 # builtin help
type ls                 # alias, builtin, or file?
```

**Mistakes.** Reading a man page top to bottom like a novel. Skim NAME, SYNOPSIS and the flag you need, then search with `/`. Man pages are reference material and often lack examples. Pair them with `tldr`, a community project of example-first summaries (`sudo apt install tldr`). If `man` says the system was minimized, run `sudo unminimize` to restore the documentation. A web tutorial may describe a different version from the one installed. The local man page describes yours. On locked-down production servers there is often no browser and no internet, so `man` and `--help` may be the only documentation you have.

---

## 8. Hands-on exercises (about 60 minutes)

Do these in your Ubuntu tab. Write each answer, and the command that found it, in a notes file. You will reuse them in the project.

1. **Navigation drill (10 minutes).** Starting from `~`, using only relative paths, go to `/etc`, then `/var/log`, then back to `~`. Do it again with `cd -` to bounce between the last two. *Check:* `pwd` matches each target, and you never typed a path that starts with `/`.
2. **Scavenger hunt (20 minutes).** Find your Ubuntu version (`/etc/os-release`), your user's line in `/etc/passwd` and the shell it lists, your CPU model (`less /proc/cpuinfo`, then search `/model name`), your total memory (`/proc/meminfo`), and the most recent package installed (`/var/log/dpkg.log`). *Check:* five answers, each with the command that found it.
3. **Manual practice (10 minutes).** Use only `man` and `--help`. Find out what `ls -S` does, which `cp` flag keeps timestamps, what section 8 of the manual contains (`man man`), and what `/dev/null` is (`man null`). *Check:* you did not use a web search.
4. **Create, copy, move, delete (15 minutes).** In `~/dev/drill`, create `a/b/c` with one command. Create three empty files in `c`. Copy `a` to `a-copy`, rename one file, move one file up two levels, and delete `a-copy` after previewing it with `ls -R`. *Check:* `ls -R ~/dev/drill` shows what you expected, and you predicted the output before you ran each command.
5. **The `cp -r` trap (5 minutes).** Run `cp -r a backup` twice, then `ls -R backup`. *Check:* you can explain why `backup/a` appeared the second time.

---

## 9. Project: a project folder and filesystem field guide, built from the CLI

**Goal.** Show that you can create, organize and inspect files using only the command line, and that you understand the Linux layout well enough to explain it to someone else.

**Repo name:** `100-days-software-day-04-linux-cli-i`

**Repository layout (finished state):**

```
100-days-software-day-04-linux-cli-i/
├── README.md
├── commands.md              # every command you ran, with a one-line explanation
├── fhs-field-guide.md       # ten directories you explored and what you found
├── cheatsheet.md            # your own reference for today's commands
├── organized/               # the "inbox" after you sorted it
│   ├── documents/
│   ├── images/
│   ├── music/
│   └── archive/
└── habit-tracker/           # a skeleton for your Day 1 app
    ├── README.md
    ├── src/
    │   └── app.py
    ├── tests/
    │   └── .gitkeep
    └── docs/
        └── .gitkeep
```

**Starter: create a messy inbox.** Paste this into your Ubuntu terminal:

```bash
mkdir -p ~/dev/100-days-software-day-04-linux-cli-i/inbox
cd ~/dev/100-days-software-day-04-linux-cli-i/inbox
touch report-final.docx report-final-v2.docx budget.xlsx notes.txt todo.txt
touch holiday.jpg beach.jpg screenshot.png song.mp3 podcast.mp3
touch "meeting notes.txt" .old-config
```

**Step-by-step tasks:**

1. Run `ls` and then `ls -la` in `inbox`. Note the file that appears only with `-a`.
2. From the repo folder, create `organized/{documents,images,music,archive}` with one `mkdir -p` command.
3. Rename `meeting notes.txt` to `meeting-notes.txt`.
4. Move documents (`.docx`, `.xlsx`, `.txt`) into `organized/documents`, images (`.jpg`, `.png`) into `organized/images`, and audio into `organized/music`, using globs. Preview each glob with `ls` first.
5. Treat `report-final.docx` as an old version. Move it to `organized/archive`. Copy `report-final-v2.docx` to `organized/archive/report-final-v2-backup.docx`.
6. Delete `.old-config` with `rm -i`, then remove the empty `inbox` with `rmdir`. If `rmdir` refuses, run `ls -a inbox` and find what you missed.
7. Create the `habit-tracker` skeleton with `mkdir -p` and `touch`. Add `.gitkeep` files to its empty folders, and to the empty `organized/` subfolders too. GitHub drops empty folders.
8. Open the folder with `code .` and write `commands.md`. List every command from steps 1 to 7, in order, with one line saying what each did. Run `history` to recover any you forgot.
9. Write `fhs-field-guide.md`. For ten directories from section 2, record what lives there, one real file you found inside (with the command you used), and whether you could read it as a normal user.
10. Write `cheatsheet.md` in your own words. For each command from today, list its three most useful flags and one mistake to avoid.
11. Run `explorer.exe .`, create the GitHub repo, and upload everything with **Add file > Upload files** by dragging the folders in. From Day 6 you will push with `git` instead.

**README checklist:**

- [ ] What this repo shows, in two sentences
- [ ] The before (`ls -la inbox`) and after (`ls -R organized`) listings, pasted as code blocks
- [ ] Links to `commands.md`, `fhs-field-guide.md` and `cheatsheet.md`
- [ ] One mistake you made today and how you recovered (for example, a `cp -r` surprise or a failed `rmdir`)

**Done when:**

- No files remain uncategorized, and the `inbox` folder is gone.
- `commands.md` lists every command needed to reproduce the result from the starter block, in order.
- `fhs-field-guide.md` covers ten directories, each with a real example file and the command used.
- The empty folders appear on GitHub because each holds a `.gitkeep`.
- Someone could follow `commands.md` on a fresh Ubuntu machine and end up with the same tree.

---

## 10. Check your understanding

1. Why can't `cd` be an ordinary program in `/usr/bin` the way `ls` is?
2. You are in `/home/ada/dev/app/src`. Write the shortest relative path to `/home/ada/notes.txt`, and say when you would prefer the absolute path.
3. A teammate typed `rm * .log` intending to delete the log files. What happened, and what habit would have prevented it?
4. You run `cp -r config config-backup` on Monday and again on Tuesday. What does `config-backup` contain after Tuesday, and why?
5. Moving a 20 GB video from `~/videos` to `~/archive` is instant, but moving it to `/mnt/c/Users/ada/Videos` takes minutes. Explain the difference.
6. An app on a server is failing. Which two top-level directories would you check first, and what would you look for in each?
7. `man passwd` and `man 5 passwd` show different pages. Why does Linux need both?

**Answers (check after you try):**

1. A separate program runs as a child process with its own working directory. It could change only its own directory and then exit, which leaves the shell where it was. So `cd` must be built into the shell.
2. `../../../notes.txt` (up from `src` to `app`, to `dev`, to `ada`), or `~/notes.txt`, which is shorter. Prefer absolute paths in scripts and documentation, where the starting directory is unknown.
3. The space made `*` a separate argument, so `rm` deleted every file in the directory, then reported that `.log` does not exist. Preview globs with `ls` first, check `pwd`, and use `rm -i` when you are unsure.
4. On Monday, `config-backup` was created as a copy of `config`. On Tuesday it already existed, so `cp` copied `config` into it. You now have the Monday copy plus `config-backup/config`. Use a dated destination name, or delete the old backup first.
5. Within one filesystem, `mv` only rewrites a directory entry. `/mnt/c` is a different filesystem (the Windows drive), so `mv` has to copy all 20 GB and then delete the original.
6. `/var/log` for recent errors from the app or the system, and `/etc` for its configuration (a wrong setting or a recent change).
7. Names collide across sections. Section 1 documents the `passwd` command, and section 5 documents the `/etc/passwd` file format. Sections keep both available.

---

## 11. Sources to read and watch

**Read:**
- [Wikipedia: Filesystem Hierarchy Standard](https://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard)
- [Wikipedia: Unix](https://en.wikipedia.org/wiki/Unix) (history section)
- [Wikipedia: Ls](https://en.wikipedia.org/wiki/Ls)
- [Wikipedia: Man page](https://en.wikipedia.org/wiki/Man_page)
- [GNU Coreutils manual](https://www.gnu.org/software/coreutils/manual/), the official documentation for `ls`, `cp`, `mv`, `rm`, `cat`, `touch`, `mkdir` and `pwd`
- [man7.org: Linux man pages online](https://man7.org/linux/man-pages/), for example the [ls(1) page](https://man7.org/linux/man-pages/man1/ls.1.html)
- *Filesystem Hierarchy Standard 3.0*: search the title on refspecs.linuxfoundation.org for the full specification.
- [The Linux Command Line by William Shotts](https://linuxcommand.org/tlcl.php), a free book. Chapters 1 to 5 match today.

**Tutorials:**
- [Ubuntu: The Linux command line for beginners](https://ubuntu.com/tutorials/command-line-for-beginners)
- GeeksforGeeks: search "Basic Linux Commands" and "Linux File Hierarchy Structure".
- [explainshell.com](https://explainshell.com/): paste any command to see each part explained from its man page.

**Watch:**
- [MIT Missing Semester: Course overview + the shell](https://missing.csail.mit.edu/2020/course-shell/) (the lecture video is linked on the page).
- AT&T Tech Channel, *The UNIX System: Making Computers More Productive* (1982). Search the title on YouTube to hear Ritchie, Thompson and Kernighan explain the design you used today.
- Search YouTube for "Linux Directories Explained in 100 Seconds" (Fireship) for a fast tour of the hierarchy.

**Practice:**
- [OverTheWire: Bandit](https://overthewire.org/wargames/bandit/). Connect with `ssh bandit0@bandit.labs.overthewire.org -p 2220` and complete levels 0 to 4. They test today's skills: hidden files, names with spaces, and a file named `-`.

**Note on links:** open each one once. If a page has moved, search its title.

---

## 12. Tomorrow

Day 5, **Linux CLI II**, turns today's single commands into tools that work together. You will search inside files with `grep`, search the tree with `find`, chain commands with pipes, send output into files with redirection, control access with permissions and `chmod`, manage processes with signals, and write your first real shell script. Today's `ls -l` permission strings, the PIDs in `/proc` and the logs in `/var/log` are exactly what you will work with tomorrow.
