# omp-sandbox

A macOS [Seatbelt](https://www.chromium.org/developers/design-documents/sandbox/osx-sandboxing-design/) wrapper for [Oh My Pi (omp)](https://omp.dev) that confines the AI agent to a specific workspace directory with strict read and write boundaries — without patching omp or adding any runtime dependencies.

## What it does

| Boundary | Policy |
|---|---|
| **Write** | Allowed only in your workspace + OMP's own state dirs + macOS runtime scratch |
| **Read ($HOME)** | Allowlist: `~/.omp` and your workspace only — everything else in `$HOME` is denied |
| **Read (system)** | Open via `allow default` — OMP's runtimes need `/usr`, `/System`, `/Library`, `/opt/homebrew`, etc. |
| **Network** | Open — Seatbelt cannot do per-domain filtering; OMP needs model APIs |
| **YOLO** | Opt-in (`OMP_SANDBOX_YOLO=1`) — auto-approves all tool calls |

Personal data that is read-denied by default: Keychain, Messages, Mail, Calendars, Contacts, iCloud Drive, Safari, Health, Wallet, Passes, and all other `$HOME` paths not in the allowlist.

## Requirements

- macOS (any version shipping `/usr/bin/sandbox-exec`)
- [Oh My Pi](https://omp.dev) installed and on `PATH` (e.g. `brew install omp`)

## Quick start

```bash
# 1. Clone
git clone https://github.com/bnivanov/omp-sandbox ~/.omp/sandbox

# 2. Make executable
chmod +x ~/.omp/sandbox/omp-sandboxed

# 3. Verify it works from your project directory
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --self-test

# 4. Run omp inside the sandbox
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed

# 5. Optional: add a shell alias
echo 'alias omp-sandbox="~/.omp/sandbox/omp-sandboxed"' >> ~/.zshrc
```

## Configuration

All configuration is via environment variables — no config files.

| Variable | Default | Purpose |
|---|---|---|
| `OMP_SANDBOX_WORKSPACE` | `$PWD` | Directory OMP can read and write. Defaults to wherever you run the script from. |
| `OMP_SANDBOX_YOLO` | `0` | Set to `1` to pass `--auto-approve` to omp (no tool-approval prompts). Use only in sandboxed sessions. |
| `OMP_SANDBOX_EXTRA_WRITE` | _(empty)_ | Colon-separated extra writable paths. Leading `~` expands to `$HOME`. Each path is canonicalized and checked against a denylist (`.ssh`, `.aws`, Keychains, etc.). |
| `OMP_SANDBOX_PASS_ENV` | _(empty)_ | **Colon-separated env var names to forward into the clean sandbox env. Required for provider API keys** — e.g. `OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY`. Without this, OMP cannot reach its model API. |
| `OMP_SANDBOX_INHERIT_ENV` | `0` | Set to `1` to pass the full parent environment instead of the minimal allowlist. Insecure — exposes all shell secrets to the sandboxed process and any host it reaches. |
| `OMP_SANDBOX_CONFIRM_WIDE` | `0` | Set to `1` to allow `$HOME` or `/` as workspace (otherwise rejected). Insecure. |
| `OMP_SANDBOX_SKIP_HOME_VERIFY` | `0` | Set to `1` to skip the `dscl` home-directory verification. Insecure — allows HOME-poisoning attacks. |
| `OMP_SANDBOX_CONFIRM_SENSITIVE_WRITE` | `0` | Set to `1` to allow sensitive paths (`.ssh`, `.aws`, Keychains, etc.) in `OMP_SANDBOX_EXTRA_WRITE`. Insecure. |

Examples:

```bash
# Fix the workspace to a specific path regardless of cwd.
# Unquoted ~ is expanded by the shell; the script also handles a literal ~ if you
# export the variable (export OMP_SANDBOX_WORKSPACE='~/proj' works too).
OMP_SANDBOX_WORKSPACE=~/projects/myproject ~/.omp/sandbox/omp-sandboxed

# YOLO mode — auto-approve all tool calls
OMP_SANDBOX_WORKSPACE=~/projects/myproject OMP_SANDBOX_YOLO=1 ~/.omp/sandbox/omp-sandboxed

# Allow omp to write screenshots to ~/Downloads in addition to the workspace
OMP_SANDBOX_EXTRA_WRITE=~/Downloads ~/.omp/sandbox/omp-sandboxed

# Pass provider API key into the minimal clean environment (required by default)
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY OMP_SANDBOX_WORKSPACE=~/projects/myproject \
  ~/.omp/sandbox/omp-sandboxed

# Multiple keys, colon-separated
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY:OPENAI_API_KEY ~/.omp/sandbox/omp-sandboxed

# Legacy: pass full parent environment — exposes all shell secrets (insecure)
OMP_SANDBOX_INHERIT_ENV=1 ~/.omp/sandbox/omp-sandboxed
```

## Security model

### Write allowlist

Only these locations are writable inside the sandbox:

| Path | Reason |
|---|---|
| `$OMP_SANDBOX_WORKSPACE` | Your project — OMP's working directory |
| `~/.omp` | OMP's sessions, memories, and logs |
| `/private/tmp`, `$TMPDIR` | macOS temp files (Bun, Python) |
| `/private/var/folders` | Bun/Python bytecode caches |
| `/dev` | Device nodes (`/dev/null`, `/dev/urandom`, pipes) |
| `$OMP_SANDBOX_EXTRA_WRITE` | Opt-in extra paths |

### Read allowlist (within `$HOME`)

Only these `$HOME` paths are readable:

- `~/.omp` — OMP configuration, memories, session state
- `$OMP_SANDBOX_WORKSPACE` — your project

All other `$HOME` paths are denied: `~/.ssh`, `~/.aws`, `~/.zsh_history`, `~/.docker`, `~/.kube`, `~/Library/Messages`, `~/Library/Mail`, `~/Library/Keychains`, `~/Library/Mobile Documents` (iCloud), `~/Library/Health`, `~/Library/Passes`, Safari, Contacts, and everything else.

System paths outside `$HOME` remain readable — OMP's runtimes require them.

### Environment isolation

By default the launcher uses `env -i` to give the sandboxed process only a minimal allowlist (PATH, HOME, TMPDIR, locale, terminal variables). All other shell variables — AWS credentials, GitHub tokens, and any other secrets in your shell — are **stripped automatically**.

OMP needs its model provider key to operate. Pass it explicitly:

```bash
OMP_SANDBOX_PASS_ENV=ANTHROPIC_API_KEY ~/.omp/sandbox/omp-sandboxed
```

To revert to the old full-env behaviour (not recommended):

```bash
OMP_SANDBOX_INHERIT_ENV=1 ~/.omp/sandbox/omp-sandboxed
```

### HOME verification

The launcher resolves the real user home via `/usr/bin/dscl` and **rejects launch** if the inherited `$HOME` differs — this is the exact vector for read-confinement bypass (a malicious wrapper could set `HOME=/tmp` before launch). Set `OMP_SANDBOX_SKIP_HOME_VERIFY=1` to override.

### Known remaining surface

- **`/Volumes`** — mounted external/network drives are readable. Add `(deny file-read* (subpath "/Volumes"))` inside `gen_profile` if needed.
- **Other `/Users/*` accounts** — readable on a multi-user Mac. Acceptable for a single-user personal machine.
- **Network** — all hosts reachable. Use an external proxy/filter for egress control.
- **`~/.omp`** — writable by the sandboxed process. A compromised agent could modify OMP's own memories and sessions.

### What this does NOT protect against

- Network exfiltration — OMP (and all child processes) can reach any host
- Attacks through OMP's memory/session store — `~/.omp` is intentionally writable

## Verifying the sandbox

```bash
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --self-test
```

All lines should read `PASS:` and the script exits 0. The self-test checks:

- Workspace writes succeed
- `$HOME` writes outside the workspace are denied (genuine Seatbelt denial, not just a permissions error)
- `~/.omp` is both readable and writable
- Workspace reads work (proves the `$HOME` deny + re-allow rule resolves correctly)
- Keychain, Messages, and iCloud reads are denied
- `omp --version` boots cleanly under the minimal clean environment (same as real launches)
- `--auto-approve` flag is accepted by the installed omp version
- `OMP_SANDBOX` and `PI_SANDBOX` env markers are present inside the clean env
- `TMPDIR` canonicalizes to a valid macOS scratch root, not an attacker-controlled path
- A secret canary is absent from the `env -i` sandbox — env isolation is verified end-to-end
- `.ssh` is detected as a sensitive `EXTRA_WRITE` target and rejected by the denylist

## Printing the active profile

```bash
cd ~/projects/myproject
~/.omp/sandbox/omp-sandboxed --print-profile
```

The profile is plain [SBPL](https://reverse.put.as/wp-content/uploads/2011/09/Apple-Sandbox-Guide-v1.0.pdf) (~30 lines). Review it to understand exactly what is and is not permitted.

## How it works

macOS ships `/usr/bin/sandbox-exec` on every Mac (backed by the TrustedBSD MAC framework, known internally as Seatbelt). It applies a profile written in SBPL (Sandbox Profile Language) to a process and all its children — no root required, no VM, no containers.

This wrapper:
1. Computes a per-launch SBPL profile from the current workspace path and environment
2. Writes it to a temp file
3. Invokes `sandbox-exec -f <profile> omp [args]`

The profile uses `(allow default)` as the base, then applies targeted write denials + re-allows, and a `$HOME` read deny + minimal re-allow. Rule evaluation is last-match-wins within each operation class.

`sandbox-exec` is officially deprecated by Apple but remains present and functional on all current macOS releases. There is no replacement API for unprivileged sandboxing of arbitrary binaries.

## License

MIT — see [LICENSE](LICENSE).
