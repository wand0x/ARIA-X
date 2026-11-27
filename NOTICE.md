# NOTICE — attribution, upstream licenses, and compliance

Copyright (C) 2026 01NGVI-QINGYI

ARIA-X (Autonomous Reasoning & Investigation Agent, eXtended) is an independent implementation whose design fuses three upstream open-source projects.
**ARIA-X as a whole is released under the GNU Affero General Public License v3.0 (AGPL-3.0-or-later).** The full license text is in
[LICENSE](LICENSE).

## 1. Why ARIA-X uses AGPL-3.0

Two of ARIA-X's main sources of ideas are released under AGPL-3.0:

| Upstream | License | Role in ARIA-X |
| --- | --- | --- |
| [Cairn](https://github.com/oritera/Cairn) (author: [oritera](https://github.com/oritera)) | **AGPL-3.0** | Blackboard architecture: Fact / Intent / Hint, and the reason→explore OODA loop |
| ARTEX (original author: [Autumn-27](https://github.com/Autumn-27); the original repository `Autumn-27/ARTEX` was deleted by its author) | **AGPL-3.0** | Dual graph (exploration graph + shared asset graph), planner as the sole intent generator, cross-wakeup todo chain, cross-worker process traces, approval gate |
| [PentAGI](https://github.com/vxcontrol/pentagi) (team: [VXControl](https://github.com/vxcontrol)) | **MIT** | Specialist team (researcher/coder/adviser), provider abstraction, sandboxed terminal, layered memory |

MIT is compatible with AGPL-3.0 (MIT code may be included in an AGPL project). ARIA-X deliberately chooses the same
**AGPL-3.0-or-later** as the two AGPL upstreams, both to keep the licenses consistent and to cover any derivative boundary that future code-level reuse might create. Therefore:

* Releasing, modifying, or deploying ARIA-X as a network service must follow AGPL-3.0 section 13: if you offer the program to users over a network,
  you must offer those users the corresponding source (for example, a "Source" link at the bottom of the Web console).
* Any binary or container image that distributes ARIA-X must include [LICENSE](LICENSE) and this NOTICE.

## 2. Code-attribution facts

- **ARIA-X does not copy source code from the three projects above.** The fusion is at the architecture and protocol layer: data structures, state machines,
  prompt contracts, tool boundaries, and scheduling policy are reimplemented for ARIA-X's own needs (Python standard library).
- If any upstream snippet is brought in later, you must:
  1. name the source and the original license in that file's header;
  2. stay AGPL-3.0-compatible;
  3. update this NOTICE and [THIRD_PARTY.md](THIRD_PARTY.md).
- Upstream names, logos, and trademarks belong to their respective authors; this repository uses the names only in attribution statements.

## 3. Where the upstream license texts are

This repository does not embed the upstream source trees, so it does not repeat their full license texts, but it keeps these official locations:

| Upstream | Where the license text is |
| --- | --- |
| Cairn | [oritera/Cairn — LICENSE](https://github.com/oritera/Cairn/blob/main/LICENSE) |
| ARTEX | The original repository was deleted, so there is no official LICENSE link to cite; ARIA-X did not copy its code |
| PentAGI | [vxcontrol/pentagi — LICENSE](https://github.com/vxcontrol/pentagi/blob/main/LICENSE) (MIT) |

The full MIT license text and copyright notice for PentAGI are copied in [THIRD_PARTY.md](THIRD_PARTY.md), to meet MIT's
requirement to "retain the copyright and permission notice".

## 4. No warranty, and lawful use

ARIA-X is provided **"AS IS", with no warranty of any kind**, under AGPL-3.0 sections 15–17. ARIA-X is a security-testing tool:
**use it only on systems you have written authorization to test, or that you otherwise have a legal right to test**. See the lawful-use section of [docs/SECURITY.md](docs/SECURITY.md).
The authors and contributors are not liable for misuse or for damage that results from it.

## 5. Contact and contributions

- Submitting a contribution means you agree to license that contribution under AGPL-3.0.
- For security issues, follow the disclosure process in [docs/SECURITY.md](docs/SECURITY.md). Do not post exploitable details in public.
