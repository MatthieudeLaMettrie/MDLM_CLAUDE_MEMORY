# Environment Variables — Explained Simply

## What is an environment variable?

A named box of text (`NAME=value`) that the operating system keeps and hands
to programs when they start. Programs read these boxes to configure their
own behavior — without you having to edit their code or settings files.

Think of it like sticky notes the OS keeps on hand. When a program launches,
the OS says "here are all my sticky notes" and the program can read any of
them it cares about.

```
NAME            = VALUE
PATH            = C:\Windows\System32;C:\Program Files\Git\bin
TEMP            = C:\Users\mdlm\AppData\Local\Temp
OPENAI_API_KEY  = sk-abc123...
DEBUG           = true
```

---

## Why they exist (the core purposes)

### 1. Configuration without touching code
Instead of hardcoding a setting inside a program, the program checks an
environment variable at startup.

**Example:** Claude Code checks `CLAUDE_CODE_USE_POWERSHELL_TOOL`. If it's
`1`, Claude Code uses PowerShell as its shell tool. No code change needed —
just flip the variable.

### 2. Secrets and credentials
API keys and tokens should never be typed directly into scripts (someone
could commit them to git by accident). Instead, the script reads them from
an environment variable at runtime.

**Example:**
```bash
# Bad — key visible in the script file
curl -H "Authorization: Bearer sk-abc123" https://api.example.com

# Good — key lives in an environment variable, not in the file
curl -H "Authorization: Bearer $OPENAI_API_KEY" https://api.example.com
```

### 3. Telling programs where things are (PATH)
`PATH` is the most common environment variable. It's a list of folders.
When you type `git` in a terminal, the OS searches every folder listed in
`PATH` until it finds `git.exe`.

**Example:** if `PATH` didn't include Git's install folder, typing `git`
anywhere would give `'git' is not recognized as an internal or external command`.

### 4. Per-user / per-machine settings
The same program can behave differently on different computers, because
each machine (or user account) has its own copy of these variables.

**Example:** your laptop might have `NODE_ENV=development` while the
production server has `NODE_ENV=production` — same app code, different
behavior, because of one variable.

### 5. Feature flags / toggles
Turning on debug logging, verbose output, or experimental features without
a config file.

**Example:** `DEBUG=true` might make an app print extra logs; `DEBUG=false`
(or unset) keeps it quiet.

### 6. Passing info from parent process to child process
When Program A starts Program B, B automatically inherits A's environment
variables. This is how settings "flow down" without explicit arguments.

**Example:** when you open a terminal, it inherits your user's environment
variables — that's why `echo $PATH` (or `echo $env:PATH` in PowerShell)
works immediately without you setting anything in that terminal.

---

## Scopes in Windows

| Scope | Applies to | Needs admin? | Stored in |
|---|---|---|---|
| **User** | Just your Windows account | No | Registry: `HKCU\Environment` |
| **System** | Every user on the machine | Yes | Registry: `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` |

Windows does **not** use a text file for permanent environment variables —
they live in the **registry**, a built-in database Windows uses for
settings. You can view/edit them via:
- GUI: `Win+R` → `sysdm.cpl` → *Advanced* tab → *Environment Variables*
- PowerShell: `[Environment]::GetEnvironmentVariable('NAME', 'User')`

## Session vs. Persistent

| Type | Example | Lasts |
|---|---|---|
| **Session-only** | `$env:DEBUG = "true"` (PowerShell) or `export DEBUG=true` (bash) | Only in that one open terminal window |
| **Persistent (User)** | `[Environment]::SetEnvironmentVariable('DEBUG','true','User')` | Every new terminal/program, forever, until removed |

Setting a session-only variable is like writing on a whiteboard that gets
erased when you close the terminal. Setting a persistent one is like
writing it into the registry — every new terminal reads it fresh.

---

## Quick real-world example, end to end

1. You get an API key from a service: `sk-abc123`.
2. Instead of pasting it into your Python script, you set it once:
   ```powershell
   [Environment]::SetEnvironmentVariable('MY_API_KEY', 'sk-abc123', 'User')
   ```
3. Your script reads it:
   ```python
   import os
   key = os.environ["MY_API_KEY"]
   ```
4. Now the script works on any machine where that variable is set, and the
   key never appears in the code or in git.

---

## TL;DR

Environment variables = a global, named settings store the OS keeps and
every program can read. Used for configuration, secrets, locating tools
(`PATH`), per-machine differences, and passing settings from parent to
child processes — all without editing code.
