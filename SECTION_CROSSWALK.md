# xdv-draft to xdv-spec Crosswalk

This document maps archived `xdv-draft` sections to canonical `xdv-spec` IDs.

## Draft Section Mapping

| Archived Draft Section | Canonical xdv-spec IDs |
|---|---|
| Overview / Core Definition | XDV-001, XDV-002 |
| Domain Model (K/Q/Phi) | XDV-002, XDV-003 |
| Cross-Domain Virtualization | XDV-010, XDV-014 |
| DAL (Domain Abstraction Layer) | XDV-011 |
| CDS (Cross-Domain Scheduler) | XDV-012, XDV-082 |
| UMF (Unified Memory Fabric) | XDV-013 |
| Domain Hypervisor | XDV-014, XDV-083 |
| SDBM (Secure Boundary Manager) | XDV-015 |
| ODT (Observability/Trace) | XDV-070, XDV-071, XDV-072 |
| Execution Model | XDV-004, XDV-020, XDV-021, XDV-022, XDV-023, XDV-024 |
| Scheduling Semantics | XDV-005, XDV-012, XDV-041, XDV-082 |
| Security Architecture | XDV-030, XDV-031, XDV-032, XDV-033, XDV-034 |
| DPL Integration | XDV-050, XDV-051, XDV-052, XDV-053 |
| Hybrid Compute Platforms / Hardware Integration | XDV-060, XDV-061, XDV-062 |
| Compatibility / Runtime Surface | XDV-052, XDV-081 |
| Networking / Distributed / Cloud | XDV-040, XDV-041, XDV-042, XDV-043 |
| Reference Implementation Scope | XDV-080, XDV-081, XDV-082, XDV-083 |
| Conformance / Certification | XDV-084 |

## Legacy File Mapping

| Archived File | Canonical Primary Targets |
|---|---|
| `xdv-white_paper.md` | XDV-001..005, XDV-010..015, XDV-030..034 |
| `xdvfs-White-Paper.md` | XDV-013, XDV-023, XDV-024, XDV-070..072 |
| `DESIGN_PRINCIPLES.md` | XDV-001, XDV-002, XDV-005 |
| `XDV-OS-Implementation-Plan.md` | XDV-062, XDV-080, XDV-081, XDV-084 |
| `XDV-OS-Implementation-Plan-DPL.md` | XDV-050, XDV-051, XDV-052, XDV-053, XDV-080, XDV-081 |
| `XDV-OS-Implementation-Plan-v3.md` | XDV-050..053, XDV-062, XDV-080..084 |
| `XDV-OS-Implementation-Plan-v4.md` | XDV-052, XDV-062, XDV-080, XDV-081, XDV-084 |

## Resolution Rule

If any archived draft statement conflicts with `xdv-spec`, the `xdv-spec` statement is authoritative.
