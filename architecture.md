# Architecture & Operational Adoption Plan

FreeIPA 4.13.4 on FreeBSD 15.1 with Authoritative BIND 9.20.29 (`FBSD-IPA-DNS-MEMO-1`) Architecture Specification, Subsystem Topology, Repository Scaffold, and Organizational Adoption Plan.

---

## 1. Executive Summary and Architectural Principles

This document establishes the official technical architecture and organizational adoption plan for deploying **FreeIPA 4.13.4** (`net/freeipa-server`) alongside authoritative, non-recursive **ISC BIND 9.20.29** (`dns/bind920`) on **FreeBSD 15.1-RELEASE**, codified under standard identifier **`FBSD-IPA-DNS-MEMO-1`**.

### 1.1. Core Architectural Invariants

1. **Strict Privilege & Functional Separation**: FreeIPA is the authoritative master for identity (389 Directory Server), Kerberos authentication (MIT KDC), PKI certificates (Dogtag CA), and client policy (SSSD). External BIND 9 is the sole authority for DNS.
2. **No Integrated DNS on FreeBSD**: Because `bind-dyndb-ldap` is unavailable on FreeBSD, FreeIPA integrated DNS (`--setup-dns`) is strictly prohibited. FreeIPA interacts with BIND solely via authenticated RFC 2136 DNS UPDATE using a scoped TSIG key (`ipa-ddns`).
3. **Mail Namespace Boundary**: The identity update key (`ipa-ddns`) is cryptographically and logically isolated from the mail namespace (`MX`, apex `TXT`, `_dmarc`, `*._domainkey`, and `mail` address records).
4. **Hermetic Linkage & Port Governance**: All GSSAPI consumers (`security/cyrus-sasl2-gssapi`, `security/py-gssapi`) must link against Ports MIT Kerberos (`security/krb5`) resolving `/usr/local/lib/libgssapi_krb5.so`, never base Heimdal.
5. **Cryptographic & Operational Reproducibility**: Repository code, computational verification notebooks, and publishing toolchains conform to deterministic CPython 3.12.14 profiles and SHA-256 pinned wheelhouses.

---

## 2. Technical Standard Specification (`FBSD-IPA-DNS-MEMO-1`)

### 2.1. System Baseline & Metadata

| Specification Field | Baseline Parameter |
|:---|:---|
| **Document Identifier** | `FBSD-IPA-DNS-MEMO-1` (Rev 1.0) |
| **Operating System** | FreeBSD 15.1-RELEASE |
| **Ports Baseline Commit** | `07689140206dc74cd3a761d4b3983820b58172dc` |
| **FreeIPA Baseline** | FreeIPA 4.13.4 (`net/freeipa-server`, SHA-256: `b2f24d763875a9b5...`) |
| **BIND Baseline** | ISC BIND 9.20.29 (`dns/bind920`) |
| **Primary Nameserver / Host** | `ns1.example.com` / `ipa.example.com` (`192.0.2.10`) |
| **Secondary Nameserver** | `ns2.example.com` (`192.0.2.11`) |
| **Kerberos Realm / Domain** | `EXAMPLE.COM` / `example.com` |

### 2.2. Service Topology and Trust Boundaries

```text
                           Internet / Client Resolvers
                                        |
                                    .com TLD
                                        |
                      +-----------------+-----------------+
                      |                                   |
                      v                                   v
            +----------------------+            +----------------------+
            | ns1.example.com      |            | ns2.example.com      |
            | 192.0.2.10           |            | 192.0.2.11           |
            | FreeBSD 15.1         |            | FreeBSD 15.1         |
            | BIND 9.20.29         |            | BIND 9.20.29         |
            | Primary Authoritative|            | Secondary (Replica)  |
            +----------+-----------+            +-----------+----------+
                       |                                    ^
        RFC 2136       | Local Host                         |
        TSIG: ipa-ddns | (Scoped update-policy)             | TSIG AXFR/IXFR
        Explicit RRset v                                    | xfr-example-com
            +----------------------+                        |
            | ipa.example.com      |------------------------+
            | 192.0.2.10           |
            | FreeBSD 15.1         |
            | FreeIPA 4.13.4       |
            +----------------------+
```

### 2.3. Scoped BIND 9 Configuration

The primary nameserver (`ns1`) enforces an authoritative-only posture (`recursion no;`) and limits dynamic updates strictly to approved FreeIPA discovery records:

```named
include "/usr/local/etc/namedb/keys/ipa-ddns.key";
include "/usr/local/etc/namedb/keys/xfr-example-com.key";

options {
    directory "/usr/local/etc/namedb";
    pid-file "/var/run/named/pid";
    session-keyfile "/var/run/named/session.key";

    listen-on { 127.0.0.1; 192.0.2.10; };
    listen-on-v6 { none; };

    recursion no;
    allow-recursion { none; };
    allow-query { any; };
    allow-transfer { none; };

    version "DNS Server";
};

zone "example.com" {
    type primary;
    file "dynamic/example.com.zone";

    notify yes;

    also-notify {
        192.0.2.11 key xfr-example-com;
    };

    allow-transfer {
        key xfr-example-com;
    };

    update-policy {
        grant ipa-ddns name _kerberos.example.com.             TXT;
        grant ipa-ddns name _ldap._tcp.example.com.            SRV;
        grant ipa-ddns name _kerberos._tcp.example.com.        SRV;
        grant ipa-ddns name _kerberos._udp.example.com.        SRV;
        grant ipa-ddns name _kerberos-master._tcp.example.com. SRV;
        grant ipa-ddns name _kerberos-master._udp.example.com. SRV;
        grant ipa-ddns name _kpasswd._tcp.example.com.         SRV;
        grant ipa-ddns name _kpasswd._udp.example.com.         SRV;
        grant ipa-ddns name ipa-ca.example.com.                A AAAA;
    };
};
```

The secondary server (`ns2`) receives zone replication solely through `xfr-example-com.key` and never receives `ipa-ddns.key`.

---

## 3. Organizational Adoption Plan

The adoption of `FBSD-IPA-DNS-MEMO-1` is structured into six strictly gated implementation phases governed by organizational change control and acceptance testing.

```text
+-----------------------------------------------------------------------------------+
|                            ORGANIZATIONAL ADOPTION ROADMAP                        |
+-----------------------------------------------------------------------------------+
  Phase 0: Poudriere Build & GSSAPI Linkage Governance
      │    - Configure Poudriere with GSSAPI_MIT=on, GSSAPI_BASE=off
      │    - Build & audit FreeIPA 4.13.4 and BIND 9.20.29 packages
      ▼
  Phase 1: Authoritative Nameserver Infrastructure Deployment
      │    - Provision ns1 (primary) and isolated ns2 (secondary) on FreeBSD 15.1
      │    - Generate discrete TSIG keys (ipa-ddns, xfr-example-com)
      │    - Seed bootstrap zone & verify recursion refusal (AC-05, AC-06, AC-07)
      ▼
  Phase 2: FreeIPA Host Provisioning & Base Installation
      │    - Validate FQDN resolution & base ntpd operational state
      │    - Execute ipa-server-install --no-ntp (without --setup-dns)
      │    - Enable gssproxy & freeipa-server rc.d scripts (AC-04, AC-08, AC-09)
      ▼
  Phase 3: Automated Discovery Record Publication
      │    - Extract discovery RRsets via `ipa dns-update-system-records --dry-run`
      │    - Apply bound RFC 2136 dynamic updates using ipa-ddns.key
      │    - Verify SRV/TXT records across ns1 and ns2 (AC-10, AC-11, AC-13, AC-14)
      ▼
  Phase 4: Security Boundary & Mail Namespace Containment
      │    - Negative testing: verify ipa-ddns is rejected on MX, SPF, DKIM, DMARC
      │    - Enforce mail namespace isolation (AC-12, AC-18)
      ▼
  Phase 5: Client Auto-Discovery & Resilience Validation
      │    - Validate SSSD client join & KDC auto-discovery without static IPs
      │    - Simulate ns1 outage to verify ns2 secondary autonomy (AC-15, AC-16, AC-17)
      ▼
  Phase 6: Production Cutover, Runbook Codification, & Publication
           - Replace RFC 2606 placeholders with operational domain & IP pools (AC-01)
           - Register parent glue & cut over production DNS delegation
           - Publish verified architecture to freeipa.softcloud.dev
```

### 3.1. Phase 0: Poudriere Build & Linkage Governance
- **Objective**: Guarantee binary compatibility and eliminate symbol collisions between base Heimdal and Ports MIT Kerberos.
- **Actions**:
  1. Configure Poudriere options for `security/cyrus-sasl2-gssapi` and `security/py-gssapi`:
     ```make
     security_cyrus-sasl2-gssapi_SET=GSSAPI_MIT
     security_cyrus-sasl2-gssapi_UNSET=GSSAPI_BASE
     security_py-gssapi_SET=GSSAPI_MIT
     security_py-gssapi_UNSET=GSSAPI_BASE
     ```
  2. Compile `net/freeipa-server` (4.13.4) and `dns/bind920` (9.20.29) against Ports baseline `07689140206dc74cd3a761d4b3983820b58172dc`.
  3. Run ELF linkage audit (`readelf -d / ldd`) asserting dependence on `/usr/local/lib/libgssapi_krb5.so` and complete absence of `/usr/lib/libgss*.so`.
- **Exit Gate**: PASS AC-02, AC-03.

### 3.2. Phase 1: Authoritative Nameserver Infrastructure Deployment
- **Objective**: Establish resilient, authoritative-only DNS with segregated TSIG authentication.
- **Actions**:
  1. Deploy `ns1.example.com` (`192.0.2.10`) and `ns2.example.com` (`192.0.2.11`) across separate power and failure domains.
  2. Generate discrete HMAC-SHA256 TSIG keys:
     - `ipa-ddns`: Restricted to `ns1` and `ipa`.
     - `xfr-example-com`: Shared strictly between `ns1` and `ns2`.
  3. Deploy named configurations and initialize the bootstrap zone `/usr/local/etc/namedb/dynamic/example.com.zone`.
  4. Verify syntax via `named-checkconf` and `named-checkzone`.
- **Exit Gate**: PASS AC-05, AC-06, AC-07, AC-15.

### 3.3. Phase 2: FreeIPA Host Provisioning & Installation
- **Objective**: Install and configure the FreeIPA server on FreeBSD 15.1 without integrated DNS dependencies.
- **Actions**:
  1. Set host FQDN to `ipa.example.com` and verify forward resolution (`dig @192.0.2.10 ipa.example.com A`).
  2. Verify FreeBSD base `ntpd` is synchronized.
  3. Execute `ipa-server-install --hostname=ipa.example.com --domain=example.com --realm=EXAMPLE.COM --no-ntp`.
  4. Persist services in `/etc/rc.conf`:
     ```sh
     sysrc freeipa_server_enable=YES
     sysrc gssproxy_enable=YES
     ```
  5. Perform a full system reboot and test administrative authentication with `kinit admin`.
- **Exit Gate**: PASS AC-04, AC-08, AC-09.

### 3.4. Phase 3: Automated Discovery Record Publication
- **Objective**: Publish the authoritative FreeIPA service records to BIND using dynamic updates.
- **Actions**:
  1. Extract discovery records from FreeIPA:
     ```sh
     ipa dns-update-system-records --dry-run --out=/root/ipa-system.nsupdate
     ```
  2. Bind target server and zone headers:
     ```sh
     printf 'server 192.0.2.10\nzone example.com.\n' > /root/ipa-system.bound.nsupdate
     cat /root/ipa-system.nsupdate >> /root/ipa-system.bound.nsupdate
     ```
  3. Apply update via `nsupdate -k /usr/local/etc/namedb/keys/ipa-ddns.key /root/ipa-system.bound.nsupdate`.
  4. Verify resolution of `_ldap._tcp`, `_kerberos._tcp`, `_kerberos._udp`, `_kpasswd._tcp`, `_kerberos` TXT, and `ipa-ca` across both nameservers.
- **Exit Gate**: PASS AC-10, AC-11, AC-13, AC-14.

### 3.5. Phase 4: Security Boundary & Mail Containment Verification
- **Objective**: Validate that FreeIPA cannot tamper with apex records, SPF, DKIM, DMARC, or mail routing hosts.
- **Actions**:
  1. Attempt unauthorized `nsupdate` operations using `ipa-ddns.key` targeting:
     - Apex `MX` and `TXT`
     - `_dmarc.example.com.` `TXT`
     - `default._domainkey.example.com.` `TXT`
     - `mail.example.com.` `A` / `AAAA`
  2. Confirm every unauthorized modification returns `REFUSED`.
- **Exit Gate**: PASS AC-12, AC-18.

### 3.6. Phase 5: Client Enrollment & High-Availability Verification
- **Objective**: Confirm end-to-end client usability and DNS secondary survival.
- **Actions**:
  1. Join test FreeBSD and Linux workstations via SSSD, validating that Kerberos KDC and LDAP endpoints are discovered purely via DNS SRV without hardcoded IPs.
  2. Stop `named` on `ns1` and verify that `ns2` serves all authoritative records throughout the SOA expiry window (1,209,600 seconds).
- **Exit Gate**: PASS AC-16, AC-17.

### 3.7. Phase 6: Production Cutover & Change Control Operationalization
- **Objective**: Transition from staging placeholders to operational production domains and integrate with the publication platform.
- **Actions**:
  1. Substitute documentation placeholders (`example.com`, `192.0.2.0/24`) with production domain and IP allocations.
  2. Register parent-zone NS delegations and in-bailiwick glue records for `ns1` and `ns2`.
  3. Commit and publish the complete operational specification and verification notebooks to [freeipa.softcloud.dev](https://freeipa.softcloud.dev/).
  4. Convene the Change Control Board (CCB) for final sign-off.
- **Exit Gate**: PASS AC-01 (All 18 Acceptance Criteria verified).

---

## 4. Acceptance Criteria Verification Matrix

| Test ID | Verification Step | Conformance Criterion (PASS) |
|:---|:---|:---|
| **AC-01** | Placeholder Scrub | No operational config contains `example.com` or `192.0.2.0/24`. |
| **AC-02** | Software Baseline | FreeBSD 15.1-RELEASE; FreeIPA 4.13.4; BIND 9.20.29 confirmed. |
| **AC-03** | Kerberos Linkage | GSSAPI binaries resolve `/usr/local/lib/libgssapi_krb5.so`; zero base Heimdal links. |
| **AC-04** | Host Identity | `hostname -f` returns `ipa.example.com`; DNS resolves to `192.0.2.10`. |
| **AC-05** | BIND Config Syntax | `named-checkconf` exits with return code `0` on both servers. |
| **AC-06** | Zone File Integrity | `named-checkzone` reports `OK` on the primary zone file. |
| **AC-07** | Recursion Disabled | `dig @192.0.2.10 www.freebsd.org A` returns status `REFUSED`. |
| **AC-08** | Clean FreeIPA Install | `ipa-server-install` succeeds without `--setup-dns`. |
| **AC-09** | Boot Persistence | Services start cleanly on reboot; `kinit admin` succeeds. |
| **AC-10** | Record Extraction | `ipa dns-update-system-records --dry-run` emits valid discovery RRsets. |
| **AC-11** | Authorized Update | `ipa-ddns.key` successfully updates permitted SRV and TXT records. |
| **AC-12** | Update Containment | `ipa-ddns.key` is `REFUSED` when modifying apex MX/TXT or mail hosts. |
| **AC-13** | Discovery Resolution | All SRV, TXT, and `ipa-ca` records resolve accurately on `ns1` and `ns2`. |
| **AC-14** | Secondary Sync | Updates on `ns1` trigger NOTIFY and synchronize to `ns2`. |
| **AC-15** | Transfer Security | Unauthenticated AXFR fails; TSIG-signed AXFR succeeds. |
| **AC-16** | Secondary Autonomy | `ns2` continues serving zone traffic when `ns1` is stopped. |
| **AC-17** | Client Auto-Discovery | SSSD clients discover KDC and authenticate without static KDC IP overrides. |
| **AC-18** | Mail Namespace Shield | `ipa-ddns.key` is incapable of creating or altering MX, SPF, DKIM, or DMARC records. |

---

## 5. Standard Operating Runbooks & Defect Mitigations

### Runbook A: Dynamic Zone Maintenance (Mitigating KD-06)
Direct manual edits to `/usr/local/etc/namedb/dynamic/example.com.zone` while BIND is active corrupt the runtime journal (`.jnl`). When manual static changes are required:
```sh
# 1. Freeze zone to commit journal to disk and suspend updates
rndc freeze example.com

# 2. Perform necessary zone file edits
vi /usr/local/etc/namedb/dynamic/example.com.zone

# 3. Reload and thaw the zone
rndc reload example.com
rndc thaw example.com
```

### Runbook B: Topology Change & Discovery Resynchronization (Mitigating KD-05)
Because BIND is decoupled from FreeIPA LDAP, topology adjustments (adding replicas, rotating CA certificates) do not automatically propagate to DNS. Administrators must execute:
```sh
ipa dns-update-system-records --dry-run --out=/root/ipa-update.nsupdate
printf 'server 192.0.2.10\nzone example.com.\n' > /root/ipa-update.bound.nsupdate
cat /root/ipa-update.nsupdate >> /root/ipa-update.bound.nsupdate
nsupdate -k /usr/local/etc/namedb/keys/ipa-ddns.key /root/ipa-update.bound.nsupdate
```

### Runbook C: GSSAPI Linkage Validation (Mitigating KD-02)
Execute before any production deployment or package upgrade:
```sh
#!/bin/sh
set -eu
for mod in /usr/local/lib/sasl2/libgssapiv2.so /usr/local/lib/python*/site-packages/gssapi/raw/*.so; do
    if [ -f "$mod" ]; then
        echo "Checking $mod..."
        ldd "$mod" | grep -E "libgssapi_krb5.so" >/dev/null || (echo "ERROR: Missing MIT Kerberos linkage in $mod" && exit 1)
        ldd "$mod" | grep -E "/usr/lib/libgss" && (echo "ERROR: Forbidden Heimdal linkage in $mod" && exit 1) || true
    fi
done
echo "GSSAPI linkage audit: PASS"
```

---

## 6. Organizational Governance & RACI Matrix

| Activity / Milestone | ARB (Architecture) | SecOps (Security) | NetOps (DNS) | DevOps / IAM | QA & Compliance |
|:---|:---|:---|:---|:---|:---|
| **Port & Linkage Configuration** | **A** | **C** | **I** | **R** | **C** |
| **BIND 9 Deployment & Key Mgmt** | **C** | **A** | **R** | **C** | **C** |
| **FreeIPA Base Installation** | **I** | **C** | **I** | **R / A** | **C** |
| **Dynamic Update Policy Scoping** | **A** | **A** | **R** | **R** | **C** |
| **Mail Boundary Negative Testing** | **I** | **A** | **C** | **I** | **R** |
| **Acceptance Criteria Sign-off** | **A** | **A** | **A** | **A** | **R** |
| **Change Control Variations** | **A** | **C** | **C** | **R** | **I** |

*(R = Responsible, A = Accountable, C = Consulted, I = Informed)*

---

## 7. Memorandum Scaffold Specifications

The following repository scaffold files govern the FreeIPA SoftCloud deterministic build and publication toolchain.

### 7.1. `myst.yml`
```yaml
version: 1
project:
  title: FreeIPA SoftCloud
  description: Computational publication and reference platform for FreeIPA Identity & Access Management on Software Cloud.
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

### 7.2. `requirements.in`
```text
jupyter-book==2.1.7
ipykernel==7.3.0
PyYAML==6.0.3
```

### 7.3. Computational Verification Notebook (`notebooks/experiment.ipynb`)
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

## 8. Continuous Integration & Deployment

The automated CI/CD pipeline ([`.github/workflows/deploy.yml`](.github/workflows/deploy.yml)) builds and verifies documentation and notebooks under CPython 3.12.14 in `--strict` mode on every commit to `main`, publishing artifacts directly to [freeipa.softcloud.dev](https://freeipa.softcloud.dev/).
