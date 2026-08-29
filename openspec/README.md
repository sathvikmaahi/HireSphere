# HireSphere OpenSpec

Behavioral specifications using the [OpenSpec](https://github.com/Fission-AI/OpenSpec) format: **Purpose**, **Requirements** (SHALL/MUST), and **Scenarios** (GIVEN/WHEN/THEN).

## Start here

**[project.md](./project.md)** — project context, tech stack, repository layout, spec index, branch mapping, and links to planning docs.

## Using with OpenSpec CLI (optional)

```bash
npm install -g @fission-ai/openspec@latest
cd HireSphere
openspec init    # adds agent instructions if not present
openspec validate --strict
```

When implementing a feature branch, create a change under `openspec/changes/<change-name>/` with delta specs (`ADDED`/`MODIFIED`/`REMOVED`) and archive when done.
