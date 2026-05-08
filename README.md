# dpdk-eks-scripts

Support scripts for running DPDK workloads on a self-managed EKS cluster (Kubernetes 1.35, Amazon Linux 2023, arm64, r7g instances). Files collected here from upstream sources, adapted for AL2023 and kernel 6.12, and used as the single source of truth fetched by Terraform `user_data` at node bootstrap time.

## File inventory

| File | Source | License |
|------|--------|---------|
| `scripts/dpdk-devbind.py` | [aws-samples/dpdk-setup-eks@ec0bb30](https://github.com/aws-samples/dpdk-setup-eks/tree/ec0bb30ea6b2ae2abba688bdad6fbd3ece9da849) | MIT-0 |
| `scripts/dpdk-resource-builder.py` | aws-samples/dpdk-setup-eks@ec0bb30 | MIT-0 |
| `scripts/sriov-init.sh` | aws-samples/dpdk-setup-eks@ec0bb30 — **modified** | MIT-0 |
| `scripts/config-sriov.sh` | aws-samples/dpdk-setup-eks@ec0bb30 — **modified** (comment only) | MIT-0 |
| `scripts/get-vfio-with-wc.sh` | [amzn/amzn-drivers PR #370](https://github.com/amzn/amzn-drivers/pull/370) | Apache 2.0 |
| `systemd/sriov-init.service` | aws-samples/dpdk-setup-eks@ec0bb30 | MIT-0 |
| `systemd/config-sriov.service` | aws-samples/dpdk-setup-eks@ec0bb30 | MIT-0 |
| `patches/linux-6.12-vfio-wc.patch` | amzn/amzn-drivers PR #370 | Apache 2.0 |
| `nodeadm/bootstrap.yaml.tmpl` | new | Apache 2.0 |

MIT-0 requires no attribution. Apache 2.0 attribution is in `NOTICE`.

## What was changed

### `scripts/sriov-init.sh`
The original script downloaded `get-vfio-with-wc.sh` and kernel patches from an S3 bucket using `aws s3api get-object --bucket S3BucketName ...`. The S3 bucket name was itself a sed-substitution target injected by Terraform user_data at boot.

Replaced all three `aws s3api get-object` calls with `curl` calls pointing to this repo's raw GitHub URLs. Only the `linux-6.12-vfio-wc.patch` is fetched (the older 4.10 and 5.8 patches are not needed on AL2023). A `REPO_BASE` variable at the top of the script must be updated to your GitHub organisation name before deployment.

### `scripts/config-sriov.sh`
Added a comment block explaining the two sed-substitution placeholders (`subnetCount`, `SriovStartingInterface`) that user_data is expected to fill via `sed -i` before writing the file to `/opt/dpdk/`. No functional changes.

### `scripts/get-vfio-with-wc.sh`
No changes required. The upstream PR #370 already uses `dnf` for package management and explicitly targets `kernel6.12-devel` / `kernel6.12-headers`, which are correct for AL2023 with kernel 6.12. The script falls back to `dnf` automatically when `apt-get` is absent.

### `nodeadm/bootstrap.yaml.tmpl`
New file. A nodeadm `NodeConfig` template for AL2023 EKS nodes with DPDK-specific kubelet settings: 250 max pods, static CPU manager policy, DPDK node labels, and CPUs 0–1 reserved for the OS. Shell variable placeholders (`${CLUSTER_NAME}`, etc.) are expanded by `envsubst` in user_data before passing the config to `nodeadm init`.

## How user_data uses these files

```bash
#!/bin/bash
set -euo pipefail

BASE="https://raw.githubusercontent.com/YOUR_ORG/dpdk-eks-scripts/main"
DPDK_DIR="/opt/dpdk"

mkdir -p "${DPDK_DIR}"

# Download DPDK utilities
curl -fsSL "${BASE}/scripts/dpdk-devbind.py"        -o "${DPDK_DIR}/dpdk-devbind.py"
curl -fsSL "${BASE}/scripts/dpdk-resource-builder.py" -o "${DPDK_DIR}/dpdk-resource-builder.py"
curl -fsSL "${BASE}/scripts/sriov-init.sh"           -o "${DPDK_DIR}/sriov-init.sh"
curl -fsSL "${BASE}/scripts/config-sriov.sh"         -o "${DPDK_DIR}/config-sriov.sh"
chmod +x "${DPDK_DIR}"/*.py "${DPDK_DIR}"/*.sh

# Inject runtime values into config-sriov.sh
sed -i "s/subnetCount/${MULTUS_SUBNET_COUNT}/" "${DPDK_DIR}/config-sriov.sh"
sed -i "s/SriovStartingInterface/${SRIOV_START_IF}/" "${DPDK_DIR}/config-sriov.sh"

# Download and install systemd units
curl -fsSL "${BASE}/systemd/sriov-init.service"   -o /etc/systemd/system/sriov-init.service
curl -fsSL "${BASE}/systemd/config-sriov.service" -o /etc/systemd/system/config-sriov.service
systemctl daemon-reload
systemctl enable sriov-init.service

# Bootstrap the node with nodeadm
curl -fsSL "${BASE}/nodeadm/bootstrap.yaml.tmpl" -o /tmp/bootstrap.yaml.tmpl
export CLUSTER_NAME API_SERVER_URL B64_CLUSTER_CA SERVICE_CIDR
envsubst < /tmp/bootstrap.yaml.tmpl > /tmp/bootstrap.yaml
nodeadm init --config-source file:///tmp/bootstrap.yaml
```

## Attribution

`scripts/get-vfio-with-wc.sh` and `patches/linux-6.12-vfio-wc.patch` are derived from
[amzn/amzn-drivers](https://github.com/amzn/amzn-drivers), specifically
[PR #370](https://github.com/amzn/amzn-drivers/pull/370) by
[@roikeman](https://github.com/roikeman), obtained at commit
`2aa12fd1259aae78f3aae7ef69149013e2ca618b`.
These files are Copyright Amazon.com, Inc. and licensed under the
[Apache License, Version 2.0](LICENSE). See `NOTICE` for the full attribution notice.
