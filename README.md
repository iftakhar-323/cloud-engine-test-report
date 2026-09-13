# Cloud Engine - Baremetal & Local Testing Report

Tested on: September 13, 2026  
Test Setup:
- Control Plane: `103.174.50.21` (API, Temporal, etcd, MinIO)
- Compute Node 01: `54.38.94.139` (Firecracker, OVN Controller)
- Compute Node 02: `51.38.54.39` (Firecracker, OVN Central DB)

---

## 1. Overview

We ran manual and automated end-to-end tests for `cloud-engine` on both a local development environment and the 3-node baremetal staging cluster.

On baremetal, the core data-plane is working:
- Firecracker v1.15.1 agents are running on both compute nodes.
- Ubuntu 22.04 rootfs was built and pulled from MinIO.
- A guest microVM (`iftakhar-bm-vm1`) successfully booted to `running` state.
- Network ping into the microVM inside the VPC namespace replied with 0% packet loss and 0.34ms latency.

During the test run, we identified 6 bugs on baremetal and a few edge cases in local testing. The reproducible commands, actual terminal outputs, and screenshots are documented below.

---

## 2. Baremetal Verification (Working Features)

### 2.1 Compute Nodes Status
Both compute nodes (`node-01` and `node-02`) registered with the control plane and reported `state: "ready"` with Firecracker v1.15.1.

```bash
curl -s http://103.174.50.21:8080/nodes/list | jq .
```

![Compute Nodes Status](./test_evidence/bm_nodes_ready.png)

---

### 2.2 Account & VPC Creation
Account creation allocated an isolated VPC (`vpc-08aeb8c0`) and Geneve VNI 100.

```bash
curl -s -X POST http://103.174.50.21:8080/accounts/create \
  -H 'Content-Type: application/json' \
  -d '{"name":"iftakhar-bm-test","email":"iftakhar@poridhi.io"}' | jq .
```

![Account Creation](./test_evidence/bm_account_create.png)

---

### 2.3 MicroVM Launch and Execution
The microVM (`iftakhar-bm-vm1`) launched on Node-01 with private IP `10.0.0.1`, tap device `fc-65231f3f`, and host PID `3869818`.

```bash
curl -s http://103.174.50.21:8080/vms/65231f3faf6a46be929a589c130d777b | jq .
```

![MicroVM Running](./test_evidence/bm_vm_running.png)

---

### 2.4 SSH Metadata & VPC Namespace
Connection info shows the microVM mapped into network namespace `ns-vpc-vpc-08aeb8c0`.

```bash
curl -s http://103.174.50.21:8080/vms/65231f3faf6a46be929a589c130d777b/ssh | jq .
```

![SSH Metadata](./test_evidence/bm_vm_ssh_info.png)

---

### 2.5 In-Guest Network Ping
Pinged the microVM from Node-01 inside the VPC namespace across the virtual tap interface. Result: 3 packets transmitted, 3 received, 0% packet loss, 0.34ms round-trip latency.

```bash
ssh root@54.38.94.139 "ip netns exec ns-vpc-vpc-08aeb8c0 ping -c 3 10.0.0.1"
```

![In-Guest Ping Success](./test_evidence/bm_in_guest_ping_success.png)

---

## 3. Bugs Found on Baremetal

### Bug 1: MicroVM snapshot fails with Jailer PID mismatch
- **What happened:** Taking a snapshot of a running VM fails. The orchestrator activity `CreateSnapshotOnNode` verifies process health by checking `/proc/<pid>/cmdline`. However, the PID stored in etcd (`3869818`) is the parent Jailer wrapper process, not the child Firecracker daemon running inside the chroot jail.
- **Command run:**
  ```bash
  curl -s http://103.174.50.21:8080/vms/65231f3faf6a46be929a589c130d777b/snapshots | jq .
  ```
- **Terminal output:**
  ```json
  "error": "activity error (type: CreateSnapshotOnNode...): rpc error: code = FailedPrecondition desc = vm 65231f3faf6a46be929a589c130d777b: pid 3869818 is not this VM's firecracker process"
  ```
- **Screenshot:**

![Bug 1 - Snapshot Failure](./test_evidence/bug_bm_01_snapshot_pid_failure.png)

- **Fix:** In the node agent launcher, resolve the child Firecracker process ID from `/proc/<jailer_pid>/task/` or the cgroup process list before saving it to etcd.

---

### Bug 2: Mutating non-existent VM returns 202 Accepted
- **What happened:** Sent a restart request with a fake VM ID (`00000000000000000000000000000000`). Instead of returning 404 Not Found, the API responded with 202 Accepted and started a Temporal workflow run.
- **Command run:**
  ```bash
  curl -s -X POST http://103.174.50.21:8080/vms/00000000000000000000000000000000/restart | jq .
  ```
- **Terminal output:**
  ```json
  {
    "operation_id": "vm/00000000000000000000000000000000/restart",
    "run_id": "01a09a8c-6a31-7d8e-b6ac-9b0be550b53f",
    "state": "restarting",
    "vm_id": "00000000000000000000000000000000"
  }
  ```
- **Screenshot:**

![Bug 2 - Fake VM 202 Accepted](./test_evidence/bug_bm_02_fake_vm_202_restart.png)

- **Fix:** In `internal/api/vm_restart.go` and `vm_terminate.go`, add a synchronous `h.store.GetVM(ctx, vmID)` check before executing the Temporal workflow.

---

### Bug 3: Duplicate accounts allowed with same email (Geneve VNI leak)
- **What happened:** Called `POST /accounts/create` twice with the same email (`mytest@example.com`). Both requests succeeded and created two separate accounts (`a23535f2...` and `f15235d1...`), allocating two separate Geneve VNIs (103 and 104).
- **Command run:**
  ```bash
  curl -s -X POST http://103.174.50.21:8080/accounts/create \
    -H 'Content-Type: application/json' \
    -d '{"name":"dup-user","email":"mytest@example.com"}' | jq .
  ```
- **Screenshot:**

![Bug 3 - Duplicate Account VNI Leak](./test_evidence/bug_bm_03_duplicate_account_vni_leak.png)

- **Fix:** Maintain a unique key in etcd at `/accounts-by-email/<email>` using a transaction so subsequent registrations return 409 Conflict.

---

### Bug 4: Field name mismatch between Create and Get account endpoints
- **What happened:** `POST /accounts/create` returns the field as `"account_id"`, but `GET /accounts/:id` returns it as `"id"`. This causes parsing issues in client SDKs.
- **Command run:**
  ```bash
  curl -s http://103.174.50.21:8080/accounts/08aeb8c08fc8d0526b5cd2390df9ca2c | jq .
  ```
- **Terminal output:**
  ```json
  {
    "id": "08aeb8c08fc8d0526b5cd2390df9ca2c",
    "name": "iftakhar-bm-test",
    "email": "iftakhar@poridhi.io",
    "vpc_id": "vpc-08aeb8c0"
  }
  ```
- **Screenshot:**

![Bug 4 - Account ID Field Mismatch](./test_evidence/bug_bm_04_account_id_vs_id_divergence.png)

- **Fix:** Update the JSON tags or struct in `account_get.go` to emit both `id` and `account_id`.

---

### Bug 5: GET /accounts and GET /vms return 404
- **What happened:** Standard REST calls `GET /accounts` and `GET /vms` return 404 page not found. The API router only registers `/accounts/list` and `/vms/list`.
- **Command run:**
  ```bash
  curl -s http://103.174.50.21:8080/accounts; echo ''
  curl -s http://103.174.50.21:8080/vms; echo ''
  ```
- **Terminal output:**
  ```text
  404 page not found
  404 page not found
  ```
- **Screenshot:**

![Bug 5 - Route 404](./test_evidence/bug_bm_05_route_404_divergence.png)

- **Fix:** In `internal/api/handler.go`, register route aliases for `/accounts` and `/vms`.

---

### Bug 6: Invalid image ID allocates IP and MAC before failing
- **What happened:** When requesting a VM with a non-existent `image_id` (`img-nonexistent`), the API allocates a private IP (`10.0.0.2`) and MAC address, returning 202 Accepted. The workflow fails asynchronously a moment later and marks the VM as `terminated`, leaving an allocated IP.
- **Command run:**
  ```bash
  curl -s http://103.174.50.21:8080/vms/6f7064dfd0f8a9b6e8f20d947e63b184 | jq .
  ```
- **Terminal output:**
  ```json
  {
    "vm_id": "6f7064dfd0f8a9b6e8f20d947e63b184",
    "image_id": "img-nonexistent",
    "private_ip": "10.0.0.2",
    "mac_addr": "6E:70:64:DF:D0:F8",
    "state": "terminated"
  }
  ```
- **Screenshot:**

![Bug 6 - Premature IP Allocation](./test_evidence/bug_bm_06_invalid_img_premature_ip_lease.png)

- **Fix:** In `vm_create.go`, validate `image_id` against etcd before invoking the IPAM lease allocator.

---

## 4. Local Testing & Boundary Notes

- In the local dev environment, `POST /vms/create` returns `503 Service Unavailable: orchestrator not configured`. This is normal behavior because creating TAP devices and Jailer chroots requires Linux root privileges and `/dev/kvm`, which are only available on the baremetal servers.
- Local E2E runner execution:

![Local E2E Run](./test_evidence/local_e2e_flow_run.png)

- Concurrent user simulation run (8 parallel users):

![Concurrent Simulation Run](./test_evidence/local_concurrent_sim_run.png)

---

## 5. Summary of Issues

| Ref | Type | Severity | Description | Fix Location |
| :---: | :---: | :---: | :--- | :--- |
| **BM-01** | Bug | High | VM snapshot fails due to Jailer wrapper PID tracking | `cloud-engine/internal/agent/orchestrator.go` |
| **BM-02** | Bug | Medium | Restarting fake VM returns 202 instead of 404 | `cloud-engine/internal/api/vm_restart.go` |
| **BM-03** | Bug | Medium | Duplicate account creation with same email leaks VNIs | `cloud-engine/internal/store/account.go` |
| **BM-04** | Bug | Low | Field name mismatch (`account_id` vs `id`) | `cloud-engine/internal/api/account_get.go` |
| **BM-05** | Bug | Low | Missing standard REST routes `/accounts` and `/vms` | `cloud-engine/internal/api/handler.go` |
| **BM-06** | Bug | Medium | Invalid image ID reserves private IP before failing | `cloud-engine/internal/api/vm_create.go` |
| **LOC-01** | Code Review | High | Non-atomic `SetVMState` read-modify-write in etcd | `cloud-engine/internal/store/vm.go:71` |
| **LOC-02** | Perf | High | Linear CAS retry loop in `AllocateVNI` degrades under concurrency | `cloud-engine/internal/store/ipam.go` |
| **LOC-03** | Config | High | Missing `TEMPORAL_PAYLOAD_KEY` in `.env.example` | `cloud-engine/.env.example` |
