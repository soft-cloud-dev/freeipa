# FreeIPA SoftCloud

Welcome to **FreeIPA SoftCloud**, the enterprise Identity, Policy, and Audit (IPA) research and computational publishing platform built on **Jupyter Book 2** and hosted on **GitHub Pages** at [freeipa.softcloud.dev](https://freeipa.softcloud.dev/).

FreeIPA SoftCloud serves as an open architecture, executable reference, and operational playbook for centralized identity management, Kerberos authentication, automated PKI certificate lifecycle, dynamic DNS, and fine-grained host/role access controls across the Software Cloud ecosystem.

---

## Key Pillars

- **Unified Identity & Policy Plane**: Integrates 389 Directory Server (LDAP), MIT Kerberos KDC, Dogtag Certificate System (PKI), BIND DNS with Dynamic Lookup Zones (DLZ), and SSSD into a single authoritative identity fabric.
- **Cryptographic Reproducibility**: All dependencies are locked to exact distributions with SHA-256 integrity hashes (`--require-hashes`, `--only-binary=:all:`), eliminating dependency drift.
- **Hermetic Execution**: Computational runs target canonical **CPython 3.12.14 on Linux x86-64** with explicit virtual environment isolation.
- **Strict Profile Conformance**: Every publication artifact undergoes automated preflight verification (`tools/check_profile.py`) before build and deployment.
- **Continuous Delivery**: Fully automated CI/CD via GitHub Actions builds and verifies documentation, identity architectures, and notebooks upon every commit.

---

## Documentation Roadmap

- [**Introduction**](introduction.md): Background, platform philosophy, and operational scope for identity management.
- [**Architecture & Memorandum Reference**](architecture.md): Full technical specification, repository layout, verbatim scaffold files, FreeIPA subsystem architecture, and verification toolchain.
- [**Computational Verification Notebook**](notebooks/experiment.ipynb): Runtime validation notebook executing in the `jb2-python` kernel.
