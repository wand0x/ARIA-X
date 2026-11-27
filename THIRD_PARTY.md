# THIRD_PARTY — third-party components, dependencies, and acknowledgements

## 1. Runtime dependencies

ARIA-X's runtime **uses only the Python 3.11+ standard library** (3.12 is the target). In `pyproject.toml`,
`dependencies = []`. There is therefore no third-party Python package to ship with the distribution, and no transitive
Python dependency for an SBOM.

| Category | Dependency | Notes |
| --- | --- | --- |
| Language runtime | CPython ≥ 3.11 | Standard library only: `http.server`, `sqlite3`, `threading`, `tomllib`, `unittest`, and others |
| Optional containers | Docker / Podman | Invoked only when `sandbox.backend = "docker"`, for the Kali sandbox |
| Optional external services | Any OpenAI-compatible API, or the Anthropic API | Called over HTTP; no SDK is bundled |

Development-time tools that are not shipped in the repository (`bash`, `curl`, `docker`) are not runtime dependencies.

## 2. Sources of ideas and attribution (not code dependencies)

ARIA-X fuses design concepts from three projects, but **does not copy their code**. The full compliance statement is in
[NOTICE.md](NOTICE.md).

| Project | License | Copyright holder | Concepts ARIA-X borrows |
| --- | --- | --- | --- |
| Cairn | AGPL-3.0 | oritera / Cairn contributors | Fact/Intent/Hint blackboard, reason→explore OODA, role-less workers |
| ARTEX | AGPL-3.0 | Autumn-27 / ARTEX contributors | Dual-graph separation, planner as the sole intent generator, shared todo chain, cross-worker traces, approval gate |
| PentAGI | MIT | PentAGI Development Team (2025) | Specialist team, provider abstraction, sandboxed terminal, layered memory |

### Full text of the PentAGI MIT license

```
MIT License

Copyright (c) 2025 PentAGI Development Team

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## 3. Fonts, icons, and frontend assets

- `assets/logo.svg` and `assets/architecture.svg` are original ARIA-X SVGs. They use the system font stack
  (`ui-sans-serif, system-ui, ...`) and do not embed third-party font files.
- The Web console (`src/aria_x/web/`) loads no CDN assets and uses no third-party JS framework,
  so it runs completely offline.
- The example target `examples/demo_target.py` uses the standard-library `http.server` and contains no third-party code.

## 4. Data and trademarks

- The repository contains no real customer data, credentials, or scan results.
- Names such as "Cairn", "ARTEX", and "PentAGI" are used only for attribution. The related trademarks belong to their respective owners.
