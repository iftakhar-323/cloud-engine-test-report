# ⚡ Poridhi Cloud-Engine: At-A-Glance Test & Bug Summary

> **Prepared For:** Senior Engineering Leadership (Sagore Sarker Bhai) & Infrastructure Team  
> **Target Subsystem:** `cloud-engine` (Compute, Firecracker v1.15.1, OVN & Temporal)  
> **Environment:** Live 3-Node Baremetal Staging Cluster (`103.174.50.21`, `54.38.94.139`, `51.38.54.39`)  
> **Status:** ✅ Core Data-Plane 100% Operational | ⚠️ 6 Runtime Bugs Audited & Documented  

---

## 🚀 1. Core Data-Plane Status (What Works - 100% Success)

| Subsystem | Verified Status | Live Terminal Proof |
| :--- | :--- | :--- |
| **Compute Agents** | ✅ `node-01` & `node-02` **Ready** with Firecracker v1.15.1 | ![Nodes Ready](./test_evidence/bm_nodes_ready.png) |
| **Tenant Isolation** | ✅ Account `08aeb8c0...` & VPC `vpc-08aeb8c0` (VNI 100) Created | ![Account Create](./test_evidence/bm_account_create.png) |
| **Rootfs Pipeline** | ✅ Ubuntu 22.04 (`img-facdc7a1`) built & uploaded to MinIO S3 | Status: `ready` via Temporal |
| **MicroVM Virtualization** | ✅ VM `iftakhar-bm-vm1` (`65231f3f...`) booted to **Running** | ![VM Running](./test_evidence/bm_vm_running.png) |
| **In-Guest Network** | ✅ Ping from host netns into MicroVM: **0% Loss, 0.34ms Latency** | ![Ping Success](./test_evidence/bm_in_guest_ping_success.png) |

---

## ⚠️ 2. Baremetal Bug Summary (Key Issues & Fixes)

| # | Bug ID | Issue (Keyword) | Root Cause (RCA) | Actionable Fix | Evidence |
| :---: | :---: | :--- | :--- | :--- | :---: |
| 1 | **BM-BUG-01** | **Snapshot Fails** | Captures Jailer wrapper PID instead of Firecracker daemon PID | Track child Firecracker PID in `/proc/<pid>/task/` | [View Image](./test_evidence/bug_bm_01_snapshot_pid_failure.png) |
| 2 | **BM-BUG-02** | **Phantom 202** | Fake VM ID returns `202 Accepted` and starts phantom workflow | Add synchronous `GetVM` check before starting Temporal | [View Image](./test_evidence/bug_bm_02_fake_vm_202_restart.png) |
| 3 | **BM-BUG-03** | **Duplicate Account** | Same email creates multiple accounts & leaks Geneve VNIs | Add unique index on `/accounts-by-email/<email>` via etcd CAS | [View Image](./test_evidence/bug_bm_03_duplicate_account_vni_leak.png) |
| 4 | **BM-BUG-04** | **Contract Mismatch** | Create returns `account_id`, but Get returns `id` | Standardize structs to emit both `id` and `account_id` | [View Image](./test_evidence/bug_bm_04_account_id_vs_id_divergence.png) |
| 5 | **BM-BUG-05** | **Route 404** | `GET /accounts` and `/vms` return 404 (only `/list` works) | Add REST alias route registrations in `handler.go` | [View Image](./test_evidence/bug_bm_05_route_404_divergence.png) |
| 6 | **BM-BUG-06** | **Premature IP Lease** | Invalid `image_id` allocates IP (`10.0.0.2`) before VM fails | Validate `image_id` synchronously before IPAM allocation | [View Image](./test_evidence/bug_bm_06_invalid_img_premature_ip_lease.png) |

---

## 💻 3. Local Dev vs Baremetal Boundary

- **Local Laptop:** Control-Plane only (`etcd`, `postgres`). `POST /vms/create` returns `503 Orchestrator not configured` (expected because laptop lacks KVM/root TAP privileges).
- **Baremetal Staging:** Full Data-Plane with real AWS Firecracker microVMs and OVN overlay switching.

---

> 📄 **Full Detailed Report:** See [PORIDHI_LOCAL_AND_BAREMETAL_TESTING_GUIDE_AND_BUG_REPORT.md](./PORIDHI_LOCAL_AND_BAREMETAL_TESTING_GUIDE_AND_BUG_REPORT.md) for full commands, outputs, and deep-dive RCA.
