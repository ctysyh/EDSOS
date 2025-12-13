<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# Contributing to This Project

Thank you for your interest in contributing! This project uses a modular structure with multiple licenses depending on the component. Please read this guide carefully before submitting changes.

## 1. Code of Conduct

All contributors are expected to adhere to professional and respectful collaboration practices. While we do not enforce a formal CoC at this time, courtesy and technical rigor are essential.

## 2. License Awareness

This project uses **different licenses for different parts**. Your contributions must comply with the license of the directory you are modifying:

| Directory | License | Requirements |
|----------|--------|--------------|
| `dsl/` | [Apache-2.0](LICENSES/Apache-2.0.txt) | Permissive; suitable for interpreters, DSL specs, and libraries |
| `src/` | [LGPL-3.0-or-later](LICENSES/LGPL-3.0-or-later.txt) | Requires derivative works to remain under LGPL unless dynamically linked |
| `docs/` (except `Foundations/`) | [LGPL-3.0-or-later](LICENSES/LGPL-3.0-or-later.txt) | Includes architecture docs, guides, and index files like `DOCUMENTS_FOR_PROJECT` |
| `docs/Foundations/` | [CC-BY-4.0](LICENSES/CC-BY-4.0.txt) | Academic/theoretical content; attribution required |

## 3. File Licensing (REUSE Compliance)

We follow the [REUSE specification](https://reuse.software/) to ensure clear licensing.

### Every new file must include an SPDX-License-Identifier

- For **code files** (`.py`, `.c`, `.rs`, etc.):
  ```python
  # SPDX-License-Identifier: Apache-2.0
  ```
- For **Markdown/docs**:
  ```markdown
  <!-- SPDX-License-Identifier: LGPL-3.0-or-later -->
  ```
- For **plain text files**:
  ```
  SPDX-License-Identifier: CC-BY-4.0
  ```

Place this identifier at the very top of the file (first line or within first comment block).

## 4. Pull Request Guidelines

- Clearly state which component(s) your change affects.
- If adding new files, ensure correct SPDX headers are present.
- Avoid mixing changes across license domains (e.g., don’t modify `dsl/` and `src/` in the same PR unless necessary).
- Update `docs/DOCUMENTS_FOR_PROJECT` if you add/remove documented files in `docs/`.

## 5. Verification

You can check license compliance locally:

```bash
pip install reuse
reuse lint
```

All files must pass this check before merging.

## 6. Questions?

If you’re unsure which license applies to your contribution, please open an issue or ask maintainers before submitting a pull request.

Thank you for helping make this project better!