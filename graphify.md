# Graphify

## Setup: at the start of every project

Set Graphify up when the project is created, right after `git init`, no matter how new or small the project is:

1. `graphify install --platform <tool>` (once per machine and AI tool; skip if the Graphify skill is already installed for that tool)
2. `graphify <tool> install` (in the project, for the project's AI tool)
3. `graphify hook install` (in the project, so the graph updates after every commit)
4. `/graphify` to build the first graph
5. Check: `graphify hook status`, `ls graphify-out`

`graphify-out/` is always in `.gitignore`. Hooks are not committed: after cloning a project on a new machine, run `graphify hook install` again.

## Same behavior on every machine and OS

Graphify writes machine-specific values into files that get committed. Fix them right after setup:

- After `graphify <tool> install`, replace every absolute path to the graphify executable in the generated hook config with the bare `graphify` command. The executable is then found through `PATH` on any machine and OS.
- After `graphify hook install`, remove the `graphify-out/graph.json` merge rule it adds to `.gitattributes` (`graphify-out/` is ignored, so the rule does nothing). Delete `.gitattributes` if nothing else is left in it.
- Re-running `graphify <tool> install` writes the absolute path again; apply the fix again.
- Committed Graphify files never contain absolute paths or user names.

## Usage

- If the project has `graphify-out/GRAPH_REPORT.md`, read it before exploring the codebase.
- When the graph exists, prefer `graphify query` / `graphify explain` over broad file searches.
- After big structural changes, suggest re-running `/graphify`.
