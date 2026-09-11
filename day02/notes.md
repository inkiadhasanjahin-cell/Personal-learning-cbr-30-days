# CLI Toolkit — grep, find, sed, awk, file, strings, xxd, base64, chmod

A working reference for the everyday Unix toolkit: searching text, searching the filesystem, editing streams, processing columns, and inspecting files at the byte level. Each section has the core syntax, the flags actually worth memorizing, worked examples against one shared dataset, and practice problems with hidden solutions — try each one yourself before opening it (solutions are in collapsible `<details>` blocks; they expand in GitHub and most Markdown viewers, and read fine as plain text otherwise).

**Contents:** [Setup](#setup) · [grep](#grep) · [find](#find) · [sed](#sed) · [awk](#awk) · [file](#file) · [strings](#strings) · [xxd](#xxd) · [base64](#base64) · [chmod & permissions](#chmod--permissions) · [Final challenge](#final-challenge--combine-everything)

---

## Setup — shared practice data

Every example and practice problem below reuses these two files. Create them once in a scratch folder and work through the sections in order.

```bash
mkdir ~/cli-practice && cd ~/cli-practice
```

Create `server.log`:

```bash
cat > server.log << 'EOF'
2024-01-15 10:23:01 ERROR user=alice ip=192.168.1.10 msg="login failed"
2024-01-15 10:23:45 INFO  user=bob   ip=192.168.1.12 msg="login success"
2024-01-15 10:24:02 ERROR user=alice ip=192.168.1.10 msg="login failed"
2024-01-15 10:25:10 WARN  user=carol ip=10.0.0.5     msg="disk space low"
2024-01-15 10:26:33 INFO  user=dave  ip=192.168.1.15 msg="file uploaded"
2024-01-15 10:27:59 ERROR user=eve   ip=10.0.0.9     msg="permission denied"
2024-01-15 10:29:14 INFO  user=bob   ip=192.168.1.12 msg="logout"
EOF
```

Create `staff.csv`:

```bash
cat > staff.csv << 'EOF'
name,dept,salary
Alice,Engineering,95000
Bob,Sales,62000
Carol,Engineering,88000
Dave,Marketing,71000
Eve,Sales,67000
EOF
```

Keep a terminal open on this folder as you go — reading a command and running it are very different skills, and this toolkit only sticks if your fingers learn it too.

---

## grep

Searches text for lines matching a pattern. The single most-used filter in the whole toolkit — most pipelines end or start with it.

### Syntax

```bash
grep [OPTIONS] PATTERN [FILE...]
```

With no file, `grep` reads from standard input, so it chains naturally after a pipe: `ps aux | grep nginx`.

### Key flags

| Flag | Meaning |
|---|---|
| `-i` | ignore case |
| `-v` | invert match — show lines that *don't* match |
| `-n` | show line numbers |
| `-c` | print a count of matching lines instead of the lines |
| `-l` / `-L` | list filenames that match / don't match, not the lines themselves |
| `-w` | match whole words only |
| `-o` | print only the matched part of the line, not the whole line |
| `-r` / `-R` | recurse into directories |
| `-E` | extended regex — lets you use `+ ? \| ()` without backslashes |
| `-A n` / `-B n` / `-C n` | show n lines of context After / Before / around (Context) each match |

### Regex you actually need

| Token | Meaning |
|---|---|
| `.` | any single character |
| `*` | zero or more of the previous token |
| `^` / `$` | start / end of line |
| `[abc]` / `[^abc]` | any one of a, b, c / none of a, b, c |
| `[0-9]` | a character range |
| `+ ? \| ( )` | one-or-more, zero-or-one, alternation, grouping — need `-E` or a backslash before them in basic mode |

### Worked examples

```bash
# every ERROR line
grep ERROR server.log

# case-insensitive, with line numbers
grep -in error server.log

# lines NOT from bob
grep -v user=bob server.log

# count how many times alice appears
grep -c alice server.log

# pull out just the IP addresses
grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' server.log

# ERROR or WARN, with 1 line of context after each
grep -A1 -E 'ERROR|WARN' server.log
```

> **Gotcha:** in basic (default) grep, `+`, `?`, `|` and `()` are literal characters unless escaped with a backslash. Reach for `-E` the moment you need any of them — it saves you from a wall of backslashes.

### Practice problems

1. Find every line in `server.log` that mentions `carol`, ignoring case.
   <details><summary>Show solution</summary>

   ```bash
   grep -i carol server.log
   ```
   </details>

2. Count how many `INFO` lines exist.
   <details><summary>Show solution</summary>

   ```bash
   grep -c INFO server.log
   ```
   </details>

3. Show only lines that are *not* ERROR and *not* INFO (i.e. just WARN).
   <details><summary>Show solution</summary>

   ```bash
   grep -v -E 'ERROR|INFO' server.log
   # or simply: grep WARN server.log
   ```
   </details>

4. Extract just the usernames (the word after `user=`) from every line, one per output line.
   <details><summary>Show solution</summary>

   ```bash
   grep -Eo 'user=[a-z]+' server.log | grep -Eo '[a-z]+$'
   # cleaner with cut, once you meet it: grep -o 'user=[a-z]*' server.log | cut -d= -f2
   ```
   </details>

5. Find lines where the IP starts with `192.168`.
   <details><summary>Show solution</summary>

   ```bash
   grep -E '192\.168\.' server.log
   ```
   </details>

6. Search recursively for the word `TODO` as a whole word (not `TODOS`) in every file under the current directory, showing line numbers.
   <details><summary>Show solution</summary>

   ```bash
   grep -rnw TODO .
   ```
   </details>

---

## find

Walks a directory tree and reports files/directories matching tests you give it — name, type, size, age, permissions — then optionally acts on each match.

### Syntax

```bash
find [PATH...] [EXPRESSION]
```

Expression = tests (what to match) + actions (what to do), read left to right. No path given means search the current directory.

### Key tests & actions

| Test / action | Meaning |
|---|---|
| `-name "pat"` / `-iname` | match filename by shell glob (case-sensitive / insensitive) |
| `-type f` / `-type d` | regular files / directories only |
| `-mtime -7` | modified less than 7 days ago (`+7` = more than, `7` = exactly) |
| `-mmin -60` | same idea, in minutes |
| `-size +10M` | larger than 10MB (`-10k`, `+1G`, etc.) |
| `-perm 644` | exact permission bits |
| `-maxdepth n` | don't recurse past n directory levels |
| `-empty` | empty files/directories |
| `!` / `-not` | negate the next test |
| `-exec cmd {} \;` | run `cmd` on each match, once per file |
| `-delete` | delete each match (use after testing with `-print` first!) |

### Worked examples

```bash
# every .log file below the current directory
find . -name "*.log"

# only directories, one level deep
find . -maxdepth 1 -type d

# files bigger than 5MB anywhere under /var
find /var -type f -size +5M

# files modified in the last 2 days
find . -type f -mtime -2

# .tmp files older than 30 days -> delete them
find . -name "*.tmp" -mtime +30 -delete

# run chmod on every .sh file found
find . -name "*.sh" -exec chmod +x {} \;

# files NOT owned by the www-data group, combined with grep via a pipe
find . -type f | grep -v node_modules
```

> **Gotcha:** `-exec cmd {} \;` runs the command once per file (safe but slower). `-exec cmd {} +` batches many files into one invocation, like `xargs` — much faster for thousands of files, but the command must accept multiple filename arguments.

### Practice problems

1. Find every `.csv` file in the current directory tree.
   <details><summary>Show solution</summary>

   ```bash
   find . -name "*.csv"
   ```
   </details>

2. Find only regular files (no directories) modified in the last 1 day.
   <details><summary>Show solution</summary>

   ```bash
   find . -type f -mtime -1
   ```
   </details>

3. Find all empty files in the current directory (not subdirectories).
   <details><summary>Show solution</summary>

   ```bash
   find . -maxdepth 1 -type f -empty
   ```
   </details>

4. Find every file *except* those ending in `.log`.
   <details><summary>Show solution</summary>

   ```bash
   find . -type f ! -name "*.log"
   ```
   </details>

5. Find all `.log` files and, for each one, print a line count using `wc -l`.
   <details><summary>Show solution</summary>

   ```bash
   find . -name "*.log" -exec wc -l {} \;
   ```
   </details>

6. Find directories named exactly `tmp` anywhere below the current directory, case-insensitively.
   <details><summary>Show solution</summary>

   ```bash
   find . -type d -iname "tmp"
   ```
   </details>

---

## sed

The "stream editor" — applies an editing script to text line by line, without opening an interactive editor. Built for substitutions and line-based edits inside scripts and pipelines.

### Syntax

```bash
sed [OPTIONS] 'SCRIPT' [FILE...]
```

### The workhorse: substitution

```bash
sed 's/PATTERN/REPLACEMENT/FLAGS'
```

| Flag | Meaning |
|---|---|
| (none) | replace only the first match per line |
| `g` | replace every match on the line (global) |
| `i` | case-insensitive match |
| `p` | print the line (usually paired with `-n` to print *only* changed lines) |

### Key options & commands

| Piece | Meaning |
|---|---|
| `-n` | suppress automatic printing — output only what you explicitly `p`rint |
| `-i` | edit the file in place (`-i.bak` keeps a backup — recommended while learning) |
| `-e` | chain multiple scripts in one call |
| `N,Mp` | address range — act only on lines N through M |
| `/re/d` | delete every line matching regex `re` |
| `&` | in the replacement, means "the whole matched text" |
| `\1`, `\2` | backreferences to groups captured with `\(...\)` |

### Worked examples

```bash
# replace the first "ERROR" on each line with "ISSUE"
sed 's/ERROR/ISSUE/' server.log

# replace ALL occurrences of "192.168.1.12" (global flag)
sed 's/192.168.1.12/REDACTED/g' server.log

# delete every WARN line
sed '/WARN/d' server.log

# print only lines 2 through 4
sed -n '2,4p' server.log

# case-insensitive replace, edit file in place with a .bak backup
sed -i.bak 's/info/INFO/gi' server.log

# wrap every matched IP in brackets, using & for "the match"
sed -E 's/[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[&]/' server.log

# swap "name,dept" order using a captured group backreference
sed -E 's/^([^,]+),([^,]+)/\2,\1/' staff.csv
```

> **Gotcha:** without `-i`, sed prints the edited result to standard output and leaves the original file untouched — perfect for testing a script safely before committing to it with `-i`.

### Practice problems

1. Replace every `ERROR` with `CRITICAL` in `server.log` and print the result (don't edit the file yet).
   <details><summary>Show solution</summary>

   ```bash
   sed 's/ERROR/CRITICAL/g' server.log
   ```
   </details>

2. Delete all lines containing `bob`, printing only the remaining lines.
   <details><summary>Show solution</summary>

   ```bash
   sed '/bob/d' server.log
   ```
   </details>

3. Print only the last 2 lines of `server.log` using sed (not `tail`).
   <details><summary>Show solution</summary>

   ```bash
   sed -n '$p;$!{x;$!d};x' server.log
   # simpler and more common in practice: sed -n '6,7p' server.log (once you know the line count)
   ```
   </details>

4. In `staff.csv`, replace every comma with a tab character.
   <details><summary>Show solution</summary>

   ```bash
   sed 's/,/\t/g' staff.csv
   ```
   </details>

5. Comment out every line of `staff.csv` by prefixing it with `#`.
   <details><summary>Show solution</summary>

   ```bash
   sed 's/^/#/' staff.csv
   ```
   </details>

6. In place, change every `Engineering` to `Eng` in `staff.csv`, keeping a backup file called `staff.csv.bak`.
   <details><summary>Show solution</summary>

   ```bash
   sed -i.bak 's/Engineering/Eng/g' staff.csv
   ```
   </details>

---

## awk

A full pattern-scanning and column-processing language. Where grep finds lines and sed edits text, awk thinks in *fields* — ideal for CSV/log data and quick reports.

### Syntax

```bash
awk 'PATTERN { ACTION }' file
```

For every input line: if `PATTERN` matches (or is omitted), run `ACTION`. Either half can be omitted — a pattern alone prints matching lines; an action alone with no pattern runs on every line.

### Built-in variables

| Variable | Meaning |
|---|---|
| `$0` | the whole current line |
| `$1, $2, ...` | 1st, 2nd, ... field of the current line |
| `NF` | Number of Fields on the current line |
| `NR` | Number of the current Record (line number) |
| `FS` | input Field Separator (default: whitespace) |
| `OFS` | output field separator, used when you rebuild `$0` |

### Structure

```bash
awk -F',' '
BEGIN { runs once, before any input }
/pattern/ { runs for each matching line }
{ runs for every line if no pattern given }
END { runs once, after all input }
' file
```

### Worked examples

```bash
# print the 3rd column (whitespace-separated) of every log line
awk '{print $3}' server.log

# -F sets the field separator: print names from the CSV
awk -F',' '{print $1}' staff.csv

# skip the header line by checking NR
awk -F',' 'NR > 1 {print $1, $3}' staff.csv

# only rows where salary (field 3) exceeds 70000
awk -F',' 'NR > 1 && $3 > 70000 {print $1}' staff.csv

# sum a column, using BEGIN/END and an accumulator
awk -F',' 'NR > 1 {sum += $3} END {print "total:", sum}' staff.csv

# reformat: name is uppercase, custom output separator
awk -F',' 'NR > 1 {print toupper($1), $2}' OFS=' - ' staff.csv

# combine with a pattern: only ERROR lines, print time + user
awk '/ERROR/ {print $2, $4}' server.log
```

> **Gotcha:** `-F` only changes how *input* is split into fields. To change the separator awk uses when it reprints `$0` or joins fields with commas in `print`, set `OFS` as well.

### Practice problems

1. Print just the log level (3rd column) for every line in `server.log`.
   <details><summary>Show solution</summary>

   ```bash
   awk '{print $3}' server.log
   ```
   </details>

2. Print the department and salary columns from `staff.csv`, skipping the header.
   <details><summary>Show solution</summary>

   ```bash
   awk -F',' 'NR > 1 {print $2, $3}' staff.csv
   ```
   </details>

3. Compute the average salary in `staff.csv`.
   <details><summary>Show solution</summary>

   ```bash
   awk -F',' 'NR > 1 {sum += $3; n++} END {print sum / n}' staff.csv
   ```
   </details>

4. Print only the names of people in the `Sales` department.
   <details><summary>Show solution</summary>

   ```bash
   awk -F',' -v dept="Sales" 'NR > 1 && $2 == dept {print $1}' staff.csv
   ```
   </details>

5. Print every line of `server.log` along with its line number, using `NR` (don't use `grep -n`).
   <details><summary>Show solution</summary>

   ```bash
   awk '{print NR": "$0}' server.log
   ```
   </details>

6. Count how many lines in `server.log` have more than 8 fields.
   <details><summary>Show solution</summary>

   ```bash
   awk 'NF > 8 {count++} END {print count+0}' server.log
   ```
   </details>

---

## file

Identifies what a file actually is by inspecting its content (magic bytes), not by trusting its extension. First move when you meet an unfamiliar or extension-less file.

### Syntax

```bash
file [OPTIONS] FILE...
```

### Key flags

| Flag | Meaning |
|---|---|
| `-i` | output an official MIME type instead of a human description |
| `-z` | look inside compressed files and describe the contents |
| `-b` | brief mode — omit the filename from the output |

### Worked examples

```bash
file server.log
# server.log: ASCII text

file /bin/ls
# /bin/ls: ELF 64-bit LSB pie executable, x86-64, ...

file -i staff.csv
# staff.csv: text/csv; charset=us-ascii

# rename an extensionless download to match its real type
file mystery_download
```

> **Why it matters:** a file called `photo.jpg` that is secretly a shell script, or an "executable" that's really just plain text, both show up correctly under `file` because it reads the actual bytes — this is exactly how magic-byte spoofing gets caught.

### Practice problems

1. Determine the true type of `/bin/bash` on your system.
   <details><summary>Show solution</summary>

   ```bash
   file /bin/bash
   ```
   </details>

2. Get the MIME type of `staff.csv`.
   <details><summary>Show solution</summary>

   ```bash
   file -i staff.csv
   ```
   </details>

3. Rename `server.log` to `mystery` (no extension) and confirm `file` still correctly reports it as text.
   <details><summary>Show solution</summary>

   ```bash
   cp server.log mystery
   file mystery
   # mystery: ASCII text
   ```
   </details>

4. Run `file` on every item in `/usr/bin` and count how many are described as `ELF` binaries.
   <details><summary>Show solution</summary>

   ```bash
   file /usr/bin/* | grep -c ELF
   ```
   </details>

---

## strings

Scans a (usually binary) file and prints every run of printable characters above a minimum length. Your first tool for peeking inside a binary without a disassembler.

### Syntax

```bash
strings [OPTIONS] FILE
```

### Key flags

| Flag | Meaning |
|---|---|
| `-n LEN` | only show runs at least `LEN` characters long (default 4) |
| `-t x` | prefix each string with its byte offset, in hex |
| `-e ENC` | character encoding to scan for (e.g. `s` = single-byte, `l` = 16-bit little-endian for UTF-16) |

### Worked examples

```bash
# dump all readable text from a binary
strings /bin/ls

# only strings of 8+ characters — cuts a lot of noise
strings -n 8 /bin/ls

# find a suspicious URL or path embedded in a binary
strings suspicious.bin | grep -E 'https?://'

# show where each string sits in the file
strings -t x /bin/ls | head
```

> **Common use:** hunting for embedded config, hardcoded credentials, version strings, or URLs inside a compiled program or a firmware image — `strings file | grep something` is one of the most common one-two combos in binary triage and CTF challenges.

### Practice problems

1. Extract every printable string from `/bin/ls` that is 10 characters or longer.
   <details><summary>Show solution</summary>

   ```bash
   strings -n 10 /bin/ls
   ```
   </details>

2. Search a binary (e.g. `/bin/ls`) for any string that looks like a file path (contains a `/`).
   <details><summary>Show solution</summary>

   ```bash
   strings /bin/ls | grep /
   ```
   </details>

3. Create a file containing hidden text inside binary padding, then use `strings` to recover just the hidden message.
   <details><summary>Show solution</summary>

   ```bash
   printf '\x00\x01\x02SECRET-FLAG-42\x03\x04' > hidden.bin
   strings hidden.bin
   ```
   </details>

4. Count how many distinct strings of length 6+ appear in `/bin/ls`.
   <details><summary>Show solution</summary>

   ```bash
   strings -n 6 /bin/ls | sort -u | wc -l
   ```
   </details>

---

## xxd

Prints (or builds) a hex dump — the raw bytes of a file in hexadecimal, alongside their printable ASCII representation. Where `strings` shows you the text, `xxd` shows you everything, including the bytes strings would skip.

### Syntax

```bash
xxd [OPTIONS] FILE
```

### Key flags

| Flag | Meaning |
|---|---|
| `-l N` | only dump the first N bytes |
| `-s N` | seek — start the dump at byte offset N |
| `-p` | plain/postscript style — continuous hex, no offsets or ASCII column |
| `-r` | reverse — turn a hex dump (or `-p` hex string) back into binary |
| `-i` | output as a C-style byte array, ready to paste into source code |

### Reading the output

```
00000000: 4865 6c6c 6f2c 2077 6f72 6c64 21 0a     Hello, world!.
```

Left to right: the byte offset, the bytes in hex (2 per byte, grouped in pairs), then the same bytes shown as ASCII where printable (a dot stands in for anything non-printable).

### Worked examples

```bash
# full hex dump of a small file
xxd server.log | head

# just the first 32 bytes
xxd -l 32 server.log

# start reading at byte 100
xxd -s 100 server.log

# get a continuous hex string, no formatting — handy for scripting
echo -n "hi" | xxd -p
# 6869

# reverse: turn that hex string back into bytes
echo -n "6869" | xxd -r -p
# hi
```

> **Common use:** confirming a file's exact magic bytes (e.g. a PNG starts with `89 50 4e 47`), diagnosing invisible/control characters that `cat` won't show you, or patching a few bytes of a binary and writing it back with `xxd -r`.

### Practice problems

1. Dump the first 16 bytes of `server.log` in hex.
   <details><summary>Show solution</summary>

   ```bash
   xxd -l 16 server.log
   ```
   </details>

2. Convert the text `CLI` into its raw hex byte representation (no offsets/ASCII column).
   <details><summary>Show solution</summary>

   ```bash
   echo -n "CLI" | xxd -p
   # 434c49
   ```
   </details>

3. Take the hex string `68656c6c6f` and turn it back into readable text.
   <details><summary>Show solution</summary>

   ```bash
   echo -n "68656c6c6f" | xxd -r -p
   # hello
   ```
   </details>

4. Check whether a file called `maybe.png` is really a PNG by inspecting its first 8 bytes.
   <details><summary>Show solution</summary>

   ```bash
   xxd -l 8 maybe.png
   # a real PNG starts: 8950 4e47 0d0a 1a0a
   ```
   </details>

5. Dump 8 bytes of `server.log` starting from offset 20.
   <details><summary>Show solution</summary>

   ```bash
   xxd -s 20 -l 8 server.log
   ```
   </details>

---

## base64

Encodes arbitrary bytes into a safe 64-character text alphabet (and decodes back). Not encryption — just a reversible, text-safe representation for binary data that has to travel through text-only channels.

### Syntax

```bash
base64 [OPTIONS] [FILE]
base64 -d [OPTIONS] [FILE]    # decode
```

### Key flags

| Flag | Meaning |
|---|---|
| `-d` / `--decode` | decode base64 back to raw bytes |
| `-w N` | wrap encoded output at N characters (`-w 0` = one unbroken line) |

### Worked examples

```bash
# encode a string
echo -n "hello world" | base64
# aGVsbG8gd29ybGQ=

# decode it back
echo -n "aGVsbG8gd29ybGQ=" | base64 -d
# hello world

# encode a whole file, single unbroken line
base64 -w 0 server.log > server.log.b64

# decode a file back to its original bytes
base64 -d server.log.b64 > server.log.restored

# round trip check — should print nothing if identical
diff server.log server.log.restored
```

> **Gotcha:** base64 output is roughly 33% larger than the input and is trivially reversible by anyone — never treat it as a security measure, only as a transport-safe encoding (embedding a binary in JSON, an email attachment, a data URL, etc).

### Practice problems

1. Base64-encode the string `Unix Toolkit`.
   <details><summary>Show solution</summary>

   ```bash
   echo -n "Unix Toolkit" | base64
   # VW5peCBUb29sa2l0
   ```
   </details>

2. Decode `Q0xJIHRvb2xraXQ=` back to text.
   <details><summary>Show solution</summary>

   ```bash
   echo -n "Q0xJIHRvb2xraXQ=" | base64 -d
   # CLI toolkit
   ```
   </details>

3. Encode `staff.csv` to a file called `staff.b64`, then decode it back and confirm the result is byte-for-byte identical to the original with `diff`.
   <details><summary>Show solution</summary>

   ```bash
   base64 staff.csv > staff.b64
   base64 -d staff.b64 > staff.restored.csv
   diff staff.csv staff.restored.csv
   ```
   </details>

4. Chain `xxd` and `base64`: get the raw hex bytes of the text `go` using `xxd -p`, then separately get its base64 form, and confirm they decode to the same original text.
   <details><summary>Show solution</summary>

   ```bash
   echo -n "go" | xxd -p        # 676f
   echo -n "go" | base64        # Z28=
   echo -n "676f" | xxd -r -p   # go
   echo -n "Z28=" | base64 -d   # go
   ```
   </details>

---

## chmod & permissions

Every file has three permission triads — owner, group, others — each with read/write/execute bits. `chmod` changes those bits; reading them correctly is half the job.

### Reading `ls -l`

```
-rwxr-xr--  1 alice staff  1240 Jan 15 10:23 deploy.sh
│└┬┘└┬┘└┬┘
│ │  │  └── others: r-- (read only)
│ │  └───── group:  r-x (read + execute)
│ └──────── owner:  rwx (read + write + execute)
└────────── file type: - regular file, d directory, l symlink
```

Each triad is 3 bits: **r**ead, **w**rite, e**x**ecute (or `-` if absent). For a directory, `x` means "can enter/traverse it," and `r` means "can list its contents."

### Numeric (octal) mode

Each permission has a value: `r=4, w=2, x=1`. Add them per triad to get one digit each for owner/group/others.

| Digit | Permissions |
|---|---|
| 7 | rwx (4+2+1) |
| 6 | rw- (4+2) |
| 5 | r-x (4+1) |
| 4 | r-- (4) |
| 0 | --- (nothing) |

`chmod 755 file` → owner rwx, group r-x, others r-x. `chmod 644 file` → owner rw-, group r--, others r-- (the standard "readable, editable by me only" combo).

### Symbolic mode

```bash
chmod [ugoa][+-=][rwx] file
```

| Who | Op | What |
|---|---|---|
| `u` owner, `g` group, `o` others, `a` all | `+` add, `-` remove, `=` set exactly | `r`, `w`, `x` |

```bash
chmod u+x script.sh      # give the owner execute permission
chmod go-w file.txt      # remove write from group and others
chmod a=r file.txt       # set everyone to read-only, exactly
chmod u+x,g-w script.sh  # multiple changes, comma-separated
```

### Related commands & concepts

| Command | Meaning |
|---|---|
| `chown user file` | change the file's owner |
| `chown user:group file` | change owner and group together |
| `chgrp group file` | change only the group |
| `chmod -R` | apply recursively to a whole directory tree |
| `umask` | the default permission mask subtracted from new files (commonly `022`, giving new files 644 and new dirs 755) |

**Special bits** (a 4th leading octal digit): `setuid (4000)` runs a program as its owner rather than the caller; `setgid (2000)` makes new files in a directory inherit its group; the `sticky bit (1000)` on a shared directory (like `/tmp`) stops users from deleting each other's files. You'll meet these far less often than the basic rwx bits.

> **Gotcha:** `chmod 777` ("everyone can do everything") is almost never the right fix for a permission error — it silences the symptom while opening the file to writes from any user on the system. Diagnose the actual owner/group mismatch with `ls -l` first.

### Practice problems

1. You see `-rw-r--r--` on a file. Give the owner execute permission too, using symbolic mode.
   <details><summary>Show solution</summary>

   ```bash
   chmod u+x file
   ```
   </details>

2. Set a script to be fully executable and writable by the owner, readable and executable (not writable) by everyone else — write both the numeric and symbolic form.
   <details><summary>Show solution</summary>

   ```bash
   chmod 755 script.sh
   # equivalent symbolic form:
   chmod u=rwx,go=rx script.sh
   ```
   </details>

3. Make a config file readable and writable by the owner only — nobody else should have any access.
   <details><summary>Show solution</summary>

   ```bash
   chmod 600 config.txt
   ```
   </details>

4. Given `drwxr-xr-x` on a directory, explain in one sentence whether "others" can create a new file inside it, and why.
   <details><summary>Show solution</summary>

   No — "others" has `r-x` (read + traverse) but not `w`rite on the directory itself, and write on the *directory* is what's required to create, rename, or delete entries inside it, regardless of file permissions inside.
   </details>

5. Recursively give the group write access to every file under a shared project folder, without changing owner or others.
   <details><summary>Show solution</summary>

   ```bash
   chmod -R g+w ./project
   ```
   </details>

6. You `ls -l` a file and see `-rwxr-xr-x 1 root root ...`. As a non-root, non-group user, can you edit it? Can you run it?
   <details><summary>Show solution</summary>

   You fall under "others" (`r-x`): you can read and execute it, but you cannot write/edit it — only `root` (the owner) can.
   </details>

---

## Final challenge — combine everything

These pull two or more tools together in one pipeline, the way you'll actually use them day to day.

1. From `server.log`, extract all unique IP addresses, sorted.
   <details><summary>Show solution</summary>

   ```bash
   grep -Eo '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' server.log | sort -u
   ```
   </details>

2. Find every `.log` file under the current directory larger than 0 bytes, and for each, print how many ERROR lines it contains.
   <details><summary>Show solution</summary>

   ```bash
   find . -name "*.log" -size +0c -exec sh -c 'echo -n "$1: "; grep -c ERROR "$1"' _ {} \;
   ```
   </details>

3. Using `staff.csv`: print `name: salary` for everyone in Engineering, sorted by salary descending.
   <details><summary>Show solution</summary>

   ```bash
   awk -F',' 'NR>1 && $2=="Engineering" {print $1": "$3}' staff.csv | sort -t: -k2 -nr
   ```
   </details>

4. Redact every IP address in `server.log` to `[REDACTED]`, save the result to `server.clean.log`, and confirm no IPs remain using `grep`.
   <details><summary>Show solution</summary>

   ```bash
   sed -E 's/[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[REDACTED]/g' server.log > server.clean.log
   grep -Ec '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' server.clean.log
   # should print 0
   ```
   </details>

5. You're handed a file called `drop` with no extension. Identify its real type, dump its first 16 bytes in hex, and — if it turns out to contain text — extract any readable strings 6 characters or longer.
   <details><summary>Show solution</summary>

   ```bash
   file drop
   xxd -l 16 drop
   strings -n 6 drop
   ```
   </details>

6. You find a suspicious base64 blob inside a log file, on a line starting with `PAYLOAD=`. Pull out just the blob, decode it, and hex-dump the first 16 bytes of the decoded result to check its file type by magic bytes.
   <details><summary>Show solution</summary>

   ```bash
   grep '^PAYLOAD=' file.log | sed 's/^PAYLOAD=//' | base64 -d > decoded.bin
   xxd -l 16 decoded.bin
   ```
   </details>

7. A deploy script needs to run as its owner, be readable by the whole team's group, and be completely inaccessible to everyone else. Set the correct permissions and verify with `ls -l`.
   <details><summary>Show solution</summary>

   ```bash
   chmod 750 deploy.sh
   ls -l deploy.sh
   # -rwxr-x--- ... deploy.sh
   ```
   </details>

---

*Practice beats reading — rerun each solved problem with a small variation before moving on.*
