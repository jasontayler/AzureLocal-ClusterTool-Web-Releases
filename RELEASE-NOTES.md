# High-Level Release Notes - Azure Local Cluster Tool (Web)

This file is the source for published GitHub Release bodies.
Each release body is generated from exactly one matching section (latest-only, non-cumulative).

## v0.12.9 - 2026-07-14

### Highlights
- AKS workload operations were expanded with scale and rollback, plus inline per-row actions.
- A workload browse-and-select picker was added to reduce manual entry errors during AKS actions.
- AKS pages received performance and security-focused refinements for better responsiveness.
- Fleet-level AKS visibility and action flows were refined for more consistent cross-cluster operations.

### Platform and Access
- RBAC handling for AKS and related action surfaces was tightened to align view and operate behavior.
- GitOps and AKS action paths were improved for clearer operator control and safer execution.

### Internal and Operational
- Instruction and optimization guidance files were refreshed to match current implementation patterns.

## v0.12.8 - 2026-06-01

### Highlights
- Added AKS cross-cluster workload restart actions from a global operations page.
- Added WinAuth support for multiple HciAccess group SIDs.
- Added a WinAuth cross-domain role-assignment policy gate to reduce false 403 outcomes.
