# Shell Scripting — Interview Fundamentals

## Core Concepts

### What is Shell Scripting?
A shell script is a text file containing a sequence of commands executed by a Unix/Linux shell (bash, sh, zsh, ksh). It automates repetitive tasks, manages system administration, and orchestrates deployments.

---

## Key Topics

### 1. Shebang & Script Basics
```bash
#!/usr/bin/env bash   # portable shebang
set -euo pipefail     # fail fast, treat unset vars as errors, propagate pipe failures
```
- `set -e` — exit immediately on error
- `set -u` — treat unset variables as errors
- `set -o pipefail` — return exit code of the first failed command in a pipe

### 2. Variables
```bash
NAME="world"
echo "Hello, $NAME"
readonly PI=3.14          # immutable
unset NAME                # delete variable
echo "${NAME:-default}"   # use default if unset
echo "${NAME:?error msg}" # exit with error if unset
```

### 3. Conditionals
```bash
if [[ -f "/etc/passwd" ]]; then
    echo "file exists"
elif [[ -d "/tmp" ]]; then
    echo "directory exists"
else
    echo "neither"
fi

# Integer comparison
[[ $a -eq $b ]]   # equal
[[ $a -ne $b ]]   # not equal
[[ $a -lt $b ]]   # less than
[[ $a -gt $b ]]   # greater than

# String comparison
[[ "$s" == "foo" ]]
[[ "$s" =~ ^[0-9]+$ ]]  # regex match
```

### 4. Loops
```bash
# for loop
for i in {1..10}; do echo "$i"; done

# while loop
while IFS= read -r line; do
    echo "$line"
done < input.txt

# until loop
count=0
until [[ $count -ge 5 ]]; do
    ((count++))
done

# C-style for
for ((i=0; i<10; i++)); do echo "$i"; done
```

### 5. Functions
```bash
greet() {
    local name="$1"           # local scope
    echo "Hello, $name"
    return 0                  # explicit exit code
}
greet "Alice"
result=$(greet "Bob")         # capture output
```

### 6. Arrays
```bash
arr=("a" "b" "c")
echo "${arr[0]}"              # first element
echo "${arr[@]}"              # all elements
echo "${#arr[@]}"             # array length
arr+=("d")                    # append

# Associative array (bash 4+)
declare -A map
map["key"]="value"
echo "${map["key"]}"
```

### 7. String Manipulation
```bash
str="Hello World"
echo "${#str}"                # length: 11
echo "${str,,}"               # lowercase
echo "${str^^}"               # uppercase
echo "${str/World/Bash}"      # replace first
echo "${str//l/L}"            # replace all
echo "${str:6:5}"             # substring: World
echo "${str#Hello }"          # strip prefix
echo "${str%World}"           # strip suffix
```

### 8. Exit Codes & Error Handling
```bash
command || { echo "command failed"; exit 1; }
trap 'echo "Error on line $LINENO"' ERR
trap 'cleanup' EXIT           # always run on exit

check_exit() {
    if [[ $? -ne 0 ]]; then
        echo "Previous command failed"
    fi
}
```

### 9. Input / Output
```bash
read -p "Enter name: " username
read -s -p "Password: " pass   # silent input

# Redirection
command > out.txt       # stdout to file
command >> out.txt      # append
command 2> err.txt      # stderr to file
command 2>&1            # redirect stderr to stdout
command &> all.txt      # both stdout and stderr
command < in.txt        # stdin from file

# Here-doc
cat <<EOF
multi
line
text
EOF
```

### 10. Process & Signal Management
```bash
# Background processes
long_command &
pid=$!
wait $pid

# Signal trapping
trap 'echo "Caught SIGINT"; exit 1' INT TERM

# Check if process is running
if kill -0 "$pid" 2>/dev/null; then
    echo "running"
fi
```

### 11. File & Directory Operations
```bash
[[ -f file ]]     # is a regular file
[[ -d dir ]]      # is a directory
[[ -r file ]]     # readable
[[ -w file ]]     # writable
[[ -x file ]]     # executable
[[ -s file ]]     # non-empty file
[[ -L file ]]     # symbolic link
[[ file1 -nt file2 ]]   # newer than
```

### 12. Common Text Processing Tools
| Tool | Purpose |
|------|---------|
| `grep` | Pattern search |
| `sed` | Stream editor / text substitution |
| `awk` | Field-based text processing |
| `cut` | Extract columns |
| `sort` | Sort lines |
| `uniq` | Remove duplicate lines |
| `tr` | Translate/delete characters |
| `wc` | Word/line/char count |
| `xargs` | Build command lines from stdin |

```bash
# Examples
grep -rn "pattern" /var/log/
sed -i 's/old/new/g' file.txt
awk '{print $1, $3}' data.txt
cut -d',' -f1,3 data.csv
sort -k2 -n file.txt | uniq -c
```

---

## Common Interview Questions

### Q1: What is the difference between `[` and `[[`?
`[` is POSIX-compliant and available in all shells. `[[` is a bash built-in that supports regex (`=~`), handles empty strings safely, and doesn't require quoting variables.

### Q2: How do you handle errors in a script?
- Use `set -euo pipefail` at the top
- Use `trap` for cleanup on errors/exit
- Check `$?` after critical commands
- Use `|| { ... ; exit 1; }` for inline error handling

### Q3: Difference between `$*` and `$@`?
Both represent all positional parameters. When quoted, `"$*"` expands as a single word (`"$1 $2 $3"`), while `"$@"` expands as separate words (`"$1" "$2" "$3"`). Always use `"$@"` when iterating arguments.

### Q4: How do you debug a shell script?
```bash
bash -x script.sh        # trace execution
bash -n script.sh        # syntax check only
set -x                   # enable trace within script
set +x                   # disable trace
```

### Q5: What is a subshell?
A child process spawned by the current shell. Variables set in a subshell do not affect the parent. Created by `()`, `$()`, pipes, and `&`.

### Q6: How to pass arguments to a script?
```bash
./script.sh arg1 arg2
# Inside script:
$0  # script name
$1  # first argument
$#  # number of arguments
$@  # all arguments (array-like)
```

### Q7: How do you schedule scripts?
Using `cron`:
```
# min hour day month weekday command
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

### Q8: Explain process substitution
```bash
diff <(sort file1) <(sort file2)   # compare sorted outputs
while read line; do ...; done < <(command)
```

---

## Practical Patterns for Finance/Banking Environments

### Deployment Script Skeleton
```bash
#!/usr/bin/env bash
set -euo pipefail

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/deploy_$(date +%Y%m%d_%H%M%S).log"
readonly APP_NAME="myapp"

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"; }
die() { log "ERROR: $*"; exit 1; }

cleanup() { log "Cleanup triggered"; }
trap cleanup EXIT

main() {
    log "Starting deployment of $APP_NAME"
    [[ -z "${DEPLOY_ENV:-}" ]] && die "DEPLOY_ENV is not set"
    # ... deployment steps
    log "Deployment complete"
}

main "$@"
```

### Retry Logic
```bash
retry() {
    local max=$1; shift
    local delay=$1; shift
    local attempt=1
    until "$@"; do
        if ((attempt >= max)); then
            echo "Command failed after $max attempts"
            return 1
        fi
        echo "Attempt $attempt failed. Retrying in ${delay}s..."
        sleep "$delay"
        ((attempt++))
    done
}
retry 3 5 curl -f http://healthcheck/endpoint
```

---

## Cheat Sheet

| Concept | Syntax |
|---------|--------|
| Declare variable | `VAR=value` |
| Export to env | `export VAR=value` |
| Command substitution | `$(command)` |
| Arithmetic | `$((a + b))` |
| String length | `${#var}` |
| Default value | `${var:-default}` |
| Null check & exit | `${var:?message}` |
| Positional args | `$1`, `$2`, ... `$#` |
| Last exit code | `$?` |
| Current PID | `$$` |
| Last background PID | `$!` |
