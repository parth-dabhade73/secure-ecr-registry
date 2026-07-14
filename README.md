# Secure Private Container Registry — secure-app

## Overview
This project implements a secure private container registry using Amazon ECR,
solving three problems with unmanaged public registries: security risk,
storage bloat, and lack of lifecycle management.

## Architecture
- **Repository:** `secure-app` (private, AES256-encrypted, scan-on-push enabled)
- **Region:** eu-north-1
- **Authorized identity:** `ecr-dev-user` — full push/pull/list access
- **Unauthorized identity:** `ecr-unauth-user` — no IAM permissions, used to prove access control

## Access Control (Defense in Depth)
Two independent layers protect the registry:
1. **Resource-based repository policy** — explicitly allows `ecr-dev-user` and
   explicitly DENIES every other principal via a `StringNotEquals` condition.
   An explicit deny in AWS always overrides any allow, making this airtight
   even if an unauthorized user were later granted IAM permissions.
2. **IAM identity policy** — `ecr-dev-user` has `AmazonEC2ContainerRegistryFullAccess`;
   `ecr-unauth-user` has none, so it fails even before reaching the repo policy.

## Image Tagging Strategy
| Tag | Purpose |
|---|---|
| v1, v2, v3, v4 | Immutable version history |
| latest | Mutable pointer to the newest stable build |

## Lifecycle Policy
Two rules, evaluated in priority order:
1. Expire untagged images older than 1 day (cleans up build artifacts/manifest layers)
2. Keep only the last 3 tagged images matching `v*` — older versions auto-expire

**Validated via dry-run preview:** confirmed `v1` was correctly targeted for
expiration once `v4` was pushed (4 tagged versions > retain-3 limit).

## Security Validation Results
| Test | User | Result |
|---|---|---|
| List images | ecr-unauth-user | Denied — explicit deny (resource policy) |
| Read repo policy | ecr-unauth-user | Denied — explicit deny (resource policy) |
| Docker login (GetAuthorizationToken) | ecr-unauth-user | Denied — no IAM permission |
| List images | ecr-dev-user | Success — full image list returned |

## Governance Benefits
- **Security:** Private registry + explicit-deny policy eliminates unauthorized access risk
- **Cost control:** Lifecycle rules prevent unbounded storage growth
- **Consistency:** Immutable version tags + mutable `latest` give clear rollback points
- **Auditability:** Scan-on-push flags CVEs automatically on every image
