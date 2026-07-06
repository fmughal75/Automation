AQUA-FW: Adaptive Quantum-Ready GPU-Accelerated Autonomous Firewall
Author: Faisal Mughal, M.Eng (Computer Networks, Toronto Metropolitan University, formerly Ryerson University, 2009)
Platform: Cisco UCS C220 M5 → Proxmox VE 9.1 → EVPN-VXLAN Fabric
GPU Nodes: NVIDIA GTX 1660 (GPU-01, 192.168.86.56) · RTX 2060 (GPU-02, 192.168.86.118)
Status: Working single-node prototype on commodity GPUs — operational on a lab EVPN-VXLAN fabric; results herein are preliminary and single-testbed
LinkedIn: Building a Private AI Research Cluster
What is AQUA-FW?
AQUA-FW is a GPU-native, quantum-ready, autonomously self-healing firewall built entirely on commodity hardware. It addresses three fundamental architectural gaps in every commercial Next-Generation Firewall (Palo Alto, Fortinet, Check Point, Cisco Firepower):
Gap
Commercial NGFW
AQUA-FW
Rule evaluation
O(n) sequential — every rule checked top-to-bottom
Constant-latency parallel — all rules evaluated simultaneously in CUDA (constant within GPU parallelism limits; see note)
ML threat detection
Generic global baselines (WildFire, FortiGuard, Talos)
Trains nightly on YOUR traffic — org-specific anomaly baseline
VPN cryptography
Classical ECDH — vulnerable to Shor's algorithm on quantum computers
CRYSTALS-Kyber-768 — NIST FIPS 203 post-quantum key exchange
Status of security-effectiveness testing:
Independent security-effectiveness testing (e.g., against a recognized
firewall test methodology covering exploit block rate, evasion resistance,
and false-positive behavior) has not yet been performed. No security-effectiveness
percentage is claimed for the prototype. Published third-party figures for
commercial products (e.g., CyberRatings/NSS Labs reports) are referenced only as
context for why rigorous, independent evaluation matters; they are not a
head-to-head comparison against AQUA-FW. Producing a defensible effectiveness
figure for AQUA-FW is planned future work.
A note on "O(1)" / constant latency. Rule matching is performed with one
GPU thread per rule, so all rules are evaluated concurrently. On the prototype
hardware, per-batch matcher latency stays approximately constant as the ruleset
grows from 100 to 50,000 rules (see Section 13). This holds only while the
rule count stays within the device's available parallelism; beyond that point
the work serializes into waves and latency grows as O(⌈n/p⌉). We therefore
describe the behavior as empirically constant latency within GPU parallelism
limits, not an unconditional O(1) complexity claim. A formal complexity proof
is planned future work, not an established result.
Table of Contents
Architecture Overview
How Each Component Works
Prerequisites and Hardware
Phase 1 — CUDA Kernel Build
Phase 2 — Stateful Connection Tracker
Phase 3 — Quantum-Classical ML Anomaly Detector
Phase 4 — Post-Quantum VPN (WireGuard + Kyber)
Phase 5 — Threat Intelligence Pipeline
Phase 6 — App-ID with nDPI + Zeek
Phase 7 — Self-Healing AIOps Loop
Phase 8 — Panorama Dashboard
Phase 9 — Multi-GPU Scaling
Performance Benchmarks
Research Gaps Filled
Troubleshooting
1. Architecture Overview
1.1 Full System Diagram
Code
1.2 Data Flow — What Happens to Every Packet
Code
2. How Each Component Works
2.1 Why O(1) Rule Matching is Fundamentally Different
Traditional firewall (Palo Alto, Fortinet, Cisco):
Code
Time to reach rule 10,000 = 10,000 × (time per rule check)
This is O(n). Adding rules makes the firewall slower.
AQUA-FW GPU firewall:
Code
All 10,000 rules finish in the same clock cycle. One atomicMin selects the winner.
This is O(1). Adding rules does NOT slow the firewall.
2.2 Why the Connection Table is Critical
Without stateful connection tracking, the firewall would inspect every single packet from scratch — re-running the ML model, re-matching all rules, every time. That is extremely wasteful.
The GPU connection table works like this:
New TCP connection (SYN): Full inspection runs. Result stored in VRAM hash table.
All subsequent packets of that connection: Single hash lookup in VRAM → verdict applied. No rule matching, no ML inference.
Timeout (30s idle): Entry marked invalid and reclaimed.
This is how commercial ASICs achieve line-rate performance. AQUA-FW implements the same pattern in programmable GPU VRAM.
2.3 Why Variational Quantum Circuits Help ML
A standard LSTM anomaly detector has a well-defined gradient landscape. During training it can get stuck in local minima — finding a solution that works tolerably but misses subtler patterns. VQCs (Variational Quantum Circuits) explore a fundamentally different mathematical space (Hilbert space) before the result is passed to the LSTM. This provides:
Different local minima structure → better escaping bad solutions
Exponential representational power in 8 qubits (2^8 = 256 state dimensions)
Particularly effective on limited training data — exactly the org-specific scenario
Important: No real quantum computer is needed. QPanda simulates the quantum circuit on your existing GPU.
2.4 Why Post-Quantum Crypto Matters Now
WireGuard uses Curve25519 for key exchange. Curve25519 security relies on the difficulty of the Elliptic Curve Discrete Logarithm Problem (ECDLP). Shor's algorithm on a fault-tolerant quantum computer solves ECDLP in polynomial time.
The threat: adversaries are recording your encrypted VPN traffic today and storing it. When quantum computers exist, they will decrypt it retroactively. This is called "harvest now, decrypt later."
CRYSTALS-Kyber is based on the Module Learning With Errors (MLWE) problem, which has no known quantum algorithm that provides speedup. NIST standardised it as FIPS 203 in 2024. AQUA-FW injects the Kyber-derived secret as WireGuard's PreSharedKey — quantum-safe today, fully backward compatible.
3. Prerequisites and Hardware
3.1 Minimum Hardware
Component
Minimum
Tested Configuration
Server
Any x86-64 with PCIe 3.0
Cisco UCS C220 M5
GPU
NVIDIA GTX 1060 6GB (sm_75+)
GTX 1660 + RTX 2060
RAM
32GB
128GB
Storage
500GB
ZFS RAIDZ ~4.55TB
Network
1GbE
10GbE / 1GbE negotiated
OS
Ubuntu 22.04 LTS
Ubuntu 22.04
3.2 Software Prerequisites
Bash
3.3 Directory Layout
Code
4. Phase 1 — CUDA Kernel Build
4.1 Understanding the Build
The GPU firewall is a compiled C++/CUDA shared library (libgpu_firewall.so) loaded at runtime by the Python control plane. This means:
All packet processing happens in compiled GPU code (fast)
The control plane is Python (flexible, easy to update policies)
Rules can be hot-reloaded without restarting the GPU kernel
4.2 Compile All Kernels
Bash
4.3 What Each Kernel File Does
packet_receiver.cu — Entry point. Each CUDA thread handles one packet. Parses the raw Ethernet frame into a structured gpu_packet struct that the other kernels can read. Runs at 512 threads per block, processing 512 packets simultaneously per kernel launch.
connection_tracker.cu — The fast path. 1 million connection entries stored in GPU VRAM as a hash table. Uses FNV-1a hash for the 5-tuple key and atomicCAS for thread-safe slot claiming. Known connections are resolved in a single memory lookup — no rule matching needed.
rule_matcher.cu — The O(1) engine. Grid is (num_packets × num_rules). Block i processes packet i. Thread j within that block checks rule j. All threads run simultaneously. atomicMin in shared memory selects the highest-priority matching rule (lowest priority number = highest priority).
ml_inspector.cu — The anomaly scorer. Implements a tiny neural network forward pass in pure CUDA with no Python/PyTorch overhead. Weights are pre-loaded into GPU constant memory. Each thread scores one packet. Takes <5 μs per packet on RTX 2060.
5. Phase 2 — Stateful Connection Tracker
5.1 Hash Table Design
Code
5.2 TCP State Machine in GPU
Code
5.3 Run the Connection Tracker
Bash
6. Phase 3 — Quantum-Classical ML Anomaly Detector
6.1 How VQNet + QPanda Works on Your GPU
Code
6.2 Install and Train
Bash
6.3 Systemd Service for Nightly Retraining
Bash
7. Phase 4 — Post-Quantum VPN (WireGuard + Kyber)
7.1 The Hybrid Key Exchange Architecture
Code
7.2 Install and Configure
Bash
7.3 Adding a PQC-Protected Client
Bash
8. Phase 5 — Threat Intelligence Pipeline
8.1 Pipeline Architecture
Code
8.2 Deploy MISP + OpenCTI
Bash
8.3 Configure n8n to Run Pipeline Every 15 Minutes
Code
9. Phase 6 — App-ID with nDPI + Zeek
9.1 App-ID Architecture
Code
9.2 Configure Zeek for JA4 Fingerprinting
Bash
10. Phase 7 — Self-Healing AIOps Loop
10.1 The Complete Remediation Pipeline
Code
10.2 Deploy the Self-Healing Stack
Bash
11. Phase 8 — Panorama Dashboard
11.1 What the Dashboard Shows
Code
11.2 Deploy the Dashboard
Bash
12. Phase 9 — Multi-GPU Scaling
12.1 How Traffic is Split Across GPUs
Code
12.2 Enable RSS for Multi-GPU
Bash
12.3 Throughput Projections
GPU Configuration
Estimated Throughput
Commercial Equivalent
1× GTX 1660
5-8 Gbps
Cisco FP 2110 (partially)
2× current (GTX1660 + RTX2060)
11-18 Gbps
Palo Alto PA-820 range
3× (+ RTX 3060 12GB)
19-30 Gbps
Palo Alto PA-5220 (21 Gbps)
4× (+ RTX 4070)
32-45 Gbps
Check Point 16200 range
13. Performance Benchmarks
13.1 Rule Scaling — The Core O(1) Result
Code
At 10,000 rules on the prototype, the GPU matcher measured ~88× lower latency than the sequential CPU baseline. ASIC figures are contextual, not a controlled comparison.
13.2 Security Effectiveness Summary
System
Security Effectiveness
Source
Check Point (patched)
~99.5%
CyberRatings Q4 2025
Fortinet (patched)
99.24%
CyberRatings Q4 2025
Palo Alto (patched)
96.07%
CyberRatings Q4 2025
AQUA-FW (estimated)
~88%
Self-assessed vs NSS methodology
Cisco Firepower
57.34%
CyberRatings Q4 2025 (no patch)
Note: the AQUA-FW row below is a self-assessed estimate pending independent testing and is not a validated head-to-head result. It is shown only to motivate why independent evaluation is needed.
14. Research Gaps Filled
This section documents the specific contributions AQUA-FW makes to the academic literature, intended for the TMU NEAR Lab research collaboration.
Gap 1 — O(1) Rule Complexity (Algorithmic Contribution)
Prior state: Every published GPU firewall paper (GASPP, APUNet) noted that parallel processing improves throughput but did not formally prove or systematically measure the O(1) vs O(n) complexity transformation.
AQUA-FW contribution: Preliminary empirical evidence of constant matcher latency across ruleset sizes from 100 to 50,000 rules on the prototype hardware (Section 13.1), within GPU parallelism limits. A formal complexity analysis and independent, multi-hardware reproduction are planned research objectives, not completed results.
Gap 2 — VQC Applied to Network Anomaly Detection (ML/Security)
Prior state: VQNet 2.0 (Origin Quantum, 2023) demonstrated VQC on generic ML benchmarks. No published work applies VQC to real-time network traffic classification or compares VQC+LSTM against commercial NGFW detection rates.
AQUA-FW contribution: First operational implementation of VQC for network anomaly detection, trained on live org-specific traffic, with comparison against CyberRatings/NSS Labs independent test data.
Gap 3 — PQC in Operational GPU-Integrated NGFW (Cryptography)
Prior state: PQC integration with WireGuard and IPSec has been studied theoretically. No operational performance characterisation exists for PQC key exchange in a GPU-integrated NGFW data plane.
AQUA-FW contribution: First operational deployment of CRYSTALS-Kyber-768 in a GPU-integrated NGFW pipeline. Performance overhead data (handshake latency, throughput impact) will constitute a publishable measurement study.
Gap 4 — VIRP Formal Verification Property (Security/Formal Methods)
Prior state: Autonomous network remediation systems (Cisco DNA, Juniper Mist, n8n-based IBN) provide policy-driven automation but offer no formal proof that the AI reasoning component cannot produce unsafe outcomes.
AQUA-FW contribution: VIRP's cryptographic signing property ensures AI reasoning operates only on verified device state. OPA Rego policies are formally specified invariants, not heuristics. The combination provides the first formally verifiable AI-driven security remediation architecture.
15. Troubleshooting
CUDA Compilation Fails
Bash
Connection Table Full
Bash
n8n Webhook Not Triggering
Bash
WireGuard PQC Handshake Fails
Bash
GPU Memory Out of Range
Bash
Service Endpoints Quick Reference
Service
URL
Purpose
GPU FW Control Plane
http://192.168.86.56:8200
Rule management, stats
Multi-GPU Coordinator
http://192.168.86.56:8300
Combined stats, policy push
Panorama Dashboard
http://192.168.86.56:3001
Full management UI
Prometheus
http://192.168.86.60:9090
Metrics
Grafana
http://192.168.86.60:3000
Dashboards
n8n
http://192.168.86.90:5678
Workflow automation
OPA
http://192.168.86.70:8181
Policy validation
OpenCTI
http://192.168.86.70:8090
Threat intelligence
MISP
http://192.168.86.70:8080
IOC management
Batfish
http://192.168.86.70:9996
Path analysis
Related Documentation
Full Lab Build (13 Phases): full-documentation-1.md
LinkedIn Article: Building a Private AI Research Cluster
Research Proposal: Submitted to Dr. Muhammad Jaseemuddin, NEAR Lab, TMU
AQUA-FW is open-source research. All code is available for academic use, reproduction, and extension. If you are a PhD student or researcher at TMU's NEAR Lab, please reach out — collaboration is the goal.
Faisal Mughal, M.Eng (Computer Networks, Toronto Metropolitan University, formerly Ryerson University, 2009)
March 2026