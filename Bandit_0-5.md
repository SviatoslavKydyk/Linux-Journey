# OverTheWire Bandit Write-Up: Levels 0 to 5

Technical write-up covering command-line syntax, file inspection, search operations, and syntax troubleshooting during OverTheWire Bandit levels 0 through 5.

> **Note:** Passwords are intentionally redacted. OverTheWire asks players not to publish passwords or spoilers, so this document focuses on methodology and the concepts each level teaches.

---

## Environment

| Item | Detail |
|------|--------|
| Client | Windows 10/11, Command Prompt with the built-in OpenSSH client |
| Server | `bandit.labs.overthewire.org` |
| Port | `2220` |
| Authentication | Password obtained from the previous level |

Connection template:

```bash
ssh bandit<N>@bandit.labs.overthewire.org -p 2220
```

On the first connection, SSH prompts to verify the server's ED25519 host key fingerprint. Accepting it stores the key in `known_hosts`, and later connections verify against it.

---

## Summary

| Level | Objective | Core Concept | Key Command(s) |
|-------|-----------|--------------|----------------|
| 0 | Log in and read a file | SSH on a non-standard port | `ssh`, `cat` |
| 1 | Read a file named `-` | Filenames parsed as options | `cat ./-` |
| 2 | Read a file with spaces and leading dashes | Quoting and path prefixes | `cat ./"--spaces in this filename--"` |
| 3 | Find a hidden file | Dotfiles and hidden entries | `ls -la`, `ls -A`, `find` |
| 4 | Identify the human-readable file | File type detection | `file ./*` |
| 5 | Find a file by size, readable, non-executable | Filtering with `find` | `find . -type f -size 1033c` |

---

## Level 0: Establishing the SSH Connection

**Objective:** Log in to the game server and retrieve the password from a file in the home directory.

**Approach:**

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
cat readme
```

**Key points:**
- SSH defaults to port 22. The `-p` flag is required for the non-standard port 2220.
- Password input is not echoed to the terminal, which is expected behavior.

**Password:** `[REDACTED]`

---

## Level 1: Filename Beginning with a Dash

**Objective:** Read a file named `-` in the home directory.

**Approach:**

```bash
cat ./-
```

**Key points:**
- A bare `-` is interpreted by most utilities as standard input rather than a filename, so `cat -` waits for input.
- Prefixing the path with `./` makes the argument an explicit relative path, which removes the ambiguity.

**Password:** `[REDACTED]`

---

## Level 2: Spaces and Leading Dashes in a Filename

**Objective:** Read a file named `--spaces in this filename--`.

**Troubleshooting log:**

| Attempt | Command | Result | Cause |
|---------|---------|--------|-------|
| 1 | `cat "spaces in this filename"` | No such file or directory | Guessed name was wrong |
| 2 | `cat /home/"spaces in this filename"` | No such file or directory | Wrong directory and wrong name |
| 3 | `ls`, then `cat "--spaces in this filename--"` | `unexpected argument` error | Leading `--` parsed as an option |
| 4 | `cat ./"--spaces in this filename--"` | Success | `./` prefix disambiguates the path |

**Working command:**

```bash
ls
cat ./"--spaces in this filename--"
```

**Key points:**
- Always run `ls` to confirm the exact filename instead of assuming it.
- Quotes handle the spaces, but they do not stop a leading dash from being read as an option.
- Two valid fixes: prefix with `./`, or use `--` to signal the end of options (`cat -- "--spaces in this filename--"`).

**Password:** `[REDACTED]`

---

## Level 3: Hidden Files

**Objective:** Find a hidden file inside the `inhere` directory.

**Approach:**

```bash
cd ~/inhere
ls -la
cat ./"...Hiding-From-You"
```

**Commands compared:**

| Command | Behavior |
|---------|----------|
| `ls` | Hides all entries beginning with `.` |
| `ls -a` | Shows all entries, including `.` and `..` |
| `ls -A` | Shows all entries except `.` and `..` |
| `ls -la` | Long format with permissions, owner, and size, including hidden entries |
| `find` | Lists everything recursively, hidden files included |

**Key points:**
- On Linux, a leading dot marks a file as hidden. It is a display convention, not a security control.
- `cd /inhere` failed because the leading `/` refers to the filesystem root. The directory lives at `~/inhere`, illustrating absolute versus relative paths.

**Password:** `[REDACTED]`

---

## Level 4: Identifying the Human-Readable File

**Objective:** Among ten files, find the one containing human-readable text.

**Troubleshooting:**

```bash
file "-file00"
# file: Cannot open `ile00' (No such file or directory)
```

The leading `-f` was consumed as an option flag, leaving `ile00` as the filename. Using `./` fixes it, and a glob checks every file at once.

**Working approach:**

```bash
cd ~/inhere
file ./*
cat ./-file07
```

**Key points:**
- `file` inspects file contents (magic bytes), not extensions. Binary content is reported as `data`, while text is reported as `ASCII text`.
- `./*` expands to every file in the directory with the `./` prefix already applied, which avoids the dash problem entirely.
- Checking files one at a time works but does not scale. The glob is faster and less error-prone.

**Password:** `[REDACTED]`

---

## Level 5: Searching by File Properties

**Objective:** In a directory tree of 20 subdirectories, find the single file that is human-readable, exactly 1033 bytes, and not executable.

**Troubleshooting log:**

| Command | Outcome |
|---------|---------|
| `file ./*` | Only showed that all 20 entries are directories |
| `di` | Typo; command not found |
| `du` | Reported directory sizes, not useful for locating a file |
| `find` | Listed 200 entries, too many to inspect manually |
| `find . -type f -size 1033c` | Narrowed the result to one file |

**Working command:**

```bash
find . -type f -size 1033c
cat ./maybehere07/.file2
```

**Recommended command covering all three criteria:**

```bash
find . -type f -size 1033c ! -executable
```

**Key points:**
- `-type f` limits results to regular files.
- `-size 1033c` matches exact size in bytes. The `c` suffix means bytes, whereas `k` means kilobytes and the default unit is 512-byte blocks.
- `! -executable` excludes files with the execute permission.
- The target file was hidden (leading dot), which is why simple `ls` inspection of each directory would have missed it. `find` includes hidden files by default.

**Password:** `[REDACTED]`

---

## Key Takeaways

1. **Quoting and paths are separate problems.** Quotes protect spaces from the shell. A `./` prefix (or `--`) protects leading dashes from the command's option parser.
2. **Verify names before using them.** `ls` (and `ls -la` for hidden entries) removes guesswork.
3. **Use content-based inspection.** `file` identifies types by content, which is more reliable than extensions.
4. **Filter instead of browsing.** `find` with `-type`, `-size`, and `-executable` replaces manual inspection across many directories.
5. **Read error messages carefully.** Messages such as `Cannot open 'ile00'` and `unexpected argument` point directly at how the command was parsed.

---

## Commands Reference

| Command | Purpose |
|---------|---------|
| `ssh user@host -p PORT` | Connect over SSH on a custom port |
| `ls -la` / `ls -A` | List files including hidden entries |
| `cat ./file` | Print file contents using an explicit relative path |
| `file ./*` | Identify the type of every file in a directory |
| `find . -type f -size Nc` | Locate files of an exact byte size |
| `find . ! -executable` | Exclude executable files |
| `du` | Report disk usage per directory |
