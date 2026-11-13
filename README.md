# Hyperscaler Packet Journey & Chaos Simulator

An interactive, single-page network engineering simulator (`hyperscaler_simulator.html`) that visualizes how a packet travels from a client laptop across the public internet, through distributed Edge Points of Presence (PoPs) and a private optical WAN backbone, down into a regional data center's folded Clos fabric and compute pod.

Built with zero external build dependencies (self-contained HTML5, SVG, Tailwind CSS, and vanilla JavaScript), the simulator lets you trace individual requests hop-by-hop, inspect on-the-wire header encapsulations, and inject real-world infrastructure failures to observe automated control-plane recovery.


![Description of the screenshot](logrok-screenshot01.png)

---

## Quick Start

Because the simulator is packaged as a single self-contained HTML file, no compilation or backend server is required:

1. **Direct Browser Open:** Double-click `hyperscaler_simulator.html` or open it directly in any modern browser (Chrome, Firefox, Edge, Safari).
2. **Optional Local HTTP Server:**
   ```bash
   python3 -m http.server 8080
   # Navigate to http://localhost:8080/hyperscaler_simulator.html
   ```

---

## End-to-End Architecture (The 7-Stage Pipeline)

The simulator models a complete hyperscaler ingress and return path (`Client` $\leftrightarrow$ `Anycast VIP 198.51.100.10/32` $\leftrightarrow$ `Backend App Pod 10.200.1.10`):

```text
[ Client Laptop ] ──▶ [ Public Internet IXP (BGP Anycast AS64496) ]
                             │
               ┌─────────────┴─────────────┐
               ▼ (Primary)                 ▼ (Standby / Failover)
     [ Edge PoP A: Chicago ]     [ Edge PoP B: New York ]
       ├─ 1. Border Router (eBGP)
       ├─ 2. DDoS Scrubber (BGP Flowspec RFC 8955 / TCAM)
       ├─ 3a. L4 Load Balancer (Maglev / XDP 5-Tuple Hash)
       └─ 3b. L7 Edge Proxies (Envoy TLS 1.3 Termination)
               │
               ▼
     [ 4. Private Optical WAN Core ] (SRv6 Traffic Engineering)
               │
               ▼
     [ 5. Data Center Border Leaf / Gateway ] (SRv6 Pop → VXLAN VNI 100)
               │
         ┌─────┴─────┐
         ▼           ▼
     [ 6. Clos Spine 1 & Spine 2 ] (eBGP Unnumbered RFC 5549 • ECMP)
         │           │
         └─────┬─────┘
               ▼
     [ 7. ToR Leaf VTEP & App Pod ] (VXLAN Decap → Container veth 10.200.1.10)
```

### Stage-by-Stage Encapsulation & Protocol Reference

| Stage | Component | Key Protocols & RFCs | Data-Plane & Header Action |
| :--- | :--- | :--- | :--- |
| **Step 1** | **Edge Border Router** (`br-ord-01`) | eBGP Anycast (RFC 4786) | Attracts client traffic to the topologically nearest PoP via shortest BGP AS-Path (`203.0.113.42` $\rightarrow$ `198.51.100.10`). |
| **Step 2** | **DDoS Scrubbing & ACLs** | BGP Flowspec (RFC 8955), uRPF, TCAM | Evaluates ingress traffic against line-rate hardware TCAM filters; drops volumetric attack flows (`traffic-rate 0`) while passing `TCP/443`. |
| **Step 3a** | **Stateless L4LB** | Maglev / Katran / eBPF XDP | Computes a consistent 5-tuple hash and encapsulates packets (GUE/GRE) to a healthy L7 reverse proxy without breaking active TCP sessions. |
| **Step 3b** | **L7 Edge Proxy** | Envoy, TLS 1.3, HTTP/2 gRPC | Terminates client TLS locally at the metro edge, injects `X-Forwarded-For` and W3C `traceparent` headers, and forwards over a pooled backend session. |
| **Step 4** | **Optical WAN Backbone** | Segment Routing IPv6 (SRv6 TE) | Pushes an IPv6 Segment Routing Header (`fc00:dc1:gateway::1`) to steer traffic across private long-haul fiber with deterministic latency. |
| **Step 5** | **DC Border Leaf / Gateway** | BGP EVPN, VXLAN (RFC 7348) | Pops the WAN SRv6 header and encapsulates the tenant packet into VXLAN (`UDP 4789`, `VNI 100`) addressed to the target ToR Leaf VTEP (`10.0.0.21`). |
| **Step 6** | **Folded Clos Spines** | eBGP (RFC 7938), RFC 5549, BFD | Routes purely on the outer underlay IPv4 header using Equal-Cost Multi-Path ($\text{Hash} \pmod N$) across Spines 1 and 2 (`AS 65200`). |
| **Step 7** | **ToR Leaf VTEP & Compute Pod** | VTEP Decap, Anycast GW, Linux `veth` | Strips outer Ethernet/IP/UDP/VXLAN headers and delivers the inner HTTP/2 request to the Kubernetes pod (`10.200.1.10`), which returns `HTTP 200 OK`. |

---

## Interactive Features

### 1. Simulation & Packet Tracing Controls
* **Fire Traced Packet:** Launches a highlighted `GET /api/v1/orders` request from the Client Laptop, automatically advancing the Stage Inspector as it traverses Steps 1–7, and animates the `HTTP 200 OK` response all the way back to the client.
* **Step-by-Step Pause Mode:** When toggled **ON**, any traced packet automatically pauses at each architectural stage so you can inspect the exact encapsulation stack before clicking **Next Hop**.
* **Continuous Traffic Toggle:** Enables or disables background client traffic flows across the topology.
* **Live Telemetry Ribbon:** Tracks real-time Edge Ingress Rate (`Tbps`), Active End-to-End Path, Flowspec Drop Percentage, and End-to-End Round-Trip Time (`ms`).

### 2. 7-Stage Packet Encapsulation & Header Inspector
Click any node on the interactive SVG topology canvas or select any of the **Step 1 – Step 7** tabs in the lower-left deck to view:
* Architectural explanation and protocol standards for that stage.
* Built-in resilience and automated failover mechanics.
* Color-coded **On-the-Wire Protocol Stack** showing where outer headers (`GUE`, `SRv6`, `VXLAN`) are added, routed, or stripped.

### 3. Live BGP & Control-Plane Event Log
The lower-right NOC console streams timestamped control-plane events (`BGP`, `FLOWSPEC`, `MAGLEV`, `ENVOY`, `SRv6-TE`, `BFD`, `VXLAN`, `TRACE`) whenever packets are traced or fault injections trigger reconvergence.

---

## Chaos Engineering & Failure Scenarios

Use the **Chaos Engineering & Automated Failover Controls** panel to test how hyperscaler networks survive component failures without dropping user traffic:

1. **Simulate DDoS Flood (`UDP Port 53 DNS Amplification`):**
   * Launches a `3.4 Tbps` volumetric attack stream from the Botnet node toward the Anycast VIP.
   * After ~1.8 seconds of simulated sFlow anomaly detection, the mitigation controller injects an **RFC 8955 BGP Flowspec** update (`Match [Proto=UDP, DstPort=53] -> traffic-rate 0`), dropping 100% of malicious packets in hardware TCAM at Step 2 while legitimate `TCP/443` traffic flows unharmed.
2. **Edge PoP A Outage (`Chicago ORD Fiber / Power Cut`):**
   * Simulates a complete metro PoP failure. Edge PoP A withdraws its eBGP Anycast announcement (`198.51.100.10/32`), causing public internet IXP routing to converge onto **Edge PoP B (New York JFK)**.
3. **L7 Proxy Crash (`Envoy-1 Worker Panic`):**
   * Marks `Envoy-1` unhealthy in PoP A. **Maglev L4LB** updates its consistent hash lookup ring to steer 100% of incoming flows to `Envoy-2` without resetting connections.
4. **WAN Primary Fiber Cut (`Backhoe Span Outage`):**
   * Simulates Loss of Light (LOS) on the primary optical core link. **SRv6 Traffic Engineering** immediately steers traffic onto the backup optical path (`fc00:dc1::2`).
5. **Clos Spine 1 Outage (`ASIC / BFD Link Down`):**
   * Triggers a sub-second Bidirectional Forwarding Detection (BFD) timeout between the DC Gateway and `Clos Spine 1`, reconverging eBGP ECMP so 100% of VXLAN `VNI 100` traffic traverses `Clos Spine 2`.

---

## Companion Lab Emulation (Containerlab)

To replicate this simulated data plane with real Linux network namespaces, **FRRouting (FRR)**, **IPVS**, and **Envoy**, you can deploy the companion Containerlab topology (`edge-to-core.clab.yml`):

```bash
# Deploy the 10-container end-to-end lab (~500 MB - 1 GB RAM total)
sudo clab deploy -t edge-to-core.clab.yml

# Verify end-to-end HTTP reachability through the Anycast VIP
docker exec -it clab-edge-to-core-client curl -s http://198.51.100.10/

# Verify eBGP ECMP multipathing across Clos Spines on the DC Gateway
docker exec -it clab-edge-to-core-dc-gateway vtysh -c "show ip route 10.200.1.0/24"
```
