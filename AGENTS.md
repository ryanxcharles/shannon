# Shannon

A poly-shell built on nushell, with seamless bash compatibility via a persistent
bash subprocess. Shift+Tab cycles between nu and bash modes. Named after Claude
Shannon.

## Build

```sh
cargo build
cargo run
```

The Rust crate is at the repo root.

## Architecture

Shannon IS nushell — it copies the nushell binary source code and adds mode
dispatch for bash (via a persistent bash subprocess). Nushell's REPL handles
terminal ownership, process groups, job control, signal handling, multiline
editing, completions, and all interactive features. Shannon adds a
`ModeDispatcher` trait (defined in nu-cli) that intercepts commands when the
mode is not "nu".

### Repo structure

```
shannon/              (repo root = shannonshell crate)
├── Cargo.toml        (binary + library)
├── src/              (shannon source)
├── nushell/          (nushell source, merged into repo)
├── reedline/         (reedline source, merged into repo)
├── vendor/           (reference source code, gitignored)
├── issues/           (issue tracking with experiments)
└── scripts/          (build, install, release, sync-upstream)
```

### Merged dependencies

Nushell and reedline source is merged directly into the repo with full git
history preserved (via `git subtree add`). Shannon uses path deps to reference
their crates. No crates.io publishing needed for dependencies.

- **nushell/** — fork of `nushell/nushell`. Changes: `ModeDispatcher` trait in
  nu-cli, `BashHighlighter` in nu-cli, Shift+Tab keybinding.
- **reedline/** — fork of `nushell/reedline`. No code changes.

Upstream sync: `git subtree pull --prefix nushell upstream-nushell main`

### Source files (under `src/`)

**Copied from nushell binary (the shell):**

- `main.rs` — entry point, startup sequence (from nu binary, modified)
- `run.rs` — `run_repl()` wrapper, creates ModeDispatcher, shows banner
- `command.rs` — CLI argument parsing
- `command_context.rs` — registers all nushell commands
- `config_files.rs` — loads env.nu, config.nu, login.nu
- `signals.rs` — Ctrl+C handler via ctrlc crate
- `terminal.rs` — Unix terminal/process group acquisition
- `logger.rs` — logging setup
- `ide.rs` — IDE/LSP features
- `experimental_options.rs` — nushell experimental feature flags
- `test_bins.rs` — nushell test utilities

**Shannon-specific (engines and dispatch):**

- `lib.rs` — library exports for the shannonshell crate
- `dispatcher.rs` — `ShannonDispatcher` implementing `ModeDispatcher`
- `bash_process.rs` — `BashProcess` (persistent bash subprocess with sentinel
  protocol)
- `shell_engine.rs` — `ShellEngine` trait
- `shell.rs` — `ShellState` (env, cwd, exit code)
- `executor.rs` — bash `export -p` output parsing

### How command execution works

**Nu mode (default):**

1. User types a command
2. Nushell's `loop_iteration()` reads input via reedline
3. `$env.SHANNON_MODE` is "nu" — falls through to nushell's parser/evaluator
4. Nushell handles everything: parsing, execution, output, env updates

**Bash mode (bash):**

1. User types a command
2. Nushell's `loop_iteration()` reads input via reedline
3. `$env.SHANNON_MODE` is "bash" — calls `ModeDispatcher::execute()`
4. Env vars converted to strings via `env_to_strings()`
5. `BashProcess` injects env vars and cwd, writes command + sentinel to bash
   stdin
6. Command output streams to stdout; sentinel block parsed for env, cwd, exit
   code
7. Result written back to nushell's Stack; REPL continues with updated state

### Environment propagation

When switching modes, all exported environment variables and cwd are preserved:

- **Nu → Bash:** `env_to_strings()` converts nushell typed values to strings
  (using `ENV_CONVERSIONS` `to_string` closures for PATH etc.). Injected into
  bash via `export` commands.
- **Bash → Nu:** `export -p` captures env vars after each command. String env
  vars written back to Stack via `add_env_var()`. Nushell's REPL automatically
  applies `from_string` conversions on the next iteration.

### Testing

Every new feature must include tests. No feature ships without test coverage.

- **Unit tests** go in each module as `#[cfg(test)] mod tests { ... }`.
- **Integration tests** go in `tests/`.
- Use `tempfile::TempDir` for tests that need filesystem fixtures.
- `cargo test` must pass before a feature is considered done.

### Key design decisions

- **Shannon IS nushell** — the binary copies nushell's startup code (~4,600
  lines) and adds mode dispatch. Shannon gets all nushell features for free: job
  control, plugins, multiline editing, completions, hooks, etc.
- **Trait injection** — `ModeDispatcher` trait defined in nu-cli, implemented by
  `ShannonDispatcher` in shannon. Nushell's fork has a ~30-line hook in
  `loop_iteration()` that dispatches to bash when `$env.SHANNON_MODE` is "bash".
  Shannon stays the primary binary; nushell stays a dependency.
- **Strings at the boundary** — env vars cross between shells as strings.
  `env_to_strings()` and `from_string` conversions handle typed values (PATH as
  list, etc.).
- **Monorepo** — nushell and reedline source merged directly into the repo with
  full git history. Path deps, no crates.io publishing needed. Upstream sync via
  `git subtree pull` (full history preserved). **NEVER use `--squash` with git
  subtree.** Full history must always be preserved for blame, log, and bisect to
  work across merged projects.
- **Vendor directory is for reference only** — vendored repos are for reading
  source code, not for building against.

## Modes

Shannon has two modes, cycled via Shift+Tab:

- **nu** — nushell (native, default)
- **bash** — bash-compatible via persistent bash subprocess

Mode is stored in `$env.SHANNON_MODE`. Shift+Tab sends `__shannon_switch` via
reedline's `ExecuteHostCommand`, which cycles the mode.

Each mode gets appropriate syntax highlighting:

- Nu mode: `NuHighlighter` (nushell's native highlighter)
- Bash mode: `BashHighlighter` (tree-sitter-bash, Tokyo Night colors)

## Config

Shannon uses standard config locations — no custom config directory:

- **Nushell config**: `~/.config/nushell/` (stock nushell location)
  - `env.nu` — nushell env setup
  - `config.nu` — nushell config (keybindings, colors, hooks, etc.)
  - `login.nu` — login shell config
  - `history.sqlite3` — SQLite command history
- **Bash config**: `~/.bash_profile` / `~/.bashrc` (standard bash locations)

The persistent bash subprocess starts with `bash --login`, which sources
`.bash_profile` (and conventionally `.bashrc`). This means "add this to your
.bashrc" instructions just work — no separate `env.sh` needed.

At startup, Shannon captures bash's post-login env vars and injects them into
nushell's Stack, so PATH, nvm, homebrew, etc. are available in both modes.

Shannon adds no custom configuration beyond what nushell provides. All shell
settings (keybindings, colors, hooks, completions) are configured via nushell's
`config.nu`.

## Issues and Experiments

Every significant piece of work gets an issue in `issues/`. Issues describe the
problem, provide background, and propose solutions. Experiments are the
incremental steps that solve the problem.

### Issue Structure

Each issue is a **folder**. The `README.md` is the issue **spine** (frontmatter,
goal, background, analysis, an ordered index of experiments, and the final
conclusion). **Every experiment is its own numbered file** in the same folder —
the README never contains experiment bodies, only links to them.

```
issues/0041-upgrade-nushell/
├── README.md                 ← spine: frontmatter, goal, background,
│                                the ordered Experiments index, conclusion
├── 01-sync-upstream.md       ← Experiment 1 (full body in its own file)
├── 02-fix-api-drift.md       ← Experiment 2
└── 03-...                    ← one file per experiment, in sequence
```

The folder name is `{NNNN}-{slug}`. The number is zero-padded to 4 digits and
globally sequential. The slug is lowercase, hyphenated, and describes the topic.

**Why one file per experiment:** it keeps experiments ordered and easy to read,
access, and organize (up to ~100 per issue with clean `NN-` filenames), and
critically, it makes experiments easy to automate: each experiment is a discrete
file created and tracked from the README, rather than ever-growing edits to one
monolithic document.

The full index of all issues is at `issues/README.md`. Regenerate it with:

```sh
scripts/build-issues-index.sh
```

#### Frontmatter

Every `README.md` starts with TOML frontmatter:

```
+++
status = "open"
opened = "2026-03-21"
+++
```

Or for closed issues:

```
+++
status = "closed"
opened = "2026-03-21"
closed = "2026-03-22"
+++
```

Issues may add their own TOML frontmatter keys — to `README.md`, experiment
files, or other issue docs — for issue-specific metadata such as per-experiment
agent provenance, as long as:

- the reserved workflow keys are preserved: `README.md` always carries `status`
  and `opened` (plus `closed` when closed), unchanged in name and meaning;
- additive keys are valid TOML between the `+++` delimiters and do not
  contradict the reserved keys or the index tooling —
  `scripts/build-issues-index.sh` reads only the reserved README keys and
  ignores the rest;
- the issue documents its own added schema in its `README.md`.

#### README.md structure

After the frontmatter, a new issue's `README.md` has these sections:

1. **Title** (H1) — `# Issue {N}: {descriptive title}`
2. **Goal** — One or two sentences describing the desired outcome.
3. **Background** — Context, prior work, constraints.
4. **Architecture** / **Analysis** / **Proposed Solutions** — Technical details.

A new issue's README has **no experiments listed yet**.

As experiments are created, the README grows an **`## Experiments`** section: an
ordered list linking to each experiment file, one per line, with a one-line
status. The README holds the links and statuses only — never the experiment
bodies. Example:

```markdown
## Experiments

- [Experiment 1: Sync Nushell upstream](01-sync-upstream.md) — **Pass**
- [Experiment 2: Fix Shannon API drift](02-fix-api-drift.md) — **Partial**
- [Experiment 3: Update release packaging](03-update-release-packaging.md) —
  **Designed**
```

Keep each status to one of: `Designed`, `In progress`, `Pass`, `Partial`,
`Fail`. Update the line when the experiment's result is recorded, so the README
doubles as an at-a-glance progress tracker.

When the issue is solved or abandoned, add the **`## Conclusion`** section to
the README (see "Closing an Issue").

#### Experiment files

Each experiment lives in its **own file** `NN-{slug}.md` in the issue folder,
where `NN` is a zero-padded two-digit number in creation order (`01`, `02`, …,
up to `99`). The slug is lowercase-hyphenated and describes the experiment.

An experiment file may begin with an optional TOML frontmatter block
(`+++ … +++`) before its H1 title — for issue-specific metadata such as agent
provenance. Experiment frontmatter is optional and must not replace the required
H1 title and H2 sections below it.

Each experiment file contains:

1. **Title** (H1) — `# Experiment {N}: {descriptive title}`
2. **Description** — What and why.
3. **Changes** — Specific code changes, listed by file.
4. **Verification** — How to test. Concrete steps and pass/fail criteria.
5. **Result** and **Conclusion** — added after the experiment runs (see
   "Recording results").

Keep each file focused; if one grows past ~1000 lines, that is a sign the
experiment is too big and should be split into the next numbered experiment.

### Multiple Open Issues

Multiple issues can be open at the same time. This allows interleaving work — a
large issue can stay open while smaller issues are opened and closed alongside
it.

### Experiments

#### When to create an experiment

Only after the issue's requirements are clear. Each experiment is designed,
implemented, and concluded before the next one is designed.

**Never list experiments upfront.** The outcome of each experiment informs what
comes next.

#### Experiment structure

Each experiment is its own file `NN-{slug}.md` (see "Experiment files" above),
and is added as a new link in the README's `## Experiments` index the moment it
is created. Inside the file, use an H1 title and H2 sections:

1. **Title** (H1) — `# Experiment {N}: {descriptive title}`
2. **Description** (H2) — What and why.
3. **Changes** (H2) — Specific code changes, listed by file.
4. **Verification** (H2) — How to test. Concrete steps and pass/fail criteria.
5. **Result** / **Conclusion** (H2) — added after it runs.

#### One at a time

Design and implement one experiment at a time. The result of Experiment 1
directly informs what Experiment 2 should be.

#### AI review gate

Every experiment must be reviewed by another AI agent before moving to the next
stage.

1. **Design review before implementation**
   - After writing the experiment design, ask another AI agent to review it.
   - Fix all real issues found by the review.
   - Record the review result in the experiment file.
   - Do not implement the experiment until the reviewing agent approves the
     design.

2. **Result review before the next experiment**
   - After implementation, verification, and result recording, ask another AI
     agent to review the completed experiment and result.
   - Fix all real issues found by the review.
   - Record the completion-review result in the experiment file.
   - Do not design or implement the next experiment until the reviewing agent
     approves the completed output.

The reviewing agent may be Codex, Claude, or another explicitly requested agent,
but it must be separate from the implementation pass.

Adversarial reviewers are allowed up to **15 minutes** to complete a review.
After spawning a reviewer, do not interrupt it, demand a bounded verdict, close
it, or proceed around it before that time has elapsed unless the user explicitly
asks you to stop or change direction. If the reviewer finishes earlier, use its
verdict normally.

#### Experiment commits

Every experiment has two required commit points:

1. **Plan commit** — after the experiment design is written, reviewed, fixed,
   approved, and linked from the issue README, commit the experiment plan before
   implementation begins.
2. **Result commit** — after implementation, verification, result recording,
   completion review, and any required fixes, commit the experiment result
   before designing the next experiment.

These commits must be separate. Do not combine an experiment plan and its result
in the same commit, and do not start the next experiment before the previous
experiment's result commit exists.

#### Recording results

After testing, append the result **inside the experiment's own file**, below
Verification:

```markdown
## Result

**Result:** Pass / Partial / Fail

{description}

## Conclusion

{what we learned, what to do next}
```

Then update that experiment's status on its line in the README's
`## Experiments` index (`Designed` → `Pass`/`Partial`/`Fail`). All three
outcomes are valuable — failed experiments eliminate dead ends.

### Closing an Issue

When the issue is solved or abandoned, add a `## Conclusion` section to the
**`README.md`** (after the `## Experiments` index), summarizing what was learned
and the outcome. Update the frontmatter to `status = "closed"` with a `closed`
date. Regenerate the index:

```sh
scripts/build-issues-index.sh
```

### Immutability

Closed issues are historical records. They are **immutable** and must NEVER be
modified. History stays as it was written.

The one-file-per-experiment structure applies to **issues created from now on**.
Earlier issues that recorded all experiments inline in a single `README.md` keep
their original form as historical records — do not retrofit them.

### Process Summary

1. **Create the issue** — `issues/{NNNN}-{slug}/README.md` with frontmatter,
   goal, background, analysis. No experiments yet.
2. **Design Experiment 1** — Create `01-{slug}.md` with the experiment body, and
   add a link to it under `## Experiments` in the README (status `Designed`).
3. **Review and commit the plan** — Get another AI agent to approve the design,
   fix real findings, record the review result, and commit the experiment plan.
4. **Implement Experiment 1** — Write the code.
5. **Record the result** — Append `## Result` / `## Conclusion` inside
   `01-{slug}.md`, and update its status on the README index line.
6. **Review and commit the result** — Get another AI agent to approve the
   completed output, fix real findings, record the completion review, and commit
   the experiment result.
7. **Repeat** — Create `02-{slug}.md` for the next experiment (the prior result
   informs it), link it from the README, and continue until the goal is met.
8. **Close the issue** — Write the `## Conclusion` in the README, update
   frontmatter, rebuild the index.

## Remember

NEVER change code unless explicitly asked. NEVER make unrequested changes.
Always do EXACTLY what your user asks — no more, no less.
