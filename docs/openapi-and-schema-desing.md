# Contract-First Design with OpenAPI 3.2 and Spectral

```mermaid
   sequenceDiagram
    autonumber
    participant Dev as Developer / Termux CLI
    participant Spec as OpenAPI 3.1 Spec
    participant Linter as Spectral CLI
    participant CI as GitHub Actions Gate

    Dev->>Spec: Write/Update Spec (.yaml)
    Dev->>Linter: Run Local Linting (`spectral lint`)
    Linter-->>Dev: Pass / Fail Report
    Dev->>CI: Push Git Commit / Open PR
    CI->>Linter: Enforce Governance Gate


