# ⚡ Poridhi Cloud-Engine: Test & Bug Audit Summary

> **Target Subsystem:** `cloud-engine` (Compute, Firecracker v1.15.1, OVN & Temporal)  
> **Environment:** Live 3-Node Baremetal Staging Cluster (`103.174.50.21`, `54.38.94.139`, `51.38.54.39`)  
> **Status:** Core Data-Plane Operational | 6 Runtime Bugs Audited & Documented  

---

## 🚀 1. Core Data-Plane Verification (What Works)

### 🔹 Compute Nodes Readiness (`node-01` & `node-02`)
- **Status:** Both baremetal compute agents registered and reporting `state: "ready"` with Firecracker v1.15.1.
- **Proof:** Node agent inspection confirms hypervisor initialization across both physical servers.

![Compute Nodes Ready](./test_evidence/bm_nodes_ready.png)

---

### 🔹 Tenant & VPC Network Provisioning
- **Status:** Dedicated tenant account created with isolated VPC (`vpc-08aeb8c0`) and Geneve VNI 100.
- **Proof:** Control-plane assigns unique overlay network parameters and stores state in etcd.

![Tenant Account Created](./test_evidence/bm_account_create.png)

---

### 🔹 MicroVM Hardware Virtualization Running
- **Status:** Real guest microVM booted to `state: "running"` with private IP `10.0.0.1`, TAP `fc-65231f3f`, PID `3869818`.
- **Proof:** Real Intel Xeon KVM virtualization confirmed on Node-01.

![MicroVM Running](./test_evidence/bm_vm_running.png)

---

### 🔹 MicroVM SSH & Network Namespace Info
- **Status:** Guest connection metadata mapped into `ns-vpc-vpc-08aeb8c0`.
- **Proof:** Proxy connection strings and logical port bindings verified.

![VM SSH Connection Info](./test_evidence/bm_vm_ssh_info.png)

---

### 🔹 In-Guest Network Reachability (ICMP Ping)
- **Status:** Ping across virtual TAP inside VPC namespace: **3 packets transmitted, 3 received, 0% packet loss, 0.34ms round-trip latency**.
- **Proof:** Virtualized network stack between host OVN switch and guest kernel is fully functional.

![In-Guest Ping Success](./test_evidence/bm_in_guest_ping_success.png)

---

## ⚠️ 2. Audited Bugs & Direct Screenshot Evidence

### 🔴 BM-BUG-01: MicroVM Snapshot Fails (Jailer PID Mismatch)
- **Keyword:** `Snapshot Failure` | **Severity:** High
- **Problem:** Captures Jailer launcher PID (`3869818`) instead of child Firecracker daemon PID inside chroot jail.
- **Result:** `/proc/<pid>/cmdline` verification fails precondition $\to$ Snapshot permanently marked `failed`.
- **Actionable Fix:** Inspect `/proc/<jailer_pid>/task/` or cgroup process list to record the actual child Firecracker daemon PID upon VM launch.

![BM-BUG-01 Snapshot Precondition Failure Evidence](./test_evidence/bug_bm_01_snapshot_pid_failure.png)

---

### 🟠 BM-BUG-02: Blind 202 Accepted on Fake VMs (Phantom Workflows)
- **Keyword:** `Phantom Workflows` | **Severity:** Medium
- **Problem:** Calling restart or terminate on non-existent VM IDs returns `202 Accepted` instead of `404 Not Found`.
- **Result:** Triggers unnecessary Temporal workflows that consume worker threads and generate false alerts.
- **Actionable Fix:** Add synchronous `GetVM` existence check in Gin route handlers before starting workflows.

![BM-BUG-02 Blind 202 Accepted Evidence](./test_evidence/bug_bm_02_fake_vm_202_restart.png)

---

### 🟡 BM-BUG-03: Unbounded Duplicate Account Creation & VNI Leak
- **Keyword:** `VNI Exhaustion` | **Severity:** Medium
- **Problem:** Submitting identical email repeatedly creates multiple accounts and VPCs.
- **Result:** Leaks and permanently exhausts unique Geneve VNIs (100–16,777,215).
- **Actionable Fix:** Maintain an inverted key index `/accounts-by-email/<email>` via atomic etcd CAS transaction.

![BM-BUG-03 Duplicate Account VNI Leak Evidence](./test_evidence/bug_bm_03_duplicate_account_vni_leak.png)

---

### 🟡 BM-BUG-04: API Contract Key Divergence (`account_id` vs `id`)
- **Keyword:** `Contract Divergence` | **Severity:** Low
- **Problem:** `POST /accounts/create` returns `"account_id"`, but `GET /accounts/:id` returns `"id"`.
- **Result:** Breaks SDKs and clients expecting consistent schema keys.
- **Actionable Fix:** Standardize response structs to emit both `id` and `account_id`.

![BM-BUG-04 Account ID vs ID Divergence Evidence](./test_evidence/bug_bm_04_account_id_vs_id_divergence.png)

---

### 🟡 BM-BUG-05: REST Route Divergence (404 on Bare Plural Nouns)
- **Keyword:** `Routing 404` | **Severity:** Low
- **Problem:** `GET /accounts` and `GET /vms` return `404 page not found` (only `/accounts/list` works).
- **Result:** Breaks standard REST conventions and API gateway reverse proxies.
- **Actionable Fix:** Add alias route registrations in `handler.go` (`r.GET("/accounts", h.ListAccounts)`).

![BM-BUG-05 Route 404 Divergence Evidence](./test_evidence/bug_bm_05_route_404_divergence.png)

---

### 🟠 BM-BUG-06: Premature IP & MAC Lease on Invalid Image ID
- **Keyword:** `Premature IP Lease` | **Severity:** Medium
- **Problem:** Requesting a VM with an invalid `image_id` immediately reserves private IP `10.0.0.2` and MAC before failing.
- **Result:** Wastes IP leases in tenant VPC.
- **Actionable Fix:** Validate `image_id` against etcd synchronously before calling the IPAM allocator.

![BM-BUG-06 Premature IP Lease Evidence](./test_evidence/bug_bm_06_invalid_img_premature_ip_lease.png)

---

## 💻 3. Local Dev vs Baremetal Boundary

- **Local Laptop:** Control-Plane validation only (`etcd`, `postgres`). `POST /vms/create` returns `503 Orchestrator not configured` (expected because laptop lacks KVM/root TAP privileges).
- **Baremetal Staging:** Full Data-Plane with real AWS Firecracker microVMs and OVN overlay switching.

![Local E2E Flow Execution](./test_evidence/local_e2e_flow_run.png)

---

## 📋 4. Master Bug Action Matrix

| Bug Ref | Environment | Severity | Component | Issue Summary | Actionable Patch |
| :---: | :---: | :---: | :---: | :--- | :--- |
| **BM-BUG-01** | Baremetal | **High** | Hypervisor | VM Snapshot fails with Jailer PID mismatch | Track child Firecracker daemon PID instead of Jailer wrapper PID. |
| **BM-BUG-02** | Baremetal | **Medium** | API / Temporal | Blind 202 Accepted on non-existent resources | Add synchronous existence check in Gin handlers before dispatching workflows. |
| **BM-BUG-03** | Baremetal | **Medium** | Store / IPAM | Unbounded duplicate accounts & VNI leak | Enforce unique email index `/accounts-by-email/<email>` via etcd CAS transaction. |
| **BM-BUG-04** | Baremetal | **Low** | Contract | Key divergence (`account_id` on POST vs `id` on GET) | Emit both `id` and `account_id` in response JSON structs. |
| **BM-BUG-05** | Baremetal | **Low** | Routing | 404 on standard REST plural nouns | Add alias route registrations in `handler.go` (`/accounts`, `/vms`). |
| **BM-BUG-06** | Baremetal | **Medium** | API / IPAM | Premature IP lease on invalid image ID | Validate `image_id` existence prior to reserving VPC private IP. |
| **LOCAL-BUG-02**| Local / Prod | **High** | etcd Store | Non-atomic `SetVMState` Read-Modify-Write | Replace `GetVM` $\to$ `PutVM` with etcd CAS ModRevision retry transaction. |
| **LOCAL-BUG-03**| Local / Prod | **High** | IPAM / Perf | `AllocateVNI` linear CAS retry loop storm | Implement randomized exponential backoff / cursor chunking. |
| **LOCAL-BUG-04**| Local / Prod | **High** | DevOps / CI | Missing `TEMPORAL_PAYLOAD_KEY` in `.env.example` | Document encryption payload key in `.env.example`. |
| **LOCAL-BUG-05**| Local / Prod | **High** | CI / Workflow | Missing Temporal replay test suite | Create `replay_test.go` and record history fixtures. |

---

> 📄 **Full Technical Reference:**  
> Detailed test procedures, configuration parameters, and RCA logs are preserved in [PORIDHI_LOCAL_AND_BAREMETAL_TESTING_GUIDE_AND_BUG_REPORT.md](./PORIDHI_LOCAL_AND_BAREMETAL_TESTING_GUIDE_AND_BUG_REPORT.md).
