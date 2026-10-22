# Ghost Records

![Ghost Records - Demo Video](https://github.com/ja1sh/ghost-records/blob/c9042282604df88e411f1fc1034b1ea154e753dc/brag.mp4)


Dangling DNS detection for AWS Elastic IPs. Finds Route53 A and CNAME records pointing at Elastic IPs your accounts no longer own — the highest-impact vector for subdomain takeover on AWS. Cross-account aware: scans multiple AWS accounts and checks DNS records against the combined EIP inventory.

## The Problem

When an Elastic IP is released but its DNS record stays behind, anyone can allocate that IP in the same region and serve content on your subdomain. This is a subdomain takeover.

Ghost Records audits your Route53 zones against your EIP inventory and flags two states:

| Finding | What it means |
|---------|---------------|
| **DANGLING** | The record points at an IP your account does not own. Takeover is possible right now. |
| **AT-RISK** | The record points at an EIP you own but haven't attached to anything. One `release-address` away from becoming dangling. |

## How It Works

1. **EIP Inventory** — Enumerates every Elastic IP across all regions in every scanned account and resolves the resource behind it (EC2 instance, Lambda ENI, NAT Gateway, ELB ENI). When multiple profiles are given, EIPs are collected into a single global inventory before any DNS evaluation.
2. **Route53 Cross-Reference** — Walks every public hosted zone, evaluates each A and CNAME record against the global EIP inventory. Cross-account matches (DNS in Account A pointing to an EIP in Account B) are identified and annotated.
3. **CNAME Chain Resolution** — CNAME targets are resolved from Route53 records first (not external DNS), so dangling IPs behind CNAME chains are correctly detected even when `dig` returns NXDOMAIN.

Non-routable IPs (`0.0.0.0`, RFC 1918, loopback, link-local) are automatically skipped.

## Prerequisites

- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [jq](https://jqlang.github.io/jq/download/)
- `dig` (included with most systems; install via `dnsutils` or `bind-utils` if missing)
- IAM permissions: `SecurityAudit` + `ReadOnlyAccess` (read-only)

## Usage

```bash
# Single account
./ghost_records.sh --profiles prod-sa

# Multiple accounts
./ghost_records.sh --profiles acct1,acct2

# Specific regions only
./ghost_records.sh --profiles prod-sa --regions us-east-1,ap-south-1

# Show healthy (matched) records too
./ghost_records.sh --profiles prod-sa --verbose

# Emit machine-readable JSON
./ghost_records.sh --profiles prod-sa --json

# Faster scans with more parallelism
./ghost_records.sh --profiles prod-sa --parallel 8
```

### Options

| Flag | Description | Default |
|------|-------------|---------|
| `--profiles`, `-p` | Comma-separated AWS CLI profile names | *required* |
| `--regions` | Comma-separated regions, or `all` | `all` |
| `--output-dir`, `-o` | Output directory | `./ghost_records_<timestamp>` |
| `--timeout` | Per-API-call timeout in seconds | `12` |
| `--parallel` | Max parallel region scans | `4` |
| `--verbose`, `-v` | Show matched (OK) records | off |
| `--json` | Emit `results.json` | off |

## Output

Each run creates a timestamped directory with:

| File | Contents |
|------|----------|
| `report.md` | Human-readable findings and remediation guidance |
| `dangling_records.csv` | Records pointing at IPs outside the account |
| `at_risk_records.csv` | Records pointing at allocated-but-idle EIPs |
| `eip_inventory.csv` | Every EIP with its attached resource |
| `full_dns_map.csv` | Every A/CNAME evaluated, with status |
| `results.json` | Machine-readable summary (with `--json`) |
| `raw_json/` | Raw API responses for evidence |

## Example Output

```
  FINDINGS

  DANGLING — takeover possible now (1)
    sub3.example.com                            A      13.81.21.190

  AT-RISK — idle EIP, one release away (3)
    sub1.example.com                            A      13.207.24.192    eipalloc-057cf...
    sub4.example.com                            A      13.202.124.176   eipalloc-0fbab...
    sub5.example.com                            A      13.202.124.176   eipalloc-0fbab...
```

## Remediation

**Dangling** — Delete the DNS record, or re-allocate the address if the service is still needed.

**At-risk** — Attach the EIP to its resource, or delete the DNS record and release the address together (in that order).

**Prevent recurrence** — Add DNS cleanup to IaC teardown so records and addresses are destroyed in the same operation.


## License

[MIT](LICENSE)
