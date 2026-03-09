---

name: github-readme-bilingual

description: Use when writing or refactoring GitHub README files, especially bilingual English/Chinese README pairs that need a centered title area, centered badges, and language-switch links between README.md and README_CN.md.

---

  

# GitHub README Bilingual Writer

  

Write production-ready GitHub README files in English and Chinese with a consistent, centered header block.

  

## When to use

  

Use this skill when the user asks to:

  

- create or rewrite project README files

- output separate English and Chinese README files

- add a centered title + badges + language switch links

- make README content align with real project metadata

  

## Output convention

  

Prefer two files:

  

- `README.md` for English (default entry on GitHub)

- `README_CN.md` for Chinese

  

Both files must include language switch links to each other.

  

## Required header pattern

  

Put this block at the very top of both files:

  

```html

<div align="center">

  

# {Project Name}

  

[![License](https://img.shields.io/badge/License-{LICENSE_LABEL}-lightgrey.svg)]({LICENSE_LINK})

[![Language](https://img.shields.io/badge/language-Go-00ADD8.svg)](https://go.dev/)

[![Go Version](https://img.shields.io/badge/go-{GO_VERSION}-00ADD8.svg)](https://go.dev/doc/devel/release)

  

[**English**](./README.md) | [**中文**](./README_CN.md)

  

</div>

  

---

```

  

Notes:

  

- Keep the title area centered with `<div align="center">`.

- Keep badges on centered lines inside the same block.

- Keep language switch links inside the same centered block.

- Use relative links exactly as `./README.md` and `./README_CN.md` unless the repo requires different names.

  

## Badge rules (accuracy first)

  

Only include badges that are true for the current repository.

  

1. License badge:

- If a `LICENSE` file exists, use the real license label and a valid link.

- If no `LICENSE` file exists, use `License-Unspecified` and leave link empty `()`.

  

2. Build/CI badge:

- Add a build status badge only when CI workflow exists and is actually configured.

- Do not claim `build passing` without real CI.

  

3. Tech badges:

- Include language/runtime/version badges that are verifiable from files like `go.mod`, `package.json`, or `pyproject.toml`.

  

## README body checklist

  

After the header block, include these sections in both languages:

  

1. Project overview

2. Features

3. Project structure tree

4. Requirements

5. Quick start

6. How it works / core flow

7. Minimal example (if applicable)

8. License note

  

Keep English and Chinese structure aligned for easy maintenance.

  

## Working procedure

  

1. Inspect repository facts first (`LICENSE`, runtime/tool versions, CI workflow files, module/dependency files).

2. Draft English and Chinese files with identical section structure.

3. Verify all links and badges.

4. Avoid fake or placeholder status claims.

5. If renaming files, update all cross-links.

  

## Quality guardrails

  

- Use concise, concrete wording.

- Match terminology across both languages.

- Avoid marketing fluff.

- Do not include unverifiable badges.

- Ensure header renders correctly on GitHub.