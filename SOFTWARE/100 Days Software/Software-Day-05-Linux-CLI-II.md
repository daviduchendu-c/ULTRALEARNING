# Day 5: Linux CLI II

*100 Days of Software Engineering, Module: Foundations*

**Time:** about 4 hours
**You will build:** `logscan.sh`, a tested shell script that summarizes a log file, lists the most common errors, and returns exit codes that a CI pipeline can read. It goes in a GitHub repo.
**You should know:** Day 4 (moving around, managing files, `less`, `man`, reading `ls -l`) and Day 3 (PATH, environment variables, VS Code connected to WSL). Work in your **Ubuntu** tab.

---

## 1. The big idea

Yesterday you ran single commands. Today you combine them, and that is where the command line becomes useful. "Which of our 40 config files mention the old database host?" is one `grep`. "Which files over 100 MB did we create this week?" is one `find`. "What were the five most common errors in last night's log?" is one line of small commands joined with pipes. Save that line in a script, and an hour of clicking becomes a command anyone on the team can run, and a CI server can run on every commit.

Along the way you learn **permissions** (who can read, change and run a file) and **processes and signals** (how to see what is running and stop it cleanly).

---

## 2. Search with grep and find

First, make some practice data. This uses Day 4 commands and `printf`, which prints formatted text:

```bash
mkdir -p ~/dev/day05 && cd ~/dev/day05
printf 'INFO app started\nERROR db timeout\nINFO user login\nWARN slow query\nERROR db timeout\nERROR disk full\n' > app.log
cat app.log
```

### 2.1 `grep`

**Intuition.** Ctrl+F for every file at once, from the terminal, with a much stronger pattern language.

**Definition.** `grep PATTERN FILE...` prints every line that matches the pattern. The pattern is a **regular expression** (regex). These flags come up constantly:

| Flag | Meaning |
|---|---|
| `-i` | ignore case |
| `-n` | show line numbers |
| `-c` | count matching lines instead of printing them |
| `-v` | invert: print lines that do not match |
| `-w` | match whole words only |
| `-r` | search directories recursively |
| `-l` | print only the names of matching files |
| `-E` | extended regex, which allows alternation such as `"ERROR\|WARN"` |

Regex basics: `^` is the start of a line, `$` is the end, `.` matches any one character, `*` means "zero or more of the previous thing," and `[0-9]` matches any digit.

```bash
grep ERROR app.log              # 3 lines
grep -c ERROR app.log           # 3
grep -v INFO app.log            # everything except INFO lines
grep -E "ERROR|WARN" app.log    # errors and warnings
grep -n "^ERROR db" app.log     # lines starting with "ERROR db", numbered: 2 and 5
grep -i "error" app.log         # matches ERROR, Error and error
grep -rn "PATH" ~/.bashrc ~/.profile   # where is PATH set?
```

`grep` also reports through its exit status: 0 if something matched, 1 if nothing matched, and 2 if there was an error, such as a missing file. Scripts depend on that status, and so will your project.

**Mistakes.** Unquoted patterns with spaces: `grep db timeout app.log` searches for `db` in two files named `timeout` and `app.log`. Write `grep "db timeout" app.log`. Matching too much: `grep -c ERROR` also counts `INFO no ERRORS today`. Anchor the pattern (`^ERROR`) or use `-w`. `grep -r` can be slow on large trees. Ripgrep (`rg`) is faster, but `grep` is on every Linux machine.

### 2.2 `find`

**Intuition.** `grep` looks inside files. `find` looks for the files themselves, by name, type, size, age or owner. It works like a librarian who searches the catalogue rather than reading every book.

**Definition.** `find START_DIR TESTS... ACTION` walks the tree from a starting directory and applies the tests to each entry. Common tests are `-name "pattern"` (case-sensitive; use `-iname` to ignore case), `-type f` for files, `-type d` for directories, `-size +10M` (larger than 10 MiB), `-mtime -1` (modified in the last 24 hours) and `-maxdepth N`. The default action is to print the path. `-exec CMD {} +` runs a command on the results, and `-delete` removes them.

```bash
find ~ -name "*.log"                         # every .log under your home
find /etc -maxdepth 1 -type f -name "*.conf" # config files directly in /etc
find ~ -type f -mtime -1                     # files changed in the last day
find /usr -type f -size +50M 2>/dev/null     # large files; hide "Permission denied"
find ~/dev -type d -name "__pycache__"       # Python bytecode caches from Day 3
find ~/dev -name "*.log" -exec grep -l ERROR {} +   # which log files contain ERROR?
```

In the last line, `{}` stands for each path that `find` locates, and `+` passes many paths to one `grep` call.

**Mistakes.** Forgetting to quote the pattern. `find . -name *.log` lets the shell expand `*.log` first, using files in the current directory, so `find` searches for the wrong thing or fails. Always write `-name "*.log"`. With `-delete`, mistakes are permanent. Run the command without `-delete` first, read the list, then add `-delete` at the end. Searching from `/` as a normal user floods the screen with "Permission denied." `2>/dev/null` hides those errors.
---

## 3. Streams and pipes

**Intuition.** Every program has three hoses. One takes input (stdin). Two send output: stdout for results, and stderr for complaints. By default all three connect to your terminal. Redirection plugs a hose into a file, and a pipe plugs one program's output into the next program's input.

**Definition.** Stdin is stream 0, stdout is stream 1, and stderr is stream 2. Redirection changes where they point.

| Syntax | Meaning |
|---|---|
| `cmd > file` | stdout to file, replacing its contents |
| `cmd >> file` | stdout to file, appending |
| `cmd < file` | stdin from file |
| `cmd 2> file` | stderr to file |
| `cmd > file 2>&1` | stdout to file, then stderr to wherever stdout now goes |
| `cmd &> file` | bash shorthand for both streams to a file |
| `cmd > /dev/null` | throw the output away |

```bash
ls /etc /nope > out.txt 2> err.txt   # listing goes to out.txt, the error to err.txt
cat err.txt                          # ls: cannot access '/nope': No such file or directory
echo "first" > notes.txt             # create or overwrite
echo "second" >> notes.txt           # append
wc -l < app.log                      # 6 (wc reads stdin, so no filename is printed)
grep ERROR app.log > errors.txt 2>&1 # results and any errors, in one file
```

A pipe, written `cmd1 | cmd2`, sends the stdout of `cmd1` to the stdin of `cmd2`. Both run at the same time as separate processes, and the data streams through without a temporary file. Common filters are `sort` (`-n` for numbers, `-r` for reverse), `uniq` (merges adjacent duplicate lines, and `-c` counts them), `wc -l`, `head -n N`, `tail -n N`, `cut -d: -f1` (picks fields split by a delimiter), `tr` and `tee file` (saves a copy while passing data on).

Which login shells do the accounts on this machine use?

```bash
cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
```

```
     18 /usr/sbin/nologin
      2 /bin/bash
      1 /bin/sync
```

Your numbers will differ. `cut` takes field 7 (the shell) from each line. `sort` groups identical shells together, `uniq -c` counts each group, and `sort -rn` puts the biggest count first. The same shape works on the practice log:

```bash
grep ERROR app.log | sort | uniq -c | sort -rn | head -n 3
```

```
      2 ERROR db timeout
      1 ERROR disk full
```

**Mistakes.** `uniq` merges only duplicates that sit next to each other, so `uniq -c` without a `sort` first gives wrong counts. `>` truncates the file before the command runs. So `sort app.log > app.log` leaves you with an empty file, because the shell empties it and then `sort` reads nothing. Write to a new file and `mv` it back. Order matters with `2>&1`. `cmd > file 2>&1` captures both streams. `cmd 2>&1 > file` sends stderr to the terminal, because `2>&1` runs while stdout still points at the terminal. A pipeline's exit status is the status of its last command, so an early failure is invisible unless you set `set -o pipefail` in scripts (section 6). Once a pipeline passes four or five stages, move it into a script with comments.

---

## 4. Permissions and chmod

**Intuition.** An office where every room has three locks: one for the owner, one for the owner's team, and one for every visitor. Each lock decides separately whether you can look inside, change things, or walk through.

**Definition.** Every file has an **owner** and a **group**, plus three permission sets: owner (`u`), group (`g`) and others (`o`). Each set holds read (`r`), write (`w`) and execute (`x`). The first ten characters of `ls -l` show them:

```
 -  rwx  r-x  r--     ada  devs   logscan.sh
 │   │    │    │
 │   │    │    └── others: read only
 │   │    └─────── group:  read and execute
 │   └──────────── owner:  read, write, execute
 └──────────────── type: - file, d directory, l symlink
```

For files, `r` means read the contents, `w` means change them, and `x` means run them as a program. For directories, the meanings shift. `r` lets you list names. `w` lets you create, delete and rename entries inside. `x` lets you enter the directory with `cd` and reach the files in it. The root user skips these checks, which is why `sudo` is powerful and risky.

```bash
ls -l /etc/shadow        # -rw-r----- 1 root shadow ... (password hashes)
cat /etc/shadow          # cat: /etc/shadow: Permission denied
ls -l /etc/passwd        # -rw-r--r-- 1 root root ...  (everyone can read)
umask                    # often 0022 or 0002: bits removed from new files
touch new.txt && ls -l new.txt   # -rw-r--r-- if umask is 0022
```

New files start from `666` (read and write for everyone) minus the umask, so a umask of `0022` gives `644`. New directories start from `777`, which gives `755`.

**Mistakes.** Deleting a file depends on write permission on the directory, not on the file. A read-only file can still be deleted from a directory you can write to. These classic bits are coarse, with one owner and one group. Finer control uses ACLs (`man setfacl`). SSH refuses to use a private key that others can read, which is why most cloud engineers have run `chmod 600` on a key at least once.

### 4.1 `chmod`

**Definition.** `chmod MODE FILE` changes the permission bits. You can use symbolic mode (`who`, then `+`, `-` or `=`, then what) or octal mode, where each digit adds up r=4, w=2, x=1.

| Octal | Bits | Typical use |
|---|---|---|
| `755` | `rwxr-xr-x` | scripts, programs and directories |
| `644` | `rw-r--r--` | ordinary files |
| `700` | `rwx------` | private directories |
| `600` | `rw-------` | secrets and SSH keys |

Only the owner, or root, can change a file's mode.

```bash
printf '#!/usr/bin/env bash\necho "Hello from a script"\n' > hello.sh
./hello.sh               # bash: ./hello.sh: Permission denied
ls -l hello.sh           # -rw-r--r-- (with umask 0022)
chmod u+x hello.sh       # add execute for the owner
ls -l hello.sh           # -rwxr--r--
./hello.sh               # Hello from a script
chmod 755 hello.sh       # -rwxr-xr-x
chmod go-r secret.txt    # remove read from group and others (if secret.txt exists)
mkdir private && chmod 700 private
```

Work out `754` yourself: 7 is `rwx`, 5 is `r-x`, and 4 is `r--`, so the result is `rwxr-xr--`.

**Mistakes.** `chmod -R 777` "to fix a permission error" lets every user change or replace your files, including web-facing files. Find out which user needs which access instead. `chmod -R 644` on a directory tree removes `x` from the directories too, so nobody can enter them. Uploading a script through the GitHub website does not keep the executable bit, so the README must tell users to run `chmod +x`. Git records that bit for you from Day 6 on.

---

## 5. Processes, job control and signals

**Intuition.** Day 2 showed that every running program is a process. Now you get the control room: a list of everything running, and buttons to pause, resume and move work to the background.

**Definition.** `ps aux` lists processes with their user, PID, CPU and memory use. `ps -ef` adds the **PPID**, the parent that started each one. `top` shows a live view (`q` quits), and `pgrep -a NAME` finds processes by name. **Job control** manages the commands your shell started. `&` runs a command in the background, `jobs` lists them, `fg` and `bg` move them between foreground and background, and Ctrl+Z pauses the foreground job. `$$` holds your shell's PID.

```bash
echo $$                  # your shell's PID, for example 412
sleep 300 &              # [1] 2087   (job number 1, PID 2087)
sleep 600 &              # [2] 2088
jobs                     # lists both as Running
pgrep -a sleep           # 2087 sleep 300
                         # 2088 sleep 600
ps -o pid,ppid,cmd -p 2087   # its PPID is your shell's PID
ls /proc/2087            # Day 4's /proc, holding this process's details
fg %1                    # sleep 300 now holds your terminal
                         # press Ctrl+Z: [1]+ Stopped  sleep 300
bg %1                    # it continues in the background
```

Use the PIDs your machine prints, not the ones shown above.

**Mistakes.** Closing the terminal usually ends its background jobs (SIGHUP, below). Long-running services belong under a supervisor such as systemd. A paused job still holds its memory, files and ports, which is why a port can stay "already in use."

### 5.1 Signals

**Intuition.** Signals are taps on a process's shoulder. Most are polite requests, such as "please wrap up," and the process can respond in its own way. SIGKILL is different. It pulls the power cord.

**Definition.** A signal is a numbered message that the kernel, another process (through `kill`), or your keyboard sends to a process. The common ones on Linux are below. The numbers match x86 and ARM. Use the names in scripts.

| Signal | Number | Sent by | Default effect | Can be caught? |
|---|---|---|---|---|
| SIGHUP | 1 | terminal closed | terminate | yes |
| SIGINT | 2 | Ctrl+C | terminate | yes |
| SIGQUIT | 3 | Ctrl+\\ | terminate with core dump | yes |
| SIGKILL | 9 | `kill -9` | terminate immediately | **no** |
| SIGTERM | 15 | `kill` (default) | terminate | yes |
| SIGCONT | 18 | `fg`, `bg`, `kill -CONT` | resume | yes |
| SIGSTOP | 19 | `kill -STOP` | pause | **no** |
| SIGTSTP | 20 | Ctrl+Z | pause | yes |

"Caught" means the program can run its own handler, for example to save work and close files before it exits. SIGKILL and SIGSTOP cannot be caught, blocked or ignored. When a process dies from a signal, the shell reports exit status 128 plus the signal number. That gives 130 after Ctrl+C, 137 after SIGKILL and 143 after SIGTERM.

Save this as `polite.sh`, run `chmod +x polite.sh`, then run `./polite.sh &`:

```bash
#!/usr/bin/env bash
trap 'echo "Got SIGTERM: saving and exiting"; exit 0' TERM
trap 'echo "Got SIGINT: exiting"; exit 130' INT
echo "Running as PID $$"
while true; do
  sleep 1
done
```

```bash
kill -TERM <PID>     # prints "Got SIGTERM: saving and exiting" within about a second
./polite.sh &        # start it again
kill -KILL <PID>     # no message: SIGKILL cannot be trapped
kill -l              # list every signal name and number
```

The short delay happens because bash finishes the current `sleep 1` before it runs the trap.

**Mistakes.** Reaching for `kill -9` first. SIGKILL gives the program no chance to flush data or close database connections, so it can leave half-written files behind. Send SIGTERM, wait a few seconds, and use SIGKILL only if the process is still running. Check a PID with `ps -p PID` before you kill it. `docker stop` sends SIGTERM, then SIGKILL after 10 seconds by default, and Kubernetes waits 30 seconds by default. Services must handle SIGTERM to shut down without dropping requests. Exit code 137 in a container log usually means SIGKILL, often from running out of memory.

---

## 6. Shell scripting foundations

**Intuition.** A script is a recipe card for the shell. Anything you would type can be written down once and run again, the same way, on any Linux machine.

**Definition.** A shell script is a text file of commands. The **shebang** line `#!/usr/bin/env bash` tells the kernel to run the file with bash, and `env` finds bash through PATH. Make the file executable with `chmod +x`, then run it as `./script.sh`. The current directory is not on PATH, so the `./` is required.

```bash
#!/usr/bin/env bash
set -euo pipefail        # stop on errors, on unset variables, and on failures inside pipes

name="Ada"                       # variables: NO spaces around =
echo "Hello, $name"              # double quotes expand variables
echo 'Hello, $name'              # single quotes do not: prints $name literally
today="$(date +%Y-%m-%d)"        # command substitution
echo "Script: $0, first arg: ${1:-none}, arg count: $#"   # ${1:-none}: default if $1 is missing

if [ -f "app.log" ]; then        # test: does the file exist?
  echo "found the log"
elif [ -d "logs" ]; then
  echo "found a logs directory"
else
  echo "no log" >&2              # error messages go to stderr
fi

for file in *.log; do            # loop over a glob
  echo "$file has $(wc -l < "$file") lines"
done

while read -r line; do           # read a file line by line
  echo "> $line"
done < app.log

greet() {                        # functions take arguments as $1, $2...
  local who="$1"
  echo "Hi, $who"
}
greet "team"

exit 0                           # 0 = success; any other number = failure
```

Common tests are `-f` (file exists), `-d` (directory exists), `-r` (readable), `-s` (exists and is not empty), `-z "$x"` (string is empty), `"$a" = "$b"` (strings are equal) and `"$n" -gt 5` (number is greater than 5). `&&` runs the next command only if this one succeeded. `||` runs it only if this one failed. `$?` holds the last exit status.

**Why quoting matters.**

```bash
file="my notes.txt"
touch "$file"
rm $file      # unquoted: tries to delete "my" and "notes.txt", two wrong files
rm "$file"    # quoted: deletes the one file you meant
```

A worse case: if `$BUILD_DIR` is empty or unset (a typo is enough), `rm -rf "$BUILD_DIR"/*` becomes `rm -rf /*`. Steam's Linux client shipped a bug of this shape in 2015, and it deleted users' files. The defense is `rm -rf "${BUILD_DIR:?}"/*`, which aborts with an error if the variable is empty, combined with `set -u`.

**Mistakes.** Spaces around `=`: `name = "Ada"` runs a command called `name`. Unquoted variables are the most common shell bug. A script saved with Windows (CRLF) line endings fails with `/usr/bin/env: 'bash\r': No such file or directory`. Check with `cat -A script.sh` (CRLF lines end in `^M$`), then fix it with `sed -i 's/\r$//' script.sh`, or switch the file to LF in VS Code's status bar. `set -e` does not trigger inside `if` conditions or on the left side of `&&`, so use it as a safety net and nothing more. Run **ShellCheck** (`sudo apt install shellcheck`, then `shellcheck script.sh`) to catch unquoted variables (SC2086) and the empty-variable `rm` danger (SC2115). Once a script passes roughly 100 lines or needs real data structures, rewrite it in Python (from Day 9 onward).

---

## 7. Hands-on exercises (about 60 minutes)

Work in `~/dev/day05` with the `app.log` from section 2.

1. **grep (10 minutes).** Count the ERROR lines. Print the non-INFO lines with line numbers. Find every line that mentions `db`, regardless of case. Then run `grep -q NOPE app.log; echo $?`. *Check:* you get 3, four numbered lines, two lines, and `1`.
2. **find (10 minutes).** List every `.sh` and `.log` file under `~/dev`. Find directories named `.venv` from Day 3. Find files in `~` modified in the last 60 minutes with `-mmin -60`. *Check:* `app.log` appears in the last result.
3. **Redirection (10 minutes).** Run `ls /etc /nope`, sending the listing to `ok.txt` and the error to `bad.txt`. Then reproduce the trap. Copy `app.log` to `trap.log`, run `sort trap.log > trap.log`, and check with `wc -l trap.log`. *Check:* `bad.txt` holds one line, and `trap.log` is now empty. Explain why in one sentence.
4. **Pipes (10 minutes).** Build the login-shell pipeline from section 3 one stage at a time, and look at the output after each `|`. Then find the three most frequent words in a man page: `man ls | tr -s ' ' '\n' | sort | uniq -c | sort -rn | head -n 3`. *Check:* you can say what each stage contributed.
5. **Permissions (10 minutes).** Create `hello.sh` from section 4.1 and get it from "Permission denied" to running. Create a directory with mode `600` (no `x`) that contains one file, then try `ls` and `cd` on it. *Check:* you can explain why `cd` fails even though you own the directory, then fix it with `chmod 700`.
6. **Processes and signals (10 minutes).** Start `polite.sh` in the background, find it with `pgrep -a polite`, and send SIGTERM. Start it again and send SIGKILL. Then run `sleep 100`, press Ctrl+C, and run `echo $?`. *Check:* the trap message appears only for SIGTERM, and `$?` prints `130`.

---

## 8. Project: `logscan`, a log summary script

**Goal.** Write a shell script that summarizes a log file. It counts entries by level, lists the most common error messages, and returns an exit code that tells automation whether the log crossed an error threshold. Then test it and lint it.

**Repo name:** `100-days-software-day-05-logscan`

**Repository layout:**

```
100-days-software-day-05-logscan/
├── README.md
├── logscan.sh                 # the main script
├── make-sample-log.sh         # generates realistic test data
├── samples/
│   └── sample.log             # generated with make-sample-log.sh
├── tests/
│   ├── test-logscan.sh        # checks exit codes
│   └── fixtures/
│       └── small.log          # tiny log with known counts
├── results/
│   ├── report.txt             # logscan output on samples/sample.log
│   ├── tests.txt              # test output
│   └── shellcheck.txt         # ShellCheck output (ideally empty)
└── .gitattributes             # keeps .sh files with LF line endings
```

**Log format.** Each line holds a date, a time, the level padded to five characters, then the message.

```
2026-10-20 09:00:03 ERROR [db] connection timeout
2026-10-20 09:00:04 INFO  [api] request served
```

**Starter code.** Create every file inside Ubuntu (`mkdir -p ~/dev/100-days-software-day-05-logscan && cd "$_" && code .`; `$_` is the last argument of the previous command). Files created this way get LF line endings.

`make-sample-log.sh`:

```bash
#!/usr/bin/env bash
# make-sample-log.sh: print N fake log lines (default 200).
# Usage: ./make-sample-log.sh [N] > sample.log

set -euo pipefail

count="${1:-200}"
levels=(INFO INFO INFO INFO WARN ERROR)
messages=(
  "[auth] login ok"
  "[auth] login failed"
  "[db] connection timeout"
  "[db] slow query"
  "[api] request served"
  "[api] upstream returned 502"
)

for ((i = 1; i <= count; i++)); do
  level="${levels[RANDOM % ${#levels[@]}]}"
  msg="${messages[RANDOM % ${#messages[@]}]}"
  printf '2026-10-20 09:%02d:%02d %-5s %s\n' $((i / 60 % 60)) $((i % 60)) "$level" "$msg"
done
```

`$RANDOM` returns a new random number each time bash reads it. The array lists `INFO` four times, so INFO lines appear about four times as often as WARN or ERROR lines.

`logscan.sh`:

```bash
#!/usr/bin/env bash
# logscan.sh: summarize a log file by level and list the most common errors.
# Usage: ./logscan.sh [-n TOP] [-t THRESHOLD] LOGFILE
# Exit codes: 0 ok, 1 file problem, 2 usage error, 3 too many errors

set -euo pipefail

TOP=5
THRESHOLD=0   # 0 means "never fail because of the error count"

usage() {
  echo "Usage: $(basename "$0") [-n TOP] [-t THRESHOLD] LOGFILE" >&2
  exit 2
}

is_number() {
  case "$1" in
    ''|*[!0-9]*) return 1 ;;
    *) return 0 ;;
  esac
}

count_level() {
  # grep -c prints 0 and exits 1 when nothing matches; "|| true" keeps set -e calm.
  grep -cE "^[^ ]+ [^ ]+ $1 " "$LOGFILE" || true
}

while getopts ":n:t:h" opt; do
  case "$opt" in
    n) TOP="$OPTARG" ;;
    t) THRESHOLD="$OPTARG" ;;
    h) usage ;;
    :) echo "Option -$OPTARG needs a value" >&2; usage ;;
    \?) echo "Unknown option: -$OPTARG" >&2; usage ;;
  esac
done
shift $((OPTIND - 1))

[ "$#" -eq 1 ] || usage
LOGFILE="$1"

is_number "$TOP" || { echo "-n must be a whole number" >&2; usage; }
is_number "$THRESHOLD" || { echo "-t must be a whole number" >&2; usage; }

if [ ! -f "$LOGFILE" ] || [ ! -r "$LOGFILE" ]; then
  echo "Error: cannot read '$LOGFILE'" >&2
  exit 1
fi

tmp="$(mktemp)"
trap 'rm -f "$tmp"' EXIT
trap 'echo "Interrupted" >&2; exit 130' INT TERM

total=$(wc -l < "$LOGFILE")
info=$(count_level INFO)
warn=$(count_level WARN)
errors=$(count_level ERROR)

echo "Log report for: $LOGFILE"
echo "Generated: $(date '+%Y-%m-%d %H:%M:%S')"
echo
echo "Total lines: $total"
echo "INFO:  $info"
echo "WARN:  $warn"
echo "ERROR: $errors"
echo
echo "Top $TOP error messages:"

grep -E "^[^ ]+ [^ ]+ ERROR " "$LOGFILE" > "$tmp" || true
if [ -s "$tmp" ]; then
  # head may close the pipe early, which makes sort fail with SIGPIPE.
  # That is harmless here, so "|| true" stops set -e from ending the script.
  cut -d' ' -f4- "$tmp" | sort | uniq -c | sort -rn | head -n "$TOP" || true
else
  echo "  (none)"
fi

if [ "$THRESHOLD" -gt 0 ] && [ "$errors" -gt "$THRESHOLD" ]; then
  echo "FAIL: $errors errors is more than the threshold of $THRESHOLD" >&2
  exit 3
fi
```

The new parts work as follows.

- `getopts ":n:t:h"` parses the options. A letter followed by `:` takes a value, which arrives in `$OPTARG`. The leading `:` lets the script print its own error messages. `shift $((OPTIND - 1))` leaves the log file in `$1`.
- The regex `^[^ ]+ [^ ]+ ERROR ` anchors on the level column, so a message that merely contains the word "ERROR" is not counted.
- `cut -d' ' -f4-` keeps the message, so identical messages group together.
- The `EXIT` trap deletes the temporary file however the script ends. The `INT TERM` trap turns Ctrl+C or `kill` into a clean exit, which then fires the `EXIT` trap (section 5.1 in use).

`tests/fixtures/small.log` (2 errors, 1 warning, 2 info lines):

```
2026-10-20 09:00:01 INFO  [auth] login ok
2026-10-20 09:00:02 WARN  [db] slow query
2026-10-20 09:00:03 ERROR [db] connection timeout
2026-10-20 09:00:04 INFO  [api] request served
2026-10-20 09:00:05 ERROR [db] connection timeout
```

`tests/test-logscan.sh`:

```bash
#!/usr/bin/env bash
# test-logscan.sh: check that logscan.sh returns the right exit codes.
# Run from the repo root: ./tests/test-logscan.sh

set -u

pass=0
fail=0

check() {
  # check DESCRIPTION EXPECTED_EXIT_CODE COMMAND [ARGS...]
  local desc="$1" expected="$2"
  shift 2
  "$@" > /dev/null 2>&1
  local actual=$?
  if [ "$actual" -eq "$expected" ]; then
    echo "PASS: $desc"
    pass=$((pass + 1))
  else
    echo "FAIL: $desc (expected $expected, got $actual)"
    fail=$((fail + 1))
  fi
}

check "no arguments prints usage"      2 ./logscan.sh
check "missing file is an error"       1 ./logscan.sh no-such-file.log
check "small log succeeds"             0 ./logscan.sh tests/fixtures/small.log
check "threshold of 1 is exceeded"     3 ./logscan.sh -t 1 tests/fixtures/small.log
check "threshold of 2 is not exceeded" 0 ./logscan.sh -t 2 tests/fixtures/small.log
check "non-numeric -n is rejected"     2 ./logscan.sh -n abc tests/fixtures/small.log

echo
echo "$pass passed, $fail failed"
[ "$fail" -eq 0 ]
```

The last line makes the test script exit 0 only when every check passes, so a CI server (Day 91) can run it as-is.

`.gitattributes`:

```
*.sh text eol=lf
```

This tells Git to store and check out `.sh` files with LF endings, even on Windows. Day 6 covers it properly.

**Step-by-step tasks:**

1. Create the folder and all the files above. Run `chmod +x logscan.sh make-sample-log.sh tests/test-logscan.sh`.
2. Run `./logscan.sh tests/fixtures/small.log`. *Expect:* `INFO: 2`, `WARN: 1`, `ERROR: 2`, and `2 [db] connection timeout` in the top list.
3. Generate data: `mkdir -p samples results && ./make-sample-log.sh 500 > samples/sample.log`. Check it with `wc -l` (500).
4. Save a report: `./logscan.sh -n 3 samples/sample.log > results/report.txt`. Confirm that the INFO, WARN and ERROR counts add up to the total.
5. Try the threshold: `./logscan.sh -t 10 samples/sample.log; echo "exit: $?"`. Expect the FAIL message and `exit: 3`. Then run it with `> /dev/null` and notice the FAIL line still appears, because it goes to stderr.
6. Run the tests and save the output: `./tests/test-logscan.sh | tee results/tests.txt`. *Expect:* `6 passed, 0 failed`.
7. Lint: `sudo apt install shellcheck`, then `shellcheck logscan.sh make-sample-log.sh tests/test-logscan.sh > results/shellcheck.txt`. Fix anything it reports, rerun, and commit the final file, which should be empty.
8. Break it on purpose. Remove the `|| true` from `count_level`. Run the script on a log with no WARN lines. Make one with `grep -v WARN tests/fixtures/small.log > /tmp/nowarn.log`. The script stops silently, because `set -e` sees grep's exit code 1. Restore the line and describe what happened in the README.
9. Write the README, then upload everything to a new GitHub repo with **Add file > Upload files** (use `explorer.exe .` to find the folder). The upload does not keep the executable bit, which is why the README tells users to run `chmod +x`.

**Stretch goals (pick one):**

- Add a `-l LEVEL` option, so `./logscan.sh -l WARN file` lists the top warnings instead of the top errors.
- Make `logscan.sh` accept several log files and print one summary per file.
- Point it at a real log such as `/var/log/dpkg.log`, and adapt the pattern to that format in a copy of the script.

**README checklist:**

- [ ] What `logscan` does, in two sentences
- [ ] Install: `chmod +x *.sh tests/*.sh`
- [ ] Usage with both options, and an example report pasted as a code block
- [ ] A table of exit codes (0, 1, 2, 3) and what each means
- [ ] How to run the tests and ShellCheck
- [ ] What you learned when you broke `set -e` on purpose (step 8). Note that the exit code is 1, the same code the script uses for a file problem.
- [ ] One limitation of the script (for example, it assumes this exact log format, or it reads the file several times, which is slow on very large logs)

**Done when:**

- `./tests/test-logscan.sh` prints `6 passed, 0 failed`, and its output is committed in `results/tests.txt`.
- `results/shellcheck.txt` is committed and contains no unresolved warnings.
- `results/report.txt` shows counts that add up to the total line count of `samples/sample.log`.
- Every `.sh` file starts with a shebang, uses LF line endings (`cat -A logscan.sh | head -n 2` shows no `^M`), and quotes its variables.
- The README explains the exit codes clearly enough that someone could use `logscan` in a CI job without reading the script.

---

## 9. Check your understanding

1. You run `sort data.txt > data.txt` and the file is now empty. Explain exactly when the data was lost.
2. `grep ERROR app.log | uniq -c` reports `1` for an error that appears three times. What is wrong with the pipeline?
3. A file in your home directory has mode `444` (read-only for everyone), yet you delete it with `rm` without `sudo`. How?
4. Your service takes 20 seconds to finish in-flight requests. The platform sends SIGTERM, waits 10 seconds, then sends SIGKILL. What happens to users, and what are two ways to fix it?
5. Why is `find . -name *.txt` sometimes correct and sometimes wrong, depending on the directory you run it in?
6. A script runs `rm -rf "$OUTPUT"/*`, and `OUTPUT` was never set. What happens, and which two changes prevent it?
7. A teammate's script works with `bash deploy.sh` but fails with `./deploy.sh` and `Permission denied`. What is missing, and why does the first form work?

**Answers (check after you try):**

1. Before `sort` started. The shell sets up `> data.txt` first, which truncates the file to zero bytes. Only then does `sort` run, and it reads an empty file.
2. `uniq` merges only adjacent duplicates, and the three ERROR lines were not next to each other. Put `sort` before `uniq -c`.
3. Deleting a file changes the directory, not the file. You have write permission on your home directory, so you can remove entries from it regardless of the file's own mode.
4. The service is killed with SIGKILL at 10 seconds, so requests still in progress are cut off and users see errors. Fix it by handling SIGTERM faster (stop accepting new work and finish quickly), or by raising the platform's grace period above 20 seconds. Ideally do both.
5. Unquoted, the shell expands `*.txt` before `find` runs. In a directory with no `.txt` files, bash passes the pattern through unchanged and `find` works. In a directory with one `.txt` file, `find` searches only for that exact name. With several, `find` fails with an error. Quoting (`"*.txt"`) works everywhere.
6. The command becomes `rm -rf /*`, which tries to delete everything the user can write to. Use `"${OUTPUT:?}"/*`, so the script aborts if the variable is empty, and `set -u`, so unset variables are errors. ShellCheck would flag it too.
7. The execute bit (`chmod +x deploy.sh`) is missing. `bash deploy.sh` runs the bash program and passes the file to it as input, so only read permission is needed. `./deploy.sh` asks the kernel to execute the file itself, which requires `x`.

---

## 10. Sources to read and watch

**Read:**
- [Wikipedia: Grep](https://en.wikipedia.org/wiki/Grep)
- [Wikipedia: Standard streams](https://en.wikipedia.org/wiki/Standard_streams)
- [Wikipedia: File-system permissions](https://en.wikipedia.org/wiki/File-system_permissions)
- [Wikipedia: Signal (IPC)](https://en.wikipedia.org/wiki/Signal_(IPC))
- [GNU Bash Reference Manual](https://www.gnu.org/software/bash/manual/), especially the sections on redirections, pipelines, and the `trap` and `getopts` builtins
- [man7.org: signal(7)](https://man7.org/linux/man-pages/man7/signal.7.html), the full Linux signal list
- [Greg's Wiki: BashPitfalls](https://mywiki.wooledge.org/BashPitfalls), a community reference on common shell mistakes

**Tutorials:**
- [MIT Missing Semester: Shell Tools and Scripting](https://missing.csail.mit.edu/2020/shell-tools/)
- [MIT Missing Semester: Command-line Environment](https://missing.csail.mit.edu/2020/command-line/) (job control and signals)
- [The Linux Command Line by William Shotts](https://linuxcommand.org/tlcl.php): the chapters on redirection, permissions and processes, and Part 4 on shell scripts.

**Watch:**
- Computerphile, "Where GREP Came From": Brian Kernighan explains how `grep` was born. Search the title on YouTube.

**Practice:**
- [OverTheWire: Bandit](https://overthewire.org/wargames/bandit/), levels 5 to 10, which need `find` with size and owner tests, `grep`, and `sort | uniq`.
- [ShellCheck online](https://www.shellcheck.net/): paste any script for instant feedback. The wiki page for each warning code explains it, for example [SC2086](https://www.shellcheck.net/wiki/SC2086).

**Note on links:** open each one once. If a page has moved, search its title.

---

## 11. Tomorrow

Day 6, **Git Mental Model**, starts the Git and Collaboration module. Until now you have uploaded files to GitHub through the website, and you have seen the gaps. Empty folders vanish, the executable bit is lost, and nothing records how a file changed. Git fixes all three. You will learn what a repository, a commit and the staging area are, and you will drive Git from Ubuntu using the terminal skills from Days 4 and 5. Your first real commits can go into today's `logscan` repo, and the `.gitattributes` file you added will finally do its job.
