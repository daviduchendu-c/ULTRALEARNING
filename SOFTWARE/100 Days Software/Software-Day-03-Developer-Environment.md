# Day 3: Developer Environment

*100 Days of Software Engineering, Module: Foundations*

**Time:** about 4 hours, including installs and one restart
**You will build:** a Windows 11 developer setup with VS Code, Git, Python and WSL 2 (Ubuntu), plus a GitHub repo that proves it: an environment check script for each side, a setup log, and a bug you found with the debugger.
**You should know:** Day 2 (processes, storage and memory). You need administrator rights and about 5 GB of free disk space.

---

## 1. The big idea

On your first day at a job, someone will ask you to "get the project running locally." That means an editor, a terminal, the right language versions, the right packages, and settings stored in environment variables. Engineers who can do this in an hour, and fix it when it breaks, become productive quickly. Engineers who cannot lose days to errors like `'python' is not recognized` or `command not found`.

Those errors come from a small set of ideas: where files live, how the terminal finds programs (PATH), how programs read settings (environment variables), and how tools get installed (package managers). Today you install the machine you will use for the next 97 days, including a real Linux system through WSL 2, which Days 4 and 5 need. Then you learn what each part did.

---

## 2. Install the tools

Work in an administrator PowerShell window for the WSL step. Run the other commands in a normal window.

**Editor, Git and Python** (run in PowerShell):

```powershell
winget install --id Microsoft.VisualStudioCode -e
winget install --id Git.Git -e
winget install --id Python.Python.3.14 -e
```

`-e` means "match the ID exactly." Run `winget search python` to see current versions. If 3.14 has been superseded when you read this, install the newest stable release.

**WSL 2 and Ubuntu** (run in Windows Terminal as administrator):

```powershell
wsl --install
```

This enables the Windows features WSL needs, installs Ubuntu (the default distribution), and asks you to restart. After the restart, Ubuntu opens and asks for a **UNIX username and password**. Use a short lowercase username. The password shows no characters while you type. That is normal, and you will need it for `sudo`.

Check the result from PowerShell:

```powershell
wsl --list --verbose     # Ubuntu should show VERSION 2
wsl --update             # get the latest WSL release
```

Then update Ubuntu's packages:

```bash
sudo apt update && sudo apt upgrade -y
```

`&&` runs the second command only if the first succeeds. Day 5 covers it in detail.

**Mistakes.** If `wsl --install` fails with a message about virtualization (error code 0x80370102 is common), virtualization is off in your firmware. Restart into the BIOS or UEFI settings, enable Intel VT-x or AMD-V (SVM), and try again. WSL 2 uses RAM while it runs. `wsl --shutdown` stops it if your laptop feels slow.

---

## 3. The terminal and the filesystem

### 3.1 The terminal

**Intuition.** The terminal is a text conversation with your computer. You say "run this, with these details," and it answers in text. Clicking is like pointing at a menu. Typing a command is like giving the chef exact instructions, which you can write down and repeat.

**Definition.** Three parts are easy to confuse. The **terminal emulator** is the window (Windows Terminal, which ships with Windows 11). The **shell** is the program inside the window that reads and runs your commands. On Windows you have **Windows PowerShell 5.1** (built in), **PowerShell 7** (a separate free install, command `pwsh`), and the older **Command Prompt** (`cmd`). In WSL you use **bash**. A command line has a shape: `command argument --flag`. So `git --version` runs the program `git` with the flag `--version`.

**Worked example.** Try these in PowerShell:

```powershell
Get-Location              # where am I? (alias: pwd)
Get-ChildItem             # what is here? (aliases: ls, dir)
Set-Location Documents    # move into Documents (alias: cd)
Set-Location ..           # move up one level
$PSVersionTable.PSVersion # which PowerShell is this?
```

Three habits save time from day one: **Tab** completes names, **Up arrow** recalls earlier commands, and **Ctrl+C** stops a running command.

**Mistakes.** PowerShell defines aliases named `ls`, `cat` and `rm` so Linux users feel at home, but they are not the Linux tools. `ls -la` fails in PowerShell because `Get-ChildItem` has no such flags. When a tutorial shows a `$` prompt, it means bash, so run that command in your Ubuntu tab, not in PowerShell. Closing a terminal also stops the processes it started, so a server you launched there stops too.

### 3.2 The filesystem

**Intuition.** The filesystem is a family tree of folders. Every file has one full address, like a postal address that runs from country down to street.

**Definition.** Windows has one tree per drive. Each tree starts at a drive letter such as `C:\` and separates parts with backslashes. Linux has one tree that starts at `/` (called root). Other drives appear as folders inside it, and parts are separated by forward slashes. Two differences matter. Windows names are not case-sensitive, so `Notes.txt` and `notes.txt` are the same file. Linux names are case-sensitive, so they are two files. Names that start with a dot, such as `.gitignore`, are hidden by default on Linux.

WSL connects the two trees:

| You are in | To reach the other side |
|---|---|
| Ubuntu (WSL) | Your Windows `C:` drive is at `/mnt/c`, so your Windows home is `/mnt/c/Users/<you>` |
| Windows | Your Linux files are at `\\wsl$\Ubuntu` (or `\\wsl.localhost\Ubuntu`), and appear under **Linux** in File Explorer |
| Ubuntu (WSL) | `explorer.exe .` opens the current Linux folder in File Explorer |

**Worked example.** You are in `C:\Users\ada\projects`, and you want `C:\Users\ada\projects\habit\README.md`.

- Absolute path: `C:\Users\ada\projects\habit\README.md`
- Relative path: `habit\README.md`
- From inside `habit\src`, the relative path is `..\README.md`. The `..` means "the parent folder."

In WSL, the same file is `/home/ada/projects/habit/README.md`, or `habit/README.md` relative to `~/projects`. The `~` symbol means your home directory.

**Mistakes.** Spaces break paths unless you quote them: `cd "My Projects"`. Working on files under `/mnt/c` from Linux tools is much slower, so Microsoft's WSL documentation recommends keeping Linux projects in `~`. A third trap is code that works on Windows but fails on Linux because the file is `Config.json` and the code opens `config.json`. Most servers, containers and CI runners are Linux, so code must survive a case-sensitive, forward-slash world before it ships.

---

## 4. PATH: how the shell finds programs

**Intuition.** When you call a friend, your phone does not search the whole world. It checks your contacts in order. PATH is the shell's contact list for programs.

**Definition.** `PATH` is an environment variable that holds a list of folders. When you type a command name without a folder, the shell checks each folder in PATH, in order, and runs the first match. On Windows the list is separated by semicolons, and the shell also tries endings such as `.exe` and `.cmd`. On Linux the list is separated by colons. The current folder is not on PATH, which is why you run your own scripts as `./script.sh` on Day 5. If no folder has the program, you get `'x' is not recognized...` in PowerShell or `x: command not found` in bash.

**Worked example.** In PowerShell:

```powershell
$env:Path -split ';'      # list each folder on PATH, one per line
Get-Command git           # which file runs when I type "git"?
where.exe python          # every python.exe on PATH, in search order
```

In Ubuntu:

```bash
echo $PATH                # folders separated by colons
which python3             # the first match on PATH
type cd                   # "cd is a shell builtin": no file at all
```

Suppose `where.exe python` prints this:

```
C:\Users\ada\AppData\Local\Microsoft\WindowsApps\python.exe
C:\Users\ada\AppData\Local\Programs\Python\Python314\python.exe
```

Typing `python` runs the first one. On many Windows 11 machines, that first entry is an **App Execution Alias** that opens the Microsoft Store instead of running Python. Turn off the `python.exe` and `python3.exe` aliases in **Settings > Apps > Advanced app settings > App execution aliases**, or use the Python launcher `py`, which the python.org installer adds.

**Mistakes.** PATH changes reach only new processes. After you install something, close and reopen the terminal and VS Code. This one fact fixes many "installed but not found" problems. Do not edit PATH with the old `setx` command. It silently truncates values longer than 1024 characters, and it can wipe your PATH. Use **Start > Edit environment variables for your account** instead. Order matters too. With two Python versions on PATH, the earlier folder wins. Tools such as pyenv, nvm and Docker images work by controlling PATH, and "the CI server picked the wrong version" is usually a PATH problem.

---

## 5. Environment variables

**Intuition.** Environment variables are sticky notes attached to every process: "your home folder is here," "the language is English," "the database address is this." A process reads its notes and hands copies to any child process it starts.

**Definition.** An environment variable is a name and text value that the OS keeps for each process. When a process starts another, the child gets a **copy** of the parent's variables. Changing the copy never changes the parent. So a variable set in one terminal tab does not appear in another. Programs read variables to get configuration without hard-coding it. Common examples are `PATH`, `HOME` (Linux) or `USERPROFILE` (Windows), `TEMP` and `LANG`.

**Worked example.** This program reads a variable:

```python
import os
greeting = os.environ.get("APP_GREETING", "Hello")   # "Hello" if not set
print(greeting)
```

Set the variable for one terminal session in PowerShell:

```powershell
$env:APP_GREETING = "Good morning"
py greet.py                      # prints: Good morning
Get-ChildItem Env:               # list every variable
```

Or in bash:

```bash
export APP_GREETING="Good morning"
python3 greet.py                 # prints: Good morning
APP_GREETING="Hi" python3 greet.py   # set it for this one command only
env                              # list every variable
```

To make a variable permanent, use the "Edit environment variables for your account" window on Windows. In bash, add the `export` line to `~/.bashrc` and run `source ~/.bashrc`.

**Mistakes.** In bash, `APP_GREETING = "Hi"` fails. There must be no spaces around `=`. The bigger mistake is putting secrets (API keys, passwords) in code, or committing a `.env` file to GitHub. Bots scan public repos for leaked keys within minutes. Keep secrets in environment variables or in a `.env` file listed in `.gitignore`. Environment variables are not encrypted, and any process running as you can read them. Production systems use a secrets manager. The Twelve-Factor App method recommends storing configuration in the environment, and platforms such as Vercel, Heroku and AWS all offer an environment variables page for it.

---

## 6. Package managers and virtual environments

**Intuition.** A package manager is an app store for developers, which you drive from the terminal. It knows where to download software, which version you have, and what else that software needs.

**Definition.** A package manager installs, upgrades and removes **packages**, and resolves their **dependencies** (other packages a package needs). You will use four of them at different levels.

| Level | Tool | Example | Notes |
|---|---|---|---|
| Windows apps | **winget** (built into Windows 11) | `winget install --id Git.Git -e` | Installs programs system-wide or per user |
| Linux system | **apt** (Ubuntu) | `sudo apt install tree` | Needs `sudo` because it changes the system |
| Python libraries | **pip** inside a **venv** | `pip install requests` | Per project, recorded in `requirements.txt` |
| JavaScript libraries | **npm** (comes with Node.js, Day 70) | `npm install express` | Per project, recorded in `package.json` |

**Worked example.** Create a Python environment that belongs to one project:

```powershell
mkdir $HOME\dev\day03; cd $HOME\dev\day03
py -m venv .venv                 # create a private package folder
.\.venv\Scripts\Activate.ps1     # activate it: the prompt shows (.venv)
pip install requests             # goes into .venv only
pip freeze | Out-File -Encoding utf8 requirements.txt   # record exact versions
deactivate
```

Plain `pip freeze > requirements.txt` in Windows PowerShell 5.1 writes UTF-16 text, which GitHub displays poorly. `Out-File -Encoding utf8` writes UTF-8.

If activation fails with "running scripts is disabled on this system," run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once and try again. In Ubuntu, the same steps are `python3 -m venv .venv` and `source .venv/bin/activate`. If Ubuntu says venv is unavailable, run `sudo apt install python3-venv` first.

**Mistakes.** Installing Python packages globally mixes versions between projects until something breaks. Recent Ubuntu releases block `pip install` outside a virtual environment and show an `externally-managed-environment` error. That error protects you, so use a venv instead of forcing the install. Package managers also run code from strangers. Misspelled package names (typosquatting) have carried malware on PyPI and npm, so check a name before you install it. Every codebase has a dependency file (`requirements.txt`, `pyproject.toml`, `package.json`), and GitHub's Dependabot uses those files to flag vulnerable versions.

---

## 7. The editor and the debugger

### 7.1 VS Code

**Definition.** Visual Studio Code is a free editor from Microsoft. Its source is MIT-licensed (the "Code - OSS" project). The downloadable build adds a Microsoft license and telemetry. VSCodium is a community build without them. Open a **folder** as a workspace. Opening a single file turns off project-wide search and debug configuration. Extensions add languages and tools. Install **Python** (which brings the Python debugger) and **WSL** (both by Microsoft) now.

| Action | Shortcut |
|---|---|
| Command Palette (find any command) | Ctrl+Shift+P |
| Quick Open a file by name | Ctrl+P |
| Toggle the integrated terminal | Ctrl+\` |
| Settings | Ctrl+, |
| Go to definition | F12 |
| Rename symbol everywhere | F2 |
| Format document | Shift+Alt+F |

**Worked example.** Connect VS Code to Linux. In your Ubuntu terminal:

```bash
mkdir -p ~/dev/day03-linux
cd ~/dev/day03-linux
code .
```

The first time, VS Code installs a small server inside Ubuntu, then opens a window labeled **WSL: Ubuntu** in the bottom-left corner. The editor UI runs on Windows. The terminal, the files and the Python interpreter run in Linux. This works because WSL adds your Windows PATH to Linux's PATH by default, so `code` is found.

**Mistakes.** The classic beginner bug is forgetting to save. A dot on an editor tab means unsaved changes. Turn on **File > Auto Save** to remove the problem. Installing dozens of extensions slows the editor and causes conflicts, so keep the list short.

### 7.2 Debugging

**Intuition.** Guessing at a bug is like finding a gas leak by sniffing each room. A debugger is a meter you can hold up to any pipe while the gas flows.

**Definition.** A **debugger** runs your program under control. You set a **breakpoint** on a line. When execution reaches it, the process pauses (the CPU stops running that thread), and you can inspect variables and the **call stack** from Day 2. Then you step. **Step over** runs the current line. **Step into** enters a function call. **Step out** finishes the current function. In VS Code, **F9** toggles a breakpoint, **F5** starts or continues, **F10** steps over, **F11** steps into, **Shift+F11** steps out, and **Shift+F5** stops. Two other tools are useful too. Python's `breakpoint()` drops into the `pdb` debugger in the terminal. A **traceback** names the error, and you read it from the last line up.

**Worked example.** The `average.py` file in section 9 has a bug. It prints `56.666666666666664` for `[80, 90, 100]`, but the answer should be 90.0. Put a breakpoint on line 7, `total += numbers[i]`, and press F5. Watch `i` in the **Variables** pane. It takes the values 0 and 1, never 2. The loop skips the last number, so `total` ends at 170, and 170 / 3 = 56.67. The fix is `range(len(numbers))`, or better, `sum(numbers) / len(numbers)`.

The same hunt in the terminal with `pdb`, run from the project folder:

```
$ python3 -m pdb average.py
(Pdb) b 7          # breakpoint on line 7
(Pdb) c            # continue to it
(Pdb) p i, total   # print values
(Pdb) n            # next line
(Pdb) q            # quit
```

**Mistakes.** Changing code at random until the error disappears hides the bug instead of fixing it. Form a guess, then use the debugger to confirm or reject it. Debuggers also change timing. A race between threads or network calls can vanish when you pause, and logs with timestamps are the better tool there. Remove leftover `breakpoint()` calls before you commit, because they freeze a server that waits for input. Support and backend engineers spend much of their week reproducing bugs from tickets, and "walk me through how you would debug this" is a standard interview question.

---

## 8. Hands-on exercises (about 60 minutes)

Install commands are in section 2. Complete those first.

1. **Check the install (5 minutes).** In a new PowerShell window, run `git --version`, `py --version` and `code --version`. Run `wsl -l -v`. *Check:* all three print versions and `wsl -l -v` shows VERSION 2.
2. **Terminal (5 minutes).** Open a PowerShell tab and an Ubuntu tab. In each, print where you are (`Get-Location` and `pwd`). *Check:* one shows `C:\Users\...`, the other `/home/...`.
3. **Filesystem (10 minutes).** In Ubuntu, run `cd /mnt/c/Users` and `ls`, then `cd ~` and `explorer.exe .`. *Check:* File Explorer opens on your Linux home, and you can say why `/mnt/c` and `~` are different trees.
4. **PATH (10 minutes).** Run `where.exe python` and `where.exe py` in PowerShell, then `which python3` and `which code` in Ubuntu. *Check:* you can say which file runs for each name, and whether the App Execution Alias appears first.
5. **Environment variables (5 minutes).** In PowerShell, run `$env:APP_GREETING = "Hi"`. Open a second tab and run `$env:APP_GREETING`. *Check:* the second tab is empty, and you can explain why, using Day 2's idea of separate processes.
6. **Packages (5 minutes).** In a venv, install `requests` and run `pip freeze`. *Check:* `requests` and its dependency `urllib3` are listed.
7. **Bytecode (5 minutes).** Save the `dis` example from Day 2 as `bytecode.py` and run `py bytecode.py`. *Check:* you see `LOAD_FAST`-style instructions.

---

## 9. Project: prove your environment works

**Goal.** A repo that shows a reviewer, in two minutes, that your machine is set up correctly and that you can find a bug with a debugger.

**Repo name:** `100-days-software-day-03-developer-environment`

**Repository layout:**

```
100-days-software-day-03-developer-environment/
├── README.md
├── SETUP.md                    # what you installed, versions, problems and fixes
├── check-env.ps1               # Windows environment check
├── check-env.sh                # WSL (Ubuntu) environment check
├── results/
│   ├── check-env-windows.txt   # output of check-env.ps1
│   └── check-env-wsl.txt       # output of check-env.sh
├── average.py                  # the buggy program, then the fix
├── .vscode/
│   └── launch.json             # debug configuration
├── requirements.txt            # from pip freeze
├── screenshots/
│   └── debugger-paused.png     # VS Code stopped on a breakpoint
└── .gitignore
```

**Starter code.**

`check-env.ps1`:

```powershell
# check-env.ps1: report which developer tools this machine can find, and where.
# Run from PowerShell: .\check-env.ps1

$tools = @("code", "git", "python", "py", "wsl", "winget")

Write-Output "Tool locations"
Write-Output "--------------"
foreach ($tool in $tools) {
    $cmd = Get-Command $tool -ErrorAction SilentlyContinue | Select-Object -First 1
    if ($cmd) {
        Write-Output ("{0,-8} {1}" -f $tool, $cmd.Source)
    } else {
        Write-Output ("{0,-8} NOT FOUND on PATH" -f $tool)
    }
}

Write-Output ""
Write-Output "Versions"
Write-Output "--------"
if (Get-Command git -ErrorAction SilentlyContinue) { git --version }
if (Get-Command py -ErrorAction SilentlyContinue) { py --version }
if (Get-Command code -ErrorAction SilentlyContinue) { code --version | Select-Object -First 1 }

Write-Output ""
Write-Output "PATH entries (searched in this order)"
Write-Output "-------------------------------------"
$env:Path -split ';' | Where-Object { $_ -ne "" } | ForEach-Object { Write-Output "  $_" }
```

`check-env.sh` (create it inside Ubuntu, in `~/dev`, with `code check-env.sh`):

```bash
#!/usr/bin/env bash
# check-env.sh: report which tools WSL can find, and where.

for tool in git python3 code; do
  if command -v "$tool" > /dev/null 2>&1; then
    printf '%-8s %s\n' "$tool" "$(command -v "$tool")"
  else
    printf '%-8s NOT FOUND on PATH\n' "$tool"
  fi
done

echo
echo "Kernel: $(uname -r)"
echo
echo "PATH entries (searched in this order):"
echo "$PATH" | tr ':' '\n'
```

`average.py`:

```python
import os


def average(numbers):
    total = 0
    for i in range(len(numbers) - 1):
        total += numbers[i]
    return total / len(numbers)


if __name__ == "__main__":
    greeting = os.environ.get("APP_GREETING", "Hello")
    scores = [80, 90, 100]
    print(f"{greeting}! Average score: {average(scores)}")
```

`.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug average.py",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/average.py",
            "console": "integratedTerminal",
            "env": { "APP_GREETING": "Debugging" }
        }
    ]
}
```

`.gitignore`:

```
.venv/
__pycache__/
.env
```

**Step-by-step tasks:**

1. Create `C:\Users\<you>\dev\100-days-software-day-03-developer-environment` and open it in VS Code with **File > Open Folder**.
2. Add `check-env.ps1` and run it with `.\check-env.ps1`. If PowerShell refuses to run scripts, run `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once. Then save the output: `mkdir results`, then `.\check-env.ps1 | Out-File -Encoding utf8 results\check-env-windows.txt`.
3. In Ubuntu, create `check-env.sh` in `~/dev`, run `chmod +x check-env.sh` (Day 5 explains permissions), then run `./check-env.sh > check-env-wsl.txt` (Day 5 explains the `>`). Run `explorer.exe .`, copy `check-env.sh` into the repo folder, and copy `check-env-wsl.txt` into its `results\` folder.
4. Add `average.py` and `launch.json`. Run the file once normally and note the wrong output. Set a breakpoint on line 7, press F5 with the "Debug average.py" configuration, step through the loop, and take `debugger-paused.png` with the **Variables** and **Call Stack** panes visible.
5. Fix the bug. Run again and confirm that `Debugging! Average score: 90.0` appears in the debug terminal. That greeting proves the `env` setting in `launch.json` reached the process.
6. Create a venv, install `requests`, and save `requirements.txt` as shown in section 6.
7. Write `SETUP.md`: every install command you ran, the version each tool reported, and every error you hit with how you fixed it. You will reuse it on your next laptop.
8. Create the GitHub repo and upload the files with **Add file > Upload files**. Do not upload `.venv`.

**README checklist:**

- [ ] What the repo proves, in two sentences
- [ ] A table of tools and versions (VS Code, Git, Python, WSL, Ubuntu)
- [ ] How to run `check-env.ps1` and `check-env.sh`
- [ ] The bug in `average.py`: the symptom, how the debugger revealed it, and the fix
- [ ] One PATH or environment-variable problem you hit and solved (or a note that you hit none)

**Done when:**

- Both `results/` files are committed and show every tool found.
- `wsl -l -v` output in `SETUP.md` shows Ubuntu on VERSION 2.
- The screenshot shows VS Code paused on a breakpoint with variables visible.
- `average.py` in the repo is the fixed version, and the README describes the original bug.
- No `.venv` folder and no secrets are in the repo.

---

## 10. Check your understanding

1. You install Node.js, then type `node --version` in a PowerShell window that was already open, and get "not recognized." What happened?
2. You set `$env:API_KEY` in one terminal tab, then run your app from a VS Code terminal opened earlier. The app cannot see the key. Why?
3. Two folders on PATH each contain a `python.exe`. How does the shell choose, and how can you see which one it picked?
4. Why would a team keep a `requirements.txt` instead of saying "install whatever packages the code needs"?
5. A teammate's Python project runs on Windows but fails on the Linux build server with "file not found" for `Data/Users.csv`. The file is in the repo as `data/users.csv`. What is the cause?
6. You run Linux tools on a project stored at `/mnt/c/Users/ada/project`, and everything is slow. What would you change?
7. A bug happens only when two requests arrive at almost the same moment, and it disappears when you set a breakpoint. Why, and what tool would you use instead?

**Answers (check after you try):**

1. The installer changed PATH, but a running process keeps the environment it started with. Open a new terminal and it will work.
2. Each process gets a copy of its parent's environment when it starts. The variable was set in a different process (the other tab), so the VS Code terminal never received it. Set it in the same terminal that runs the app, or in a config file the app reads.
3. The shell searches the folders in PATH in order and runs the first match. `where.exe python` (or `Get-Command python`) shows the order and the winner.
4. It pins exact versions, so every developer and server installs the same set. Without it, a new dependency version can break the code on one machine but not another.
5. Windows filesystems ignore case, so the wrong capitalization worked there. Linux is case-sensitive, so `Data/Users.csv` and `data/users.csv` are different paths. Fix the code to match the real name.
6. Move the project into the Linux filesystem, for example `~/project`. Crossing from Linux into the Windows drive is much slower.
7. Pausing changes the timing, so the race between the two requests no longer happens. Use logging with timestamps to see the real order of events.

---

## 11. Sources to read and watch

**Read:**
- [Microsoft Learn: How to install Linux on Windows with WSL](https://learn.microsoft.com/en-us/windows/wsl/install)
- [Microsoft Learn: Set up a WSL development environment](https://learn.microsoft.com/en-us/windows/wsl/setup/environment)
- [Microsoft Learn: Working across Windows and Linux file systems](https://learn.microsoft.com/en-us/windows/wsl/filesystems)
- [Microsoft Learn: Windows Package Manager (winget)](https://learn.microsoft.com/en-us/windows/package-manager/winget/)
- [Microsoft Learn: Windows Terminal](https://learn.microsoft.com/en-us/windows/terminal/)
- [VS Code documentation](https://code.visualstudio.com/docs), especially "Debugging" and "Developing in WSL" ([direct link](https://code.visualstudio.com/docs/remote/wsl))
- [Python docs: Virtual Environments and Packages](https://docs.python.org/3/tutorial/venv.html)
- [Python docs: pdb, The Python Debugger](https://docs.python.org/3/library/pdb.html)
- [Wikipedia: PATH (variable)](https://en.wikipedia.org/wiki/PATH_(variable))
- [Wikipedia: Environment variable](https://en.wikipedia.org/wiki/Environment_variable)
- [Wikipedia: Package manager](https://en.wikipedia.org/wiki/Package_manager)
- [Wikipedia: Windows Subsystem for Linux](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux)
- [The Twelve-Factor App: III. Config](https://12factor.net/config)

**Tutorials:**
- [VS Code: Getting Started with Python in VS Code](https://code.visualstudio.com/docs/python/python-tutorial), which walks through running and debugging a Python file.
- GeeksforGeeks: search "Environment Variables in Linux/Unix" and "How to Install WSL2 on Windows".
- [MIT Missing Semester: Debugging and Profiling](https://missing.csail.mit.edu/2020/debugging-profiling/) (the debugging half).

**Watch:**
- The official Visual Studio Code YouTube channel: search "Getting started with VS Code" and "Python debugging in VS Code".
- MIT Missing Semester Lecture 1, "Course Overview + The Shell", on YouTube.

**Practice:**
- [MIT Missing Semester: Course overview + the shell](https://missing.csail.mit.edu/2020/course-shell/). Do the exercises at the bottom in your new Ubuntu terminal.

**Note on links:** open each one once. If a page has moved, search its title.

---

## 12. Tomorrow

Day 4, **Linux CLI I**, moves into the Ubuntu terminal you set up today. You will move around and manage files with `pwd`, `ls`, `cd`, `mkdir`, `touch`, `cp`, `mv` and `rm`. You will read files with `cat` and `less`, look up any command with `man`, and tour the Linux filesystem from `/` down to your home folder. The filesystem tree, PATH and WSL's two worlds from today are the map. Tomorrow you learn to walk it quickly and safely.
