<!--
SPDX-FileCopyrightText: © 2025-2026 Bib Guake
SPDX-License-Identifier: LGPL-3.0-or-later
-->

README

[![status: early-development](https://img.shields.io/badge/status-early--development-yellow)](https://github.com/ctysyh/EDSOS)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache--2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![License: CC BY 4.0](https://img.shields.io/badge/License-CC--BY--4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![License: LGPL-3.0-or-later](https://img.shields.io/badge/License-LGPL--3.0--or--later-green.svg)](https://www.gnu.org/licenses/lgpl-3.0.html)


EDSOS — Explicit Distributed Single Operating System.

Table of Contents
- Project overview
- Current status
- Repository layout
- Roadmap
- Contact / Maintainers
- Licensing
- Compliance & REUSE


Project overview
----------------
EDSOS is a project that began as a set of documents and design notes. The goal of EDSOS is to provide an opensource full stack on high-performance distributed computing.


Current status
----------------
- Development stage: early — transitioning from design documents and prototypes to a maintainable codebase.
- Documentation: foundational docs exist; they need to be extended with API-level and how-to guides.


Repository layout
----------------
- Arbor-Strux/         — basic theory
- Arxil/               — language
- edsos/               — operating system
- LICENSES/            — project licenses
- notebooks/           — exploratory notebooks and drafts
- CONTRIBUTING.md      — contributing guides
- README.md            — this file
- System-Overview.md   — overview


Roadmap
--------------------
Short-term (next few sprints)
- Update the content of the language formalization document corresponding to the new language specification
- Establish the `.arxtype` generator based on.NET BCL and NativeAOT
- Create the first version of a simple compiler that only supports `'instruct'` and `'inst_scri'` based on the formal documentation
- Produce the first version of ArxilVM based on the process subsystem documentation of EDSOS


Contact / Maintainers
---------------------
- Maintainer: Bib Guake
- For questions: open an issue.


Licensing
----------------
This project uses multiple licenses depending on the component:

- Arbor Strux Theory ( `Arbor-Strux/` directories):
  Licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](LICENSES/CC-BY-4.0.txt).  
  These documents present academic-style theoretical models, formal analyses, and semantic definitions.  
  You are free to share and adapt this material, provided you give appropriate credit.

- Arxil Components (`Arxil/` directory):  
  Licensed under the [Apache License, Version 2.0](LICENSES/Apache-2.0.txt).  
  This includes the DSL specification, interpreter, and runtime library.

- EDSOS Core Software & Engineering Documentation (`edsos/src/` and `edsos/docs/` directories):  
  Licensed under the [GNU Lesser General Public License v3.0 or later](LICENSES/LGPL-3.0-or-later.txt).

  Some files in these directories may include additional permissions (e.g., static linking exceptions) to facilitate integration. Such files are clearly marked with a custom SPDX identifier (e.g., `LGPL-3.0-or-later+static-linking-exception`) and the full license text is provided in the [`LICENSES/`](LICENSES/) directory.


Compliance & REUSE
----------------
This project follows the [REUSE licensing best practices](https://reuse.software/).  
Every source file contains an `SPDX-License-Identifier` tag indicating its license.  
You can verify compliance using the `reuse` tool:

```bash
pip install reuse
reuse lint
```

For contributions, please ensure your files include the appropriate license header.