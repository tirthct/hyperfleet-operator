# HYPERFLEET-1413: Live OLM Classic upgrade verification

**Date:** 2026-11-12  
**PR/test runner and target source:** [operator PR #268](https://github.com/openshift-hyperfleet/hyperfleet-operator/pull/268) at `ec6029de75549a004b9c0002e44a902e7dfe8823`  
**Result:** **PASS — 1 live upgrade spec passed, 0 failed** in 263.729 seconds.

## Steps performed

1. Built a **test-only** `0.1.0` operator image and OLM bundle from main commit `bf7b36b0827d0b53054a01bc960d987639c41738`, immediately before `status.components` was added. Built a **test-only** `0.2.0-rc1` image and bundle from the tested PR commit. Validated both bundles and their related-image declarations.
2. Built one digest-pinned catalog with a `stable` channel containing both bundles. Its `0.2.0-rc1` entry directly `replaces` `0.1.0`.
3. Ran `make test-e2e-upgrade` from the tested PR checkout on a disposable, single-node AMD64 GKE cluster with OLM Classic v0.46.0. The suite approved the initial InstallPlan, verified a healthy baseline with the same `HyperFleetConfig/cluster`, and created an API data record.
4. Approved the upgrade InstallPlan, observed OLM and workload history, checked image and config-hash behavior, verified the API data record, and cleaned up the test installation. Terraform then destroyed the cluster and node pool; remote state was empty and GCP no longer listed the cluster.

## Outcome

| Check | Observed result | Evidence in accompanying ZIP |
| --- | --- | --- |
| OLM upgrade | Both InstallPlans `Complete`; target CSV `hyperfleet-operator.v0.2.0-rc1` `Succeeded` | `olm.json`, `run.json` |
| API image and migration | API changed from the old to target release digest; final API Ready with zero restarts; `db-migrate` init container completed with exit code 0 | `run.json`, `deployments-history.json`, `pods.json` |
| Availability history | Observed API Deployment never fell below 1 available replica for 1 desired replica; new pod became Ready before old pod deletion | `deployments-history.json`, `pods-history.json` |
| Operator health and data | Final HFC: `Available=True`, `Progressing=False`, `Degraded=False`; pre-upgrade API record retained its fingerprint | `conditions.json`, `data-survival.json` |
| Config hash | Independent expected hashes matched all six target components; database Secret metadata changed the API hash as expected; no-op and unrelated-component checks passed | `exact-hashes.json`, `hash-probes.json` |
| Test result and cleanup | 1 passed, 0 failed; suite recorded `cleaned=true` | `junit-upgrade.xml`, `result.json`, `cleanup.json` |

The ZIP also includes `cluster-version.json` and `platforms.json`. It contains selected test evidence only; it excludes Secret values, fixture credentials, and pod logs.

## Evidence limit

This is a manual pre-merge smoke test using **locally built test artifacts**, following Ciarán's suggested release-pair recipe. It does not qualify an official previous-final/Konflux-RC pair or complete the recurring Prow CI requirement. Observed Kubernetes readiness is not an external HTTP zero-downtime SLA, and this pair does not exercise a real released schema migration.
