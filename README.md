<!--
SPDX-FileCopyrightText: © 2025 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

# README

## Licensing

This project uses multiple licenses depending on the component:

- **Arxil Components** (`Arxil/` directory):  
  Licensed under the [Apache License, Version 2.0](LICENSES/Apache-2.0.txt).  
  This includes the DSL specification, interpreter, and runtime library.

- **EDSOS Core Software & Engineering Documentation** (`src/` and most of `docs/` directories, **except `docs/0-Foundations/`**):  
  Licensed under the [GNU Lesser General Public License v3.0 or later](LICENSES/LGPL-3.0-or-later.txt).

  Some files in these directories may include additional permissions (e.g., static linking exceptions) to facilitate integration. Such files are clearly marked with a custom SPDX identifier (e.g., `LGPL-3.0-or-later+static-linking-exception`) and the full license text is provided in the [`LICENSES/`](LICENSES/) directory.

- **EDSOS Theoretical Foundations** ( `docs/0-Foundations/` directories):
  Licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSES/CC-BY-4.0.txt).  
  These documents present academic-style theoretical models, formal analyses, and semantic definitions.  
  You are free to share and adapt this material, provided you give appropriate credit.

### Compliance & REUSE

This project follows the [REUSE licensing best practices](https://reuse.software/).  
Every source file contains an `SPDX-License-Identifier` tag indicating its license.  
You can verify compliance using the `reuse` tool:

```bash
pip install reuse
reuse lint
```

For contributions, please ensure your files include the appropriate license header.