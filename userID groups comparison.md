# Compare Linux User Group Memberships

This repository provides simple commands and workflows to compare the group memberships of two Linux users.

It is useful for system administrators who want to audit group assignments, identify permission differences, and debug access issues.

---

## 🔍 Overview

The comparison works by retrieving group names for each user, saving them to files, and using standard Linux utilities to find similarities and differences.

---

## 🧩 Step 1: Save Group Memberships to Files

```bash
id -nG user1 | tr ' ' '\n' | sort > user1_groups.txt
id -nG user2 | tr ' ' '\n' | sort > user2_groups.txt
````

### Command Breakdown

* `id -nG` → display group names for the user
* `tr ' ' '\n'` → place each group on a new line
* `sort` → sort group names alphabetically
* `>` → redirect output to a file

---

## 🔄 Step 2: Compare Group Files

### Show all differences

```bash
diff user1_groups.txt user2_groups.txt
```

### Show common groups

```bash
comm -12 user1_groups.txt user2_groups.txt
```

### Groups only in user1

```bash
comm -23 user1_groups.txt user2_groups.txt
```

### Groups only in user2

```bash
comm -13 user1_groups.txt user2_groups.txt
```

---

## 📊 Example

### User memberships

* **user1**:

```text
developers
testers
staff
```

* **user2**:

```text
admins
staff
```

### Comparison results

#### Common groups

```text
staff
```

#### Only in user1

```text
developers
testers
```

#### Only in user2

```text
admins
```

---

## 🚀 Optional Automation Script

Create a reusable script for automation.

### compare_groups.sh

```bash
#!/bin/bash
# Usage: ./compare_groups.sh user1 user2

u1="$1"
u2="$2"

id -nG "$u1" | tr ' ' '\n' | sort > "${u1}_groups.txt"
id -nG "$u2" | tr ' ' '\n' | sort > "${u2}_groups.txt"

echo "Common groups:"
comm -12 "${u1}_groups.txt" "${u2}_groups.txt"

echo
echo "Groups only in $u1:"
comm -23 "${u1}_groups.txt" "${u2}_groups.txt"

echo
echo "Groups only in $u2:"
comm -13 "${u1}_groups.txt" "${u2}_groups.txt"
```

### Make executable and run

```bash
chmod +x compare_groups.sh
./compare_groups.sh user1 user2
```

---

## ✅ Why This Is Useful

* Audit Linux user group memberships
* Identify permission overlaps
* Detect configuration mismatches
* Troubleshoot access issues

---
