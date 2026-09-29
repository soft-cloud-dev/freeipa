# Architecture & Memorandum Reference

FreeIPA SoftCloud computational pipeline, identity subsystem architecture, repository scaffold, and build environment specifications.

---

## 1. Overview and Design Principles

The **FreeIPA SoftCloud** platform is engineered around strict computational determinism, cryptographic reproducibility, and enterprise identity security. Built upon **Jupyter Book 2 (JB2)** and the **MyST Markdown** ecosystem, every component—from environment dependency resolution to identity service specifications and static HTML artifact generation—is verified before deployment.

### Core Architectural Tenets

1. **Unified Identity & Policy Fabric**: Integrates LDAP directory services (389 Directory Server), Kerberos authentication (MIT KDC), Public Key Infrastructure (Dogtag PKI), dynamic DNS (BIND with DLZ), and client security services (SSSD) into an auditable and reproducible identity plane.
2. **Deterministic Dependency Locking**: All runtime Python distributions and their transitive dependencies are pinned with exact versions and cryptographic SHA-256 digests (`--require-hashes`, `--only-binary=:all:`).
3. **Canonical Execution Environment**: Builds and notebook executions target **CPython 3.12.14 on Linux x86-64**, isolating execution state and preventing architecture drift.
4. **Explicit Profile Validation**: A standalone preflight validator (`tools/check_profile.py`) enforces schema integrity, kernelspec registration, TOC exhaustiveness, and execution control compliance prior to artifact compilation.
5. **Reproducible Continuous Delivery**: GitHub Actions runs the verified pipeline in an isolated runner, builds the static publication artifact under `--strict` mode, and deploys to [freeipa.softcloud.dev](https://freeipa.softcloud.dev/).

---

## 2. FreeIPA Identity Architecture & Subsystems

FreeIPA coordinates multiple foundational identity services under a coherent management, policy, and replication layer:

```text
+-----------------------------------------------------------------------+
|                       FreeIPA Management Layer                        |
|             (Web UI, CLI Framework, JSON-RPC / REST API)             |
+-----------------------------------------------------------------------+
        |                     |                    |                |
        v                     v                    v                v
+---------------+     +---------------+    +---------------+  +---------------+
| 389 Directory |     |  MIT Kerberos |    | Dogtag PKI    |  |  BIND 9 DNS   |
| Server (LDAP) |<--->|      KDC      |    |  (Certmonger) |  |   with DLZ    |
| (Multi-Master)|     | (Single Sign) |    | (X.509 CA)    |  | (SRV Records) |
+---------------+     +---------------+    +---------------+  +---------------+
        ^                                                           ^
        |                                                           |
+-----------------------------------------------------------------------+
|               System Security Services Daemon (SSSD)                  |
|          (PAM / NSS, Local Cache, Offline Auth, HBAC / Sudo)          |
+-----------------------------------------------------------------------+
```

### 2.1. Directory Services (389 Directory Server)
- Provides the high-performance LDAPv3 data store for users, groups, hosts, services, netgroups, and policy objects.
- Multi-master replication ensures high availability and partition tolerance across FreeIPA replica nodes.
- Schema extensions support Kerberos key material storage, automount maps, and custom object classes.

### 2.2. Authentication (MIT Kerberos KDC)
- Acts as the authoritative Key Distribution Center for the `SOFTCLOUD.DEV` realm.
- Enables single sign-on (SSO) and mutual authentication using Kerberos tickets (TGT, Service Tickets).
- Manages service principals (`HTTP/`, `host/`, `ldap/`, `nfs/`) with automated keytab distribution.

### 2.3. Certificate Authority (Dogtag PKI & Certmonger)
- Built-in full-featured X.509 Certificate Authority managing the complete cryptographic lifecycle of cluster certificates.
- Automated certificate issuance, tracking, and renewal on hosts via `certmonger`.
- Supports user/host/service certificates, smart card authentication, and TLS termination.

### 2.4. Domain Name System (BIND 9 with DLZ)
- Dynamically discovers FreeIPA services using standard DNS SRV and TXT records (`_kerberos`, `_kpasswd`, `_ldap`, `_ipa-ca`).
- BIND Dynamic Lookup Zones (DLZ) store zone records directly in LDAP, enabling instant replication across replicas without zone transfers.

### 2.5. Client Integration & Policy Enforcement (SSSD)
- SSSD bridges client Linux/UNIX systems to FreeIPA using standard PAM and NSS interfaces.
- Advanced caching guarantees seamless offline login and resilient authentication during network partitions.
- Evaluates Host-Based Access Control (HBAC) and centralized Sudo policies directly at host login.

---

## 3. Repository Layout & Topology

The repository follows a clean, minimal directory topology designed for auditability:

```text
freeipa/
├── .github/
│   └── workflows/
│       └── deploy.yml            # Automated CI/CD deployment workflow
├── images/
│   ├── favicon.ico               # Multi-resolution browser icon
│   ├── favicon-16x16.png         # Small browser favicon
│   ├── favicon-32x32.png         # Standard desktop browser favicon
│   ├── favicon-192x192.png       # High-DPI web app icon
│   ├── favicon.png               # PNG favicon
│   ├── favicon.svg               # Vector favicon
│   ├── apple-touch-icon.png      # iOS home screen icon
│   ├── logo.svg                  # Primary horizontal brand lockup
│   ├── logo-dark.svg             # Contrast-adapted dark mode lockup
│   ├── logo-horizontal.svg       # Extended horizontal brand lockup
│   ├── logo-mark.svg             # Minimal icon mark
│   └── logo-mark-dark.svg        # Dark mode icon mark
├── notebooks/
│   └── experiment.ipynb          # Verified CPython 3.12.14 computational notebook
├── tools/
│   ├── build.sh                  # Hermetic publication build runner
│   ├── check_profile.py          # Profile compliance & integrity validator
│   └── lock_from_wheels.py       # Wheelhouse hash resolver & lockfile generator
├── .gitignore                    # Artifact & virtual environment exclusions
├── architecture.md               # Architecture reference & memorandum specifications
├── introduction.md               # Project background and operational scope
├── LICENSE                       # BSD 2-Clause License
├── myst.yml                      # Jupyter Book 2 project configuration
├── README.md                     # Platform overview and landing page
├── requirements.in               # Top-level dependency declarations
└── requirements.lock             # Cryptographically pinned wheel lockfile
```

---

## 4. Memorandum Scaffold Specifications

The following sections document the verbatim scaffold files and specifications defined in the repository memorandum.

### 4.1. `LICENSE`

The repository is distributed under the open-source **BSD 2-Clause License**:

```text
Copyright (c) 2026, FreeIPA SoftCloud Contributors
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

1. Redistributions of source code must retain the above copyright notice, this
   list of conditions and the following disclaimer.

2. Redistributions in binary form must reproduce the above copyright notice,
   this list of conditions and the following disclaimer in the documentation
   and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

### 4.2. `.gitignore`

Strict exclusion rules keep local cache artifacts, wheelhouses, and virtual environments outside source control:

```text
.venv/
.lock-venv/
.wheelhouse/
_build/
__pycache__/
.ipynb_checkpoints/
.DS_Store
```

### 4.3. `requirements.in`

Direct, unpinned top-level requirements defining the computational runtime tooling:

```text
jupyter-book==2.1.7
ipykernel==7.3.0
PyYAML==6.0.3
```

### 4.4. `myst.yml`

The configuration file governing book structure, metadata, branding, and table of contents:

```yaml
version: 1
project:
  title: FreeIPA SoftCloud
  description: Computational publication and research platform for FreeIPA Identity & Access Management on Software Cloud.
  github: https://github.com/soft-cloud-dev/freeipa
  toc:
    - file: README.md
    - file: introduction.md
    - file: architecture.md
    - title: Computational Notebooks
      children:
        - file: notebooks/experiment.ipynb
site:
  template: book-theme
  options:
    logo: images/logo.svg
    logo_dark: images/logo-dark.svg
    favicon: images/favicon.ico
```

### 4.5. Computational Notebook: `notebooks/experiment.ipynb`

An executable verification notebook executed by Jupyter Book during the build process to guarantee runtime environment health:

```json
{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Computational Verification\n",
    "FreeIPA SoftCloud environment runtime validation notebook."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "import sys\n",
    "print(\"Executable:\", sys.executable)\n",
    "print(\"Version:\", sys.version)\n",
    "assert sys.version_info[:3] == (3, 12, 14)"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python (JB2)",
   "language": "python",
   "name": "jb2-python"
  },
  "language_info": {
   "name": "python",
   "version": "3.12.14"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 5
}
```

---

## 5. Build & Maintenance Toolchain

### 5.1. Lockfile Generator: `tools/lock_from_wheels.py`

This maintenance utility resolves wheel archives in `.wheelhouse/`, extracts their normalized distribution metadata, computes their SHA-256 checksums, and produces a strictly pinned `requirements.lock`:

```python
import email
import hashlib
import re
import sys
import zipfile
from pathlib import Path

wheelhouse, output = map(Path, sys.argv[1:])
if output.exists():
    raise SystemExit(f"Refusing to overwrite {output}; review a new candidate")
wheels = sorted(wheelhouse.glob("*.whl"))
if not wheels:
    raise SystemExit("No wheels found")
if any(p.suffix != ".whl" for p in wheelhouse.iterdir()):
    raise SystemExit("Wheel directory contains unexpected entries")
records = {}
for wheel in wheels:
    with zipfile.ZipFile(wheel) as archive:
        names = [
            n for n in archive.namelist()
            if n.count("/") == 1 and n.endswith(".dist-info/METADATA")
        ]
        if len(names) != 1:
            raise SystemExit(f"Ambiguous metadata: {wheel.name}")
        metadata = email.message_from_bytes(archive.read(names[0]))
    name = re.sub(r"[-_.]+", "-", metadata["Name"]).lower()
    version = metadata["Version"]
    if name in records:
        raise SystemExit(f"Duplicate distribution: {name}")
    with wheel.open("rb") as stream:
        digest = hashlib.file_digest(stream, "sha256").hexdigest()
    records[name] = f"{name}=={version} --hash=sha256:{digest}"
with output.open("x", encoding="utf-8") as stream:
    stream.write("\n".join(records[n] for n in sorted(records)) + "\n")
print(f"Wrote {len(records)} pinned wheel requirements to {output}")
```

### 5.2. Profile Preflight Validator: `tools/check_profile.py`

The validation engine guarantees profile purity and strict adherence to the publication contract:

```python
import json
import os
import re
import sys
from importlib.metadata import version
from pathlib import Path

import yaml
from jupyter_client.kernelspec import KernelSpecManager

def require(condition, message):
    if not condition:
        raise SystemExit(message)

root = Path.cwd().resolve()
require(sys.implementation.name == "cpython", "CPython required")
require(sys.version_info[:3] == (3, 12, 14), "Python 3.12.14 required")
require(sys.prefix != sys.base_prefix, "Activate the project virtualenv")
for package, expected in {
    "jupyter-book": "2.1.7", "ipykernel": "7.3.0", "PyYAML": "6.0.3"
}.items():
    require(version(package) == expected, f"Wrong version: {package}")
require(version("jupyter-server"), "Jupyter Server required")
for name in ("JUPYTER_BASE_URL", "JUPYTER_TOKEN", "READTHEDOCS_CANONICAL_URL"):
    require(not os.environ.get(name), f"Unexpected environment override: {name}")
require("BASE_URL" in os.environ, "Set BASE_URL explicitly; empty is valid")
base = os.environ["BASE_URL"]
require(base == "" or (base.startswith("/") and not base.startswith("//")
        and not base.endswith("/") and not any(c in base for c in "?#\n\r")),
        "BASE_URL must be empty or a path prefix such as /docs")

kernel_dir = Path(sys.prefix) / "share/jupyter/kernels/jb2-python"
spec = json.loads((kernel_dir / "kernel.json").read_text())
require(Path(spec["argv"][0]).absolute() == Path(sys.executable).absolute(),
        "Kernel does not launch this environment's Python")
resolved = KernelSpecManager().get_kernel_spec("jb2-python")
require(Path(resolved.resource_dir).resolve() == kernel_dir.resolve(),
        "Another kernelspec overrides the project kernel")

def local_file(value):
    require(isinstance(value, str), "Source path must be a string")
    path = root / value
    require(not Path(value).is_absolute(), f"Absolute source path: {value}")
    require(path.resolve().is_relative_to(root), f"External path: {value}")
    require(path.is_file(), f"Missing source or asset: {value}")
    return path

config = yaml.safe_load(Path("myst.yml").read_text())
require(isinstance(config, dict) and config.get("version") == 1,
        "Expected myst.yml version: 1")
require("extends" not in config, "Configuration inheritance needs a profile extension")
project = config.get("project", {})
require(not project.get("plugins"), "Custom plugins need a profile extension")
require(not project.get("exclude"), "Project exclusions are not accepted")
toc = project.get("toc")
require(isinstance(toc, list) and toc, "Explicit nonempty project.toc required")
site = config.get("site", {})
require(site.get("template") == "book-theme", "Expected book-theme selector")
local_file(site.get("options", {}).get("logo"))
files = []

def walk(entries):
    require(isinstance(entries, list), "TOC children must be a list")
    for entry in entries:
        require(isinstance(entry, dict), "TOC entry must be a mapping")
        require(set(entry) <= {"file", "title", "children"},
                "Unsupported TOC entry; use file/title/children")
        require("file" in entry or entry.get("children"), "Empty TOC entry")
        if "file" in entry:
            files.append(local_file(entry["file"]))
        if "children" in entry:
            walk(entry["children"])

walk(toc)
require(len({p.resolve() for p in files}) == len(files), "Duplicate TOC file")
for path in Path("notebooks").rglob("*.ipynb"):
    if ".ipynb_checkpoints" not in path.parts:
        require(path.resolve() in {p.resolve() for p in files},
                f"Notebook absent from TOC: {path}")

banned = {"skip-execution", "raises-exception"}

def check_controls(value, label):
    if isinstance(value, dict):
        execute = value.get("execute")
        if isinstance(execute, dict):
            require(execute.get("skip") in (None, False), f"Execution skipped: {label}")
        tags = value.get("tags", [])
        if isinstance(tags, (list, str)):
            require(not any(tag in tags for tag in banned), f"Forbidden tag: {label}")
        for child in value.values():
            check_controls(child, label)
    elif isinstance(value, list):
        for child in value:
            check_controls(child, label)

def check_kernel(metadata, label):
    expected = {"name": "jb2-python", "display_name": "Python (JB2)", "language": "python"}
    actual = metadata.get("kernelspec", {})
    require(all(actual.get(k) == v for k, v in expected.items()),
            f"Wrong or absent kernelspec: {label}")

check_controls(config, "myst.yml")
for path in files:
    if path.suffix == ".ipynb":
        notebook = json.loads(path.read_text())
        check_controls(notebook.get("metadata", {}), path)
        cells = notebook.get("cells", [])
        for cell in cells:
            check_controls(cell.get("metadata", {}), path)
        computational = any(c.get("cell_type") == "code" for c in cells)
        computational = computational or any(
            "{eval}" in "".join(c.get("source", []))
            or "{code-cell}" in "".join(c.get("source", []))
            for c in cells if c.get("cell_type") == "markdown")
        if computational:
            check_kernel(notebook.get("metadata", {}), path)
    elif path.suffix == ".md":
        text = path.read_text()
        front = re.match(r"\A---\r?\n(.*?)\r?\n---(?:\r?\n|\Z)", text, re.S)
        metadata = yaml.safe_load(front.group(1)) or {} if front else {}
        require(isinstance(metadata, dict), f"Invalid frontmatter: {path}")
        check_controls(metadata, path)
        if "{code-cell}" in text or "{eval}" in text:
            check_kernel(metadata, path)
            require(not any(tag in text for tag in banned),
                    f"Review executable Markdown execution controls: {path}")
require(not Path("_config.yml").exists() and not Path("_toc.yml").exists(),
        "Remove legacy configuration from this profile")
print(f"Preflight passed for {len(files)} TOC files")
```

### 5.3. Hermetic Build Runner: `tools/build.sh`

The shell script orchestrating preflight validation, cache purging, and strict compilation:

```sh
#!/bin/sh
set -eu
export JB_ALLOW_NODEENV=0
command -v node
command -v npm
node --version
npm --version
python --version
python -m pip --version
python tools/check_profile.py
jupyter book --version
jupyter book clean --all --execute -y
jupyter book build --html --strict --execute
test -s _build/html/index.html
```

---

## 6. Continuous Deployment: `.github/workflows/deploy.yml`

The CI/CD pipeline deploys the publication to GitHub Pages on every push to `main`:

```yaml
name: Jupyter Book Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      pages: read
    timeout-minutes: 30
    defaults:
      run:
        shell: bash
    steps:
      - name: Checkout
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1
        with:
          persist-credentials: false

      - name: Configure Pages
        id: pages
        uses: actions/configure-pages@45bfe0192ca1faeb007ade9deae92b16b8254a0d

      - name: Configure Python
        uses: actions/setup-python@5fda3b95a4ea91299a34e894583c3862153e4b97
        with:
          python-version: '3.12.14'
          architecture: x64

      - name: Install locked environment
        run: |
          python -c 'import sys; assert sys.implementation.name == "cpython" and sys.version_info[:3] == (3, 12, 14), sys.version'
          python -m venv .venv
          .venv/bin/python -m pip --version
          .venv/bin/python -m pip install \
            --require-hashes \
            --only-binary=:all: \
            -r requirements.lock
          .venv/bin/python -m pip check
          .venv/bin/python -m ipykernel install \
            --sys-prefix \
            --name jb2-python \
            --display-name 'Python (JB2)'

      - name: Validate and build publication artifact
        env:
          BASE_URL: ${{ steps.pages.outputs.base_path }}
          JB_ALLOW_NODEENV: '0'
        run: |
          . .venv/bin/activate
          sh tools/build.sh

      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@fc324d3547104276b827a68afc52ff2a11cc49c9
        with:
          path: ./_build/html

  deploy:
    needs: build
    runs-on: ubuntu-24.04
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy accepted artifact
        id: deployment
        uses: actions/deploy-pages@368f82528645a54fb793d4d04e342629a3f51346
```

---

## 7. Deterministic Dependency Resolution

To guarantee binary compatibility with the Ubuntu CI runner, `requirements.lock` was generated inside a Linux/x86-64 container hosting CPython 3.12.14:

```sh
python3.12 -m venv .lock-venv
mkdir .wheelhouse
.lock-venv/bin/python -m pip download \
    --only-binary=:all: \
    --dest .wheelhouse \
    -r requirements.in
.lock-venv/bin/python tools/lock_from_wheels.py .wheelhouse requirements.lock
rm -rf .lock-venv .wheelhouse
```

The resulting lockfile pins all 77 binary distributions with SHA-256 hashes, ensuring that every build step is completely isolated and reproducible.
