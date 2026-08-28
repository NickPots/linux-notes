# Linux Deep Dive: `find` and Core Text-Processing Tools

Notes from learning `find` and the text-processing commands that pair with it (`grep`, `sort`, `uniq`, `strings`). Written as part of my Linux/sysadmin learning journey.

## 1. `find` — Locating Files

Basic structure:

```bash
find [where] [conditions] [action]
```

### Search by type

`-type` filters by filesystem object type, not content:

| Flag | Meaning |
|------|---------|
| `f` | regular file |
| `d` | directory |
| `l` | symbolic link |
| `b` | block device |
| `c` | character device |
| `s` | socket |
| `p` | named pipe |

```bash
find . -type f -name "*.txt"
find . -type d -empty
```

### Search by size

```bash
find . -type f -size +100M      # larger than 100 MB
find . -type f -size -10k       # smaller than 10 KB
find . -type f -size +1M -size -10M   # range: 1MB–10MB
```

Units: `c` (bytes), `k` (KB), `M` (MB), `G` (GB), `b` (512-byte blocks, default).

### Search by owner / group

```bash
find . -user john
find . -group developers
find . -user $(whoami)          # files owned by current user
find . -not -user $(whoami)     # files NOT owned by current user
find . -nouser                  # orphaned files (owner no longer exists)
```

### Search by executable permission

```bash
find . -type f -executable      # has execute permission
find . -type f -not -executable # does NOT have execute permission
```

Note: this checks the **permission bit**, not the actual file content. A script without `chmod +x` will show up as "not executable" here even though it's clearly a script.

### Combining `find` with `file` (real content type)

`find -type` only knows filesystem object types. To check actual content (ASCII text, ELF binary, script, etc.), pipe into `file`:

```bash
find . -type f -exec file {} +
find . -type f -exec file {} + | grep "ELF"          # real binaries only
find . -type f -exec file {} + | grep "ASCII text"   # plain text files
```

Use `+` instead of `\;` after `-exec` — it batches all files into a single `file` call instead of spawning one process per file (faster on large directories).

### Suppressing errors

`find` often hits directories it can't read (e.g. searching from `/`). Those errors go to **stderr** and can be redirected away:

```bash
find / -type f -name "*.conf" 2>/dev/null
```

| FD | Name | Purpose |
|----|------|---------|
| `0` | stdin | input |
| `1` | stdout | normal output |
| `2` | stderr | error messages |

Other useful redirections:

```bash
find / -name "*.txt" > /dev/null 2>&1   # discard everything
find / -name "*.txt" &>/dev/null        # shorthand (bash/zsh)
```

## 2. `grep` — Pattern Matching

```bash
grep "error" logfile.txt
```

| Flag | Effect |
|------|--------|
| `-i` | case-insensitive |
| `-v` | invert match (show non-matching lines) |
| `-c` | count matches |
| `-n` | show line numbers |
| `-r` | recursive through directories |
| `-E` | extended regex (e.g. `"error|warning"`) |
| `-w` | match whole words only |

## 3. `sort` — Ordering Lines

```bash
sort names.txt
sort -n numbers.txt     # numeric sort (default sort is lexical: "10" < "2")
sort -r names.txt       # reverse order
sort -k2 data.txt       # sort by 2nd column
sort -u names.txt       # sort + remove duplicates
```

## 4. `uniq` — Deduplicating Lines

`uniq` only removes **adjacent** duplicate lines, so input almost always needs to be sorted first:

```bash
sort file.txt | uniq
```

| Flag | Effect |
|------|--------|
| `-c` | prefix each line with its count |
| `-d` | show only duplicated lines |
| `-u` | show only unique (non-duplicated) lines |

## 5. `strings` — Extracting Readable Text from Binaries

```bash
strings /bin/ls
```

Prints readable text embedded in a binary — error messages, version info, linked libraries, hardcoded paths — without needing a disassembler.

| Flag | Effect |
|------|--------|
| `-n 10` | only strings with at least 10 characters (less noise) |
| `-a` | scan the entire file, not just certain sections |

## 6. Putting It Together

Find all ELF executables in a directory tree, batched for speed:

```bash
find . -type f -exec file {} + | grep "ELF" | cut -d: -f1
```

Find large files system-wide without error spam:

```bash
find / -type f -size +100M 2>/dev/null -exec ls -lh {} \;
```

Count and rank failed login attempts by IP in a log file:

```bash
grep "failed login" /var/log/auth.log | awk '{print $NF}' | sort | uniq -c | sort -rn
```

Search a suspicious binary for embedded URLs:

```bash
strings suspicious_binary | grep -E "http|www"
```

## Key Takeaway

`find` handles *locating* files by metadata (type, size, owner, permissions). It doesn't know about file *content* — for that, pair it with `file` (content type), `grep` (pattern search), and `strings` (readable text in binaries). `sort` and `uniq` turn raw output from any of these into ranked, deduplicated summaries. This combination — locate, inspect, filter, count — is the core pattern behind most Linux command-line investigation.
