# Drug Supply Chain BlockChain

[![status: stable](https://img.shields.io/badge/status-stable-1f7a1f)](.)
[![scope: blockchain](https://img.shields.io/badge/scope-blockchain-2b4c7e)](.)
[![integration: protected-qr](https://img.shields.io/badge/integration-protected--qr-0f766e)](.)

Permissioned Hyperledger Fabric blockchain for pharmaceutical anti-counterfeit workflows.

This repository is intentionally blockchain-only. Backend and Protected QR services run independently and integrate through Fabric Gateway and API contracts.

## Key Guarantees

- Immutable batch lifecycle history on ledger.
- Strict on-chain Protected QR metadata validation.
- Confidence-based physical verification evidence recorded on-chain.
- Deterministic submit/evaluate transaction boundary.
- Scripted governance for lifecycle and organization onboarding.

## Architecture

- **RegulatorMSP**: governance, lifecycle control, emergency recall.
- **ManufacturerMSP**: batch creation, QR binding, document updates, shipping.
- **DistributorMSP**: receiving and ownership continuation.
- **Backend (external service)**: API orchestration and gateway integration.
- **Protected QR (external service)**: QR generation and physical verification.

## Project Layout

```text
Drug_Chain/
    asset-transfer-drug/
        chaincode-js/
            lib/drugTracker.js
            META-INF/statedb/couchdb/indexes/indexBatchDocType.json
        scripts/
            blockchain_run.sh
            blockchain_smoke_test.sh
            update_code_centralized.sh
            add_org_centralized.sh
        infrastructure/
            canonical/
                configtx/
                compose/
                crypto-config/
                scripts/
        README.md
        ARCHITECTURE_AND_DEV_GUIDE.md
    test-network/
    bin/
    config/
    docker-compose.yml
    .env.example
    install-fabric.sh
```

## API Overview (On-Chain)

- `CreateBatch`
- `CreateBatchWithExpiry`
- `ReadBatch`
- `VerifyBatch`
- `EvaluateBatchRisk`
- `UpdateDocument`
- `BindProtectedQR`
- `ReadProtectedQR`
- `VerifyProtectedQR`
- `RecordProtectedQRVerification`
- `ShipBatch`
- `ReceiveBatch`
- `EmergencyRecall`

Detailed developer guide:

- [asset-transfer-drug/ARCHITECTURE_AND_DEV_GUIDE.md](asset-transfer-drug/ARCHITECTURE_AND_DEV_GUIDE.md)

## Protected QR Metadata Contract

All metadata fields are strict hexadecimal strings:

- `data_hash`: 8 hex chars
- `metadata_series`: 16 hex chars
- `metadata_issued`: 16 hex chars
- `metadata_expiry`: 16 hex chars
- `token_digest`: 64 hex chars (`sha256(token)`)

## Requirements

- Docker + Docker Compose
- Hyperledger Fabric binaries
- macOS/Linux shell (bash/zsh)

## Deployment

### Option A: Scripted Development Flow (Recommended)

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_run.sh prereq
./scripts/blockchain_run.sh full
./scripts/blockchain_smoke_test.sh
```

### Option B: Canonical Docker Topology

```bash
cd Drug_Chain/asset-transfer-drug
./infrastructure/canonical/scripts/canonical_bootstrap.sh generate
cd ..
docker compose up -d
```

Top-level `docker-compose.yml` mirrors canonical Fabric topology for consistency.

### Option C: Root Docker Compose Validation

```bash
cd Drug_Chain
docker compose -f docker-compose.yml config -q
```

Use this check before `docker compose up -d` to validate syntax and mounts.

## Installation

```bash
cd Drug_Chain
./install-fabric.sh docker binary
cp .env.example .env
```

## Run (Production-Like)

```bash
cd Drug_Chain/asset-transfer-drug
CC_VERSION=2.0 CC_SEQUENCE=2 ./scripts/blockchain_run.sh upgrade
./scripts/blockchain_smoke_test.sh
```

## Run (Development)

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_run.sh full
./scripts/blockchain_smoke_test.sh
```

## Usage Flow

1. Start network and deploy chaincode.

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_run.sh prereq
./scripts/blockchain_run.sh full
```

2. Run blockchain smoke flow (batch create, QR bind, verify, ship/receive).

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_smoke_test.sh
```

3. Shut down and clean network artifacts.

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_run.sh down
```

## Gateway Usage Examples

Evaluate transaction (read-only):

```javascript
const result = await contract.evaluateTransaction("ReadBatch", batchId);
```

Submit transaction (state-changing):

```javascript
await contract.submitTransaction(
    "BindProtectedQR",
    batchId,
    dataHash,
    metadataSeries,
    metadataIssued,
    metadataExpiry,
    tokenDigest,
);
```

Production integration guideline:

- Backend should use org-scoped Fabric identities per role (Manufacturer, Distributor, Regulator).
- Protected QR service should hash and validate metadata off-chain before submit.
- Keep idempotency logic in backend for retry-safe submit operations.

## Final Pre-Public Checklist

Run this exact checklist before publishing:

1. Scripted runtime regression

```bash
cd Drug_Chain/asset-transfer-drug
./scripts/blockchain_run.sh prereq
./scripts/blockchain_run.sh full
./scripts/blockchain_smoke_test.sh
./scripts/blockchain_run.sh down
```

2. Canonical Docker regression

```bash
cd Drug_Chain/asset-transfer-drug
./infrastructure/canonical/scripts/canonical_bootstrap.sh generate
cd ..
docker compose -f docker-compose.yml config -q
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml ps
docker compose -f docker-compose.yml down --remove-orphans
```

3. Final doc sanity

- Verify environment table matches `.env.example`.
- Verify chaincode API list matches deployed contract methods.
- Verify all external links resolve.

## Environment Variables

See [.env.example](.env.example).

| Name                  | Required | Description                                           |
| --------------------- | -------- | ----------------------------------------------------- |
| `CHANNEL_NAME`        | No       | Fabric channel name.                                  |
| `CC_NAME`             | No       | Chaincode name.                                       |
| `CC_LANG`             | No       | Chaincode language (`javascript`).                    |
| `CC_VERSION`          | No       | Chaincode version label.                              |
| `CC_SEQUENCE`         | No       | Lifecycle sequence number.                            |
| `VERIFY_TIMES`        | No       | Smoke test verify iterations.                         |
| `QR_DATA_HASH`        | No       | Protected QR metadata hash (8 hex).                   |
| `QR_METADATA_SERIES`  | No       | Protected QR series metadata (16 hex).                |
| `QR_METADATA_ISSUED`  | No       | Protected QR issued metadata (16 hex).                |
| `QR_METADATA_EXPIRY`  | No       | Protected QR expiry metadata (16 hex).                |
| `QR_TOKEN_DIGEST`     | No       | SHA-256 token digest (64 hex).                        |
| `QR_IS_AUTHENTIC`     | No       | Physical verification boolean for evidence recording. |
| `QR_CONFIDENCE_SCORE` | No       | Physical confidence score in `[0,1]`.                 |

## Operational Notes

- Use `submit` for state-changing methods; use `evaluate` for read-only methods.
- Always increment `CC_SEQUENCE` when updating chaincode definitions.
- Keep organization MSP keys and wallet material outside source control.

## Troubleshooting

- If `peer` is missing, run `./scripts/blockchain_run.sh prereq`.
- If lifecycle commit fails, confirm sequence/version increments.
- If smoke test fails due existing batch ID, rerun (script auto-generates unique ID).

## Security Considerations

- Enforce TLS at gateway and API edges.
- Use role-scoped identities for submit operations.
- Validate metadata format before submit transactions.
- Monitor chaincode events for anomaly and recall signals.

## Bug Reports

Found a bug? Please report it using our [Bug Report Form](https://github.com/MinzNhat/Drug_Chain/issues/new?template=bug_report.yml).

## Feature Requests

Have an idea for a new feature? Submit a [Feature Request Form](https://github.com/MinzNhat/Drug_Chain/issues/new?template=feature_request.yml).

## License

This project is licensed under the Apache-2.0 License - see the [LICENSE](LICENSE) file for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/MinzNhat/Drug_Chain/issues)
- **Discussions**: [GitHub Discussions](https://github.com/MinzNhat/Drug_Chain/discussions)
- **Email**: nhat.dang2004.cv@gmail.com

---

<div align="center">
    <p>Made with care by MinzNhat</p>
    <p>
        <a href="https://github.com/MinzNhat/Drug_Chain">Star us on GitHub</a> •
        <a href="https://github.com/MinzNhat">Visit Profile</a>
    </p>
</div>
