# RSVP-TEE-with-Planned-works
RSVP TE ++


Here’s a `README.md` you can drop straight into your GitHub repo and tweak as you like:

````markdown
# Junos RSVP-TE++ Container LSP Lab

This lab demonstrates how to use **RSVP-TE container LSPs** (aka **RSVP-TE++**) on Junos to:

- Build a **container LSP** from `vMX102` → `vMX105`.
- Dynamically **split and merge** multiple RSVP LSPs based on aggregate bandwidth.
- Drive ~**1 Mbps** of traffic through the container and observe per-LSP statistics.
- Use **admin-groups** (`PLANNED-WORK`) to steer LSPs away from a “bad/maintenance” link via an alternate path (`vMX106`) without touching IGP metrics.

---

## 1. Topology

High-level view:

```text
   vMX101      vMX103      vMX104
      \          |          /
       \         |         /
        \        |        /
         (core IS-IS / RSVP-TE ring)
                  |
                vMX102 (Ingress)
        ge-0/1/2.0      ge-0/1/3.0
            |                \
            |                 \
   [PLANNED-WORK]              vMX106
   direct 102–105 link           |
            |                    |
          vMX105 (Egress)--------+
               |
        172.16.105.1/32 (loopback)
               |
           Linux `trafgenu`
           172.16.22.2/24
````

Traffic flow for the demo:

* Source: Linux `trafgenu` (behind `vMX102`)
* Destination: `172.16.105.1/32` loopback behind `vMX105`
* Transport: RSVP-TE++ container LSP `TEPP-102-105` (from `vMX102` to `vMX105`)

---

## 2. Building Blocks

### 2.1 Auto-bandwidth LSP Template

The point-to-point LSP template defines **adaptive auto-bandwidth** behaviour for each member LSP.

```bash
set protocols mpls label-switched-path AUTOBW-template template
set protocols mpls label-switched-path AUTOBW-template retry-timer 300
set protocols mpls label-switched-path AUTOBW-template random
set protocols mpls label-switched-path AUTOBW-template adaptive

set protocols mpls label-switched-path AUTOBW-template auto-bandwidth adjust-interval 300
set protocols mpls label-switched-path AUTOBW-template auto-bandwidth adjust-threshold 20
set protocols mpls label-switched-path AUTOBW-template auto-bandwidth minimum-bandwidth 100k
set protocols mpls label-switched-path AUTOBW-template auto-bandwidth maximum-bandwidth 400k
set protocols mpls label-switched-path AUTOBW-template auto-bandwidth adjust-threshold-overflow-limit 3
set protocols mpls label-switched-path AUTOBW-template auto-bandwidth adjust-threshold-underflow-limit 3
```

**Effective behaviour (per member LSP):**

* Min bandwidth: **100 kbps**
* Max bandwidth: **400 kbps**
* Auto-bandwidth recompute every **300 s**, with 3 samples hysteresis for both overflow & underflow.

---

### 2.2 Container LSP (RSVP-TE++)

The container LSP `TEPP-102-105` from `vMX102` to `vMX105` uses the template above and is allowed to **split** into multiple member LSPs as aggregate traffic grows.

```bash
set protocols mpls container-label-switched-path TEPP-102-105 label-switched-path-template AUTOBW-template
set protocols mpls container-label-switched-path TEPP-102-105 to 11.0.0.105

set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging maximum-member-lsps 4
set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging minimum-member-lsps 1

set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging splitting-bandwidth 300k
set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging merging-bandwidth 150k
set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging maximum-signaling-bandwidth 200k
set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging minimum-signaling-bandwidth 100k

set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging normalization normalize-interval 300
set protocols mpls container-label-switched-path TEPP-102-105 splitting-merging normalization normalization-retry-duration 3600
```

**What this does:**

* Container starts with **1 member LSP**, can grow to **4**.
* If aggregate BW > **300 kbps**, it can **split** and create extra member LSPs.
* If aggregate BW < **150 kbps**, it can **merge** and reduce the number of LSPs.
* Each member LSP is signalled with 100–200 kbps, keeping per-LSP BW small and balanced.

Example output when pushing ~1 Mbps:

```text
show mpls container-lsp statistics detail ingress

Container LSP name: TEPP-102-105, State: Up, Member count: 3
 Aggregate bandwidth: 542.496kbps, Sampled Aggregate bandwidth: 970.545kbps
 ...
  Normalize: container (TEPP-102-105) into 3 members - each with bandwidth ~180kbps
```

---

### 2.3 Steering Traffic into the Container LSP

On `vMX102`, a static route points the destination loopback into the RSVP-TE++ container.

> Note: adjust to your exact addressing; below is an example pattern.

```bash
set routing-options static route 172.16.105.1/32 \
    qualified-next-hop 11.102.105.2 interface ge-0/1/2.0 \
    lsp-next-hop TEPP-102-105-1
```

From the Linux host you can verify reachability:

```bash
traceroute 172.16.105.1
# Example:
#  1  172.16.22.102
#  2  11.102.105.x / 11.102.106.x
#  3  172.16.105.1
```

---

## 3. Generating Traffic (iperf3)

On the Linux `trafgenu` host:

```bash
apt update
apt install -y iperf3

# ~1 Mbps UDP stream towards vMX105 loopback
iperf3 -c 172.16.105.1 -u -b 1M -l 1200 -t 300
```

This drives user traffic over the container LSP, giving RSVP-TE++ enough load to split into multiple member LSPs.

---

## 4. Verification

### 4.1 Container LSP membership & stats

On `vMX102`:

```bash
show mpls container-lsp
show mpls container-lsp statistics
show mpls container-lsp statistics detail ingress
```

Sample (shortened):

```text
Ingress LSP: 1 sessions
Container LSP name: TEPP-102-105, State: Up, Member count: 3
 Aggregate bandwidth: 542.496kbps, Sampled Aggregate bandwidth: 970.545kbps

11.0.0.105
  LSPname: TEPP-102-105-1, Statistics: Packets X, Bytes Y
11.0.0.105
  LSPname: TEPP-102-105-2, Statistics: Packets X, Bytes Y
11.0.0.105
  LSPname: TEPP-102-105-3, Statistics: Packets X, Bytes Y
```

This confirms:

* The container **split** into multiple LSPs.
* Traffic counters are incrementing per LSP.

---

### 4.2 MPLS data-plane

On `vMX102`:

```bash
monitor traffic interface ge-0/1/2.0 no-resolve
# or ge-0/1/3.0 depending on which link the LSP follows
```

You should see MPLS packets with different labels (per member LSP) and RSVP/LDP/IS-IS control traffic.

---

### 4.3 RSVP path check

From `vMX102`:

```bash
traceroute mpls rsvp TEPP-102-105-1 detail
```

Example (via `vMX106`):

```text
Hop 11.102.106.2 Depth 1
  Return code: Label switched at stack-depth 1
  Label 1 Value 161 Protocol RSVP-TE

Hop 11.105.106.1 Depth 2
  Return code: Egress-ok at stack-depth 1
  Label 1 Value 3 Protocol RSVP-TE

Path 1 via ge-0/1/3.0 destination 127.0.0.64
```

---

## 5. “Planned Work” – Steering Away from a Bad Link

To simulate maintenance on the direct `vMX102`–`vMX105` link, we mark it with an admin-group `PLANNED-WORK` and ensure CSPF avoids it.

### 5.1 Admin-group definition & tagging

On `vMX102`:

```bash
set protocols mpls admin-group PLANNED-WORK 0

# Direct 102–105 link
set protocols mpls interface ge-0/1/2.0 admin-group PLANNED-WORK

# Other links can have different groups as needed, e.g.
set protocols mpls interface ge-0/1/3.0 admin-group blue
```

### 5.2 CSPF avoiding PLANNED-WORK

In this lab, the LSP constraints were set such that **any link tagged with `PLANNED-WORK` is excluded** by CSPF.
(Use your preferred method: named path with `admin-group exclude PLANNED-WORK`, or TE policy, etc.)

Once the change is advertised into the TE database:

* CSPF is re-run for each `TEPP-102-105-*` member LSP.
* Paths are recomputed via `vMX106` instead of the direct 102–105 link.

You can see this in the logs (`tail -f /var/log/mpls` or `show log mpls`):

```text
CSPF for path TEPP-102-105-1(primary ) ... 
  mpls lsp TEPP-102-105-1 primary CSPF: computation result accepted 11.102.106.2 11.105.106.1
```

And in the data plane:

* **IP traceroute** from `trafgenu` now shows:

  ```text
  traceroute 172.16.105.1

  1  172.16.22.102
  2  11.102.106.2
  3  172.16.105.1
  ```

* **MPLS traceroute** shows the RSVP LSP via `vMX106` as above.

This proves that **TE-level constraints (admin-groups)** can be used to steer traffic away from a “bad/maintenance” link *without* changing IGP metrics.

---

## 6. Useful Show Commands

Quick reference:

```bash
# Container and member LSPs
show mpls container-lsp
show mpls container-lsp statistics detail ingress

# Individual RSVP LSP details
show mpls lsp name TEPP-102-105-1 extensive

# MPLS traceroute for RSVP LSP
traceroute mpls rsvp TEPP-102-105-1 detail

# IP forwarding & static route resolution
show route 172.16.105.1 detail

# Data-plane sniff
monitor traffic interface ge-0/1/2.0 no-resolve
monitor traffic interface ge-0/1/3.0 no-resolve
```

---

## 7. Next Steps / Ideas

* Scale up more member LSPs (e.g. `maximum-member-lsps 8`) and push higher bandwidth to see how the container reacts.
* Combine RSVP-TE++ with **DS-TE** / per-class TE constraints.
* Export telemetry (gNMI, sFlow, inline-jflow) for container/member LSP statistics into Prometheus / InfluxDB + Grafana.
* Extend the lab to compare behaviour vs. **SR-TE / SPRING-TE** under similar admin-group constraints.

---

```

If you want, I can also add a short “Story”/Background section at the top explaining *why* you built this lab (Vodafone use-case / RSVP-TE++ interop) to make the repo more narrative and recruiter-friendly.
::contentReference[oaicite:0]{index=0}
```
