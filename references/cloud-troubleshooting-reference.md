# Cloud Troubleshooting Reference

Technical reference for debugging Temporal Cloud connectivity, authentication, and configuration issues.

## CLI Tools Overview

Two main CLIs for Temporal Cloud:

| CLI | Purpose | Auth Method | Docs |
|-----|---------|-------------|------|
| `tcld` | Cloud control plane (namespaces, certs, accounts) | Browser-based OAuth | [tcld reference](https://docs.temporal.io/cloud/tcld) |
| `temporal` | Workflow operations (list, describe, execute) | mTLS certs or API keys | [CLI reference](https://docs.temporal.io/cli) |

**Related resources:**
- [Worker best practices](https://docs.temporal.io/best-practices/worker) - deployment, tuning, and operational guidance

## tcld Commands

### Authentication

```bash
# Standard login (opens browser)
tcld login

# Clear stale tokens and re-auth
tcld logout && tcld login

# Check current account context
tcld account get
```

### Namespace Management

```bash
# List namespaces
tcld namespace list

# Get namespace details
tcld namespace get --namespace <namespace>

# List accepted client CA certificates
tcld namespace accepted-client-ca list --namespace <namespace>

# Add CA certificate
tcld namespace accepted-client-ca add --namespace <namespace> --ca-certificate-file ca.pem

# Remove CA certificate (by fingerprint)
tcld namespace accepted-client-ca remove --namespace <namespace> --fp <fingerprint>
```

### Certificate Generation

```bash
# Generate new CA
tcld generate-certificates certificate-authority-certificate \
  --organization temporal \
  --validity-period 365d \
  --ca-certificate-file output_dir/ca.pem \
  --ca-key-file output_dir/ca.key

# Generate leaf certificate from CA
tcld generate-certificates end-entity-certificate \
  --organization temporal \
  --validity-period 365d \
  --ca-certificate-file output_dir/ca.pem \
  --ca-key-file output_dir/ca.key \
  --certificate-file output_dir/client.pem \
  --key-file output_dir/client.key

# Generate another leaf certificate
tcld generate-certificates end-entity-certificate \
  --organization temporal \
  --validity-period 365d \
  --ca-certificate-file ca.pem \
  --ca-key-file ca.key \
  --certificate-file output_dir/client.pem \
  --key-file output_dir/client.key
```

### Private Connectivity

```bash
# List connectivity rules for a namespace
tcld connectivity-rule list --namespace <namespace>

# Get rule details
tcld connectivity-rule get --connectivity-rule-id <rule-id>

# Create AWS PrivateLink rule
tcld connectivity-rule create \
  --connectivity-type private \
  --connection-id "vpce-abcde" \
  --region "aws-us-east-1"

# Create GCP Private Service Connect rule
tcld connectivity-rule create \
  --connectivity-type private \
  --connection-id "1234567890" \
  --region "gcp-us-central1" \
  --gcp-project-id "my-project-123"

# Attach connectivity rules to a namespace
tcld namespace set-connectivity-rules \
  --namespace "my-namespace.abc123" \
  --connectivity-rule-ids "rule-id-1" \
  --connectivity-rule-ids "rule-id-2"
```

### Account & User Management

```bash
# Get account info
tcld account get

# List users
tcld user list

# List service accounts
tcld service-account list

# Create service account
tcld service-account create --name <name> --description "<description>"

# Create API key for service account
tcld apikey create --service-account-id <id> --description "<description>"
```

## temporal CLI Commands

### Connection Testing

```bash
# Basic connectivity test
temporal workflow list --limit 1 \
  --address <namespace>.tmprl.cloud:7233 \
  --namespace <namespace> \
  --tls-cert-path client.pem \
  --tls-key-path client.key

# With API key (same Namespace Endpoint, different auth)
temporal workflow list --limit 1 \
  --address <namespace>.tmprl.cloud:7233 \
  --namespace <namespace> \
  --api-key <key>

# Cluster health check
temporal operator cluster health \
  --address <namespace>.tmprl.cloud:7233 \
  --namespace <namespace> \
  --tls-cert-path client.pem \
  --tls-key-path client.key
```

### Task Queue Diagnostics

```bash
# Describe task queue (shows pollers, backlog)
temporal task-queue describe \
  --task-queue <queue-name> \
  --address <address> \
  --namespace <namespace> \
  --tls-cert-path client.pem \
  --tls-key-path client.key

# Get task queue types
temporal task-queue get-build-ids \
  --task-queue <queue-name> \
  --address <address> \
  --namespace <namespace>
```

### Workflow Operations

```bash
# List workflows
temporal workflow list \
  --address <address> \
  --namespace <namespace>

# Describe specific workflow
temporal workflow describe \
  --workflow-id <id> \
  --address <address> \
  --namespace <namespace>

# Show workflow history
temporal workflow show \
  --workflow-id <id> \
  --address <address> \
  --namespace <namespace>
```

### Endpoint Selection

| Use case | Endpoint |
|----------|----------|
| Workers & clients (all auth) | `<namespace>.<account>.tmprl.cloud:7233` (Namespace Endpoint - recommended) |
| Multi-region HA (advanced) | `<region>.<cloud_provider>.api.temporal.io:7233` (Regional Endpoint) |
| Control plane (`tcld`, Cloud Ops API) | `saas-api.tmprl.cloud` |

Namespace Endpoints work for both mTLS and API key auth. Use Regional Endpoints only for advanced multi-region HA routing. See [namespace access docs](https://docs.temporal.io/cloud/namespaces#access-namespaces).

## OpenSSL Commands

### Certificate Inspection

```bash
# Check certificate expiry
openssl x509 -enddate -noout -in cert.pem

# View full certificate details
openssl x509 -text -noout -in cert.pem

# Check certificate chain
openssl verify -CAfile ca.pem cert.pem

# View certificate subject/issuer
openssl x509 -subject -issuer -noout -in cert.pem

# Check if cert matches private key
openssl x509 -modulus -noout -in cert.pem | md5
openssl rsa -modulus -noout -in key.pem | md5
# (outputs should match)
```

### TLS Connection Testing

```bash
# Test TLS handshake
openssl s_client -connect <namespace>.tmprl.cloud:7233 \
  -cert client.pem -key client.key -CAfile ca.pem

# Test with SNI (required for Temporal Cloud)
openssl s_client -connect <namespace>.tmprl.cloud:7233 \
  -servername <namespace>.tmprl.cloud \
  -cert client.pem -key client.key

# Show server certificate
openssl s_client -connect <namespace>.tmprl.cloud:7233 \
  -showcerts </dev/null 2>/dev/null | openssl x509 -text -noout
```

## Environment Variables

### SDK Connection

All auth methods use the same Namespace Endpoint.

| Variable | Description |
|----------|-------------|
| `TEMPORAL_ADDRESS` | `<namespace>.<account>.tmprl.cloud:7233` (Namespace Endpoint) |
| `TEMPORAL_NAMESPACE` | Namespace name (`<name>.<account-id>`) |
| `TEMPORAL_TLS_CLIENT_CERT_PATH` | Path to client certificate (mTLS only) |
| `TEMPORAL_TLS_CLIENT_KEY_PATH` | Path to client key (mTLS only) |
| `TEMPORAL_TLS_SERVER_CA_CERT_PATH` | Path to CA certificate (mTLS, optional for Cloud) |
| `TEMPORAL_API_KEY` | API key value (API key auth only) |

## Error Codes Reference

### Connection Errors

| Error | Meaning | Common Causes |
|-------|---------|---------------|
| `context deadline exceeded` | Connection timeout | Wrong address, firewall, DNS |
| `connection refused` | Port not reachable | Wrong port, service down |
| `no such host` | DNS resolution failed | Wrong hostname, DNS issues |
| `i/o timeout` | Network timeout | Network issues, wrong region |

## Ambiguous Error Notes

Some high-frequency error strings are too ambiguous to classify from text alone. Use surrounding context before deciding whether the problem is Cloud connectivity, worker capacity, or application behavior.

### `context deadline exceeded`

Do not treat this as a pure network error by default.

Common possibilities:
- wrong endpoint or stale endpoint format
- proxy, DNS, or firewall problem
- client-side timeout before work is accepted
- oversized payload on workflow start
- blocked or slow local execution path

Best first question:
- where exactly did this occur: workflow start, signal/update, query, poll loop, or general logs?

### `workflow is busy` / `RESOURCE_EXHAUSTED: Workflow is busy`

This is often interpreted too broadly. It commonly indicates contention or temporary execution delay on a specific operation, not necessarily a hard workflow failure.

Common possibilities:
- operation-level contention or throttling
- workload pressure or insufficient worker capacity
- user misunderstanding of what the error class refers to

Best first questions:
- which operation returned the error?
- were pollers present?
- was backlog or schedule-to-start latency increasing at the same time?

### `no pollers`

Treat this as a worker reachability or configuration clue, not a complete diagnosis.

Common possibilities:
- workers are not connected to the expected namespace or task queue
- workers are up but misconfigured
- metrics or dashboards are delayed or misleading

Best first checks:
- verify pollers from a direct task-queue surface
- compare with worker logs
- separate "no pollers" from "pollers present but overloaded"

### TLS/Certificate Errors

| Error | Meaning | Common Causes |
|-------|---------|---------------|
| `x509: certificate signed by unknown authority` | CA not trusted | CA not uploaded to namespace |
| `x509: certificate has expired` | Cert past validity | Generate new certificate |
| `x509: certificate is not valid` | Cert not yet valid | Clock skew, wrong cert |
| `tls: bad certificate` | Cert rejected | Wrong cert, missing key |
| `remote error: tls: bad certificate` | Server rejected client cert | CA not in accepted list |
| `tls: private key does not match public key` | Key mismatch | Wrong key file |

### Authentication Errors

| Error | Meaning | Common Causes |
|-------|---------|---------------|
| `PERMISSION_DENIED` | Not authorized | Wrong namespace, missing permissions |
| `UNAUTHENTICATED` | Auth failed | Invalid cert, expired token |
| `INVALID_ARGUMENT: namespace not found` | Namespace doesn't exist | Typo, wrong account |

### Rate Limiting

| Error | Meaning | Common Causes |
|-------|---------|---------------|
| `RESOURCE_EXHAUSTED` | Rate limited | Too many requests, APS limit |
| `RESOURCE_EXHAUSTED: namespace write ops` | Write rate exceeded | Too many workflow starts |
| `RESOURCE_EXHAUSTED: namespace read ops` | Read rate exceeded | Too many queries/lists |

## Cloud Endpoint Patterns

### Standard Endpoints

```
Namespace Endpoint (recommended, all auth): <namespace>.<account>.tmprl.cloud:7233
Regional Endpoint (advanced HA only):       <region>.<cloud_provider>.api.temporal.io:7233
Web UI:                                     https://cloud.temporal.io/namespaces/<namespace>
```

Namespace Endpoints work for both mTLS and API key auth. All namespaces can use them (exception: Flexible Auth pre-release namespaces).

### Regional Endpoints

Pattern: `<region>.<cloud_provider>.api.temporal.io:7233`

Example: `us-east-1.aws.api.temporal.io:7233`

Only recommended for advanced multi-region HA routing. For the full list of supported regions and providers, see [Cloud regions](https://docs.temporal.io/cloud/regions). See also [namespace access docs](https://docs.temporal.io/cloud/namespaces#access-namespaces).

**Note:** Namespace name includes account ID suffix. Full format: `<namespace-name>.<account-id>`

### Private Connectivity Endpoints

When using PrivateLink/Private Service Connect:

```
# AWS PrivateLink
Address: <namespace>.<account>.tmprl.cloud:7233
(resolves to VPC endpoint)

# GCP Private Service Connect
Address: <namespace>.<account>.tmprl.cloud:7233
(resolves to PSC endpoint)
```

## Quick Diagnostic Commands

### Full Connectivity Check

```bash
#!/bin/bash
# --- mTLS variant ---
NS="your-namespace.account-id"
CERT="client.pem"
KEY="client.key"
HOST="$NS.tmprl.cloud"
ADDRESS="$HOST:7233"

echo "=== DNS Resolution ==="
nslookup "$HOST"

echo "=== Port Connectivity ==="
nc -zv "$HOST" 7233

echo "=== TLS Handshake ==="
openssl s_client -connect "$ADDRESS" \
  -servername "$HOST" \
  -cert $CERT -key $KEY </dev/null 2>&1 | head -20

echo "=== Temporal CLI Test ==="
temporal workflow list --limit 1 \
  --address "$ADDRESS" \
  --namespace $NS \
  --tls-cert-path $CERT \
  --tls-key-path $KEY
```

```bash
#!/bin/bash
# --- API key variant (same Namespace Endpoint, different auth) ---
NS="your-namespace.account-id"
HOST="$NS.tmprl.cloud"
ADDRESS="$HOST:7233"
API_KEY="${TEMPORAL_API_KEY}"

echo "=== DNS Resolution ==="
nslookup "$HOST"

echo "=== Port Connectivity ==="
nc -zv "$HOST" 7233

echo "=== TLS Handshake ==="
openssl s_client -connect "$ADDRESS" \
  -servername "$HOST" </dev/null 2>&1 | head -20

echo "=== Temporal CLI Test ==="
temporal workflow list --limit 1 \
  --address "$ADDRESS" \
  --namespace $NS \
  --api-key "$API_KEY"
```

### Certificate Expiry Check

```bash
#!/bin/bash
for cert in *.pem; do
  echo "$cert:"
  openssl x509 -enddate -noout -in "$cert" 2>/dev/null || echo "  (not a certificate)"
done
```
