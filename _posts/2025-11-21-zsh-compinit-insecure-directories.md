---
title:  "zsh compinit: insecure directories"
date:   2025-11-21 15:16:50 +0530
categories:
  - blog
  - issues
tags:
  - Linux
  - Zsh
---



## Understanding and Fixing Zsh “Insecure Files”

The issue began with Zsh refusing to initialize its completion system due to “insecure files.”

```
zsh compinit: insecure directories, run compaudit for list.
Ignore insecure directories and continue [y] or abort compinit [n]?
```

### Why the “insecure files” error occurs

**What is compinit?**

Zsh has an advanced tab-completion system.
The function compinit:
- Scans a set of directories called `$fpath` (function search path).
- Finds completion-related functions (files starting with _, like _git, _ssh, _antigravity).
- Builds an index of them and saves that into a cache file: ~/.zcompdump.
- Loads that cache on future shells so completion is fast.

These functions have full control over how completion behaves and can execute code. For security reasons, Zsh checks every file and directory in `$fpath` before loading them. Any item that is world-writable, group-writable, or owned by an untrusted user is treated as unsafe.

### What `compinit` and `compaudit` do

* **`compinit`** initializes the completion system and performs mandatory security checks.
* **`compaudit`** is the auditing component used to identify insecure items in `$fpath`.

When `compinit` encounters a suspicious file, it halts initialization unless explicitly overridden. Running `compaudit` revealed the problem:

```bash
/usr/share/zsh/vendor-completions/_antigravity
```
It lists the files that are unsafe.

### What we found

The file was not writable by group or world. The failure was caused by ownership:

```bash
-rw-r--r-- 1 nobody nogroup …
```

Zsh rejects completion functions owned by `nobody`, because such files are considered untrustworthy.

### How the insecure file was fixed

The fix required correcting ownership:

```sh
sudo chown root:root /usr/share/zsh/vendor-completions/_antigravity
```

After this change, `compaudit` returned no insecure files.

### The follow-up parse error

Once the ownership issue was fixed, Zsh produced a different error:

```
compinit:489: bad math expression: operand expected at end of string
```

This error was not related to permissions. It appeared because earlier, when `compinit` aborted due to insecure files, it left behind a partially written or corrupted `~/.zcompdump` file. Later executions attempted to read this file and failed while parsing it.

### Fixing the parse error

Removing the corrupted dump file and forcing a clean reinitialization resolved the problem:

```sh
rm -f ~/.zcompdump*
autoload -U compinit
compinit
```

### Summary

The main issue was a completion file owned by `nobody`, which triggered Zsh’s security checks. `compaudit` identified the insecure file, correcting ownership resolved the security warning, and deleting the corrupted `~/.zcompdump` file removed the parse error that resulted from the earlier aborted initialization.
