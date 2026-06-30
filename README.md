# Nokia Basic DCI Lab — Extended

This repository extends the [Nokia basic DCI lab](https://github.com/srl-labs/nokia-basic-dci-lab) reference topology with a design correction that conditions the default route advertisement in each datacenter gateway on actual WAN reachability, enabling near-seamless failover upon WAN link failure. The extension was developed as part of a master's thesis on decentralised datacenter switching for resilient network fabrics in mission-critical environments, submitted to the University of Amsterdam (OS3) and the Royal Military Academy of Belgium.

## Background

The original Nokia basic DCI lab models two datacenter sites interconnected over an IP/MPLS WAN, using an EVPN-to-IPVPN interworking model. In the original configuration, the default route `0.0.0.0/0` in VPRN 101 on each datacenter gateway is originated from a static blackhole route. This blackhole is unconditionally active regardless of WAN reachability, causing a critical control plane flaw: when a WAN-facing port on a gateway fails, the blackhole continues to be advertised as an EVPN type-5 route into the overlay fabric, causing leaf switches to continue load-balancing cross-datacenter traffic toward the failed gateway via ECMP. This results in persistent ~35% packet loss rather than a clean failover.

This repository corrects that flaw by introducing a dedicated PE node (`p2`) that originates `0.0.0.0/0` via a BGP VPN aggregate route, conditioning the default route advertisement on the availability of the WAN path. The existing `p1` route reflector distributes this route to both datacenter gateways, which re-advertise it into the EVPN overlay. When a WAN link fails, the BGP VPN session drops, the default route is withdrawn from the VPRN routing table, the EVPN type-5 withdrawal propagates to the leaf switches via `rapid-withdrawal`, and all cross-datacenter traffic converges onto the surviving gateway — achieving near-seamless failover with typically 0–1 lost packets at a 10 ms probing interval.

## What changed

The following files differ from the original Nokia basic DCI lab. All other configuration files remain identical to the original repository and should be taken from [srl-labs/nokia-basic-dci-lab](https://github.com/srl-labs/nokia-basic-dci-lab/tree/main/configs).

| File | Change |
|------|--------|
| `dci.clab.yml` | Added `p2` node and link `p2:eth1 ↔ p1:eth3` |
| `configs/p1.partial.cfg` | Added port `1/1/c3`, interface `p2`, IS-IS, LDP and static BGP neighbor `10.0.0.50` |
| `configs/p2.partial.cfg` | New file — dedicated PE node (SR OS) connected to `p1` via IS-IS and LDP; originates `0.0.0.0/0` as an aggregate blackhole in VPRN 101; exports it into VPNv4 with community `target:65100:101` via `vprn101-export` policy; `p1` reflects it to all datacenter gateways |
| `configs/dcgw1-dc1.partial.cfg` | Added `vrf-import` policy restricting BGP VPN installation to `0.0.0.0/0` only, rejecting all other BGP VPN prefixes so DC1 routes are learned exclusively via EVPN-IFF; added `default-preference ibgp 161` to ensure the BGP VPN-learned default route is preferred over any EVPN-IFF re-advertisement of `0.0.0.0/0` from the peer gateway; removed static blackhole from VPRN 101 |
| `configs/dcgw2-dc1.partial.cfg` | Same changes as dcgw1-dc1 |

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          IP/MPLS WAN                            │
│                                                                 │
│    ┌──────────────────────────────────────────────────────┐     │
│    │                      p1 (RR)                         │     │
│    │               Route Reflector AS 1257                │     │
│    └──────┬─────────┬─────────┬─────────┬────────┬────────┘     │
│           │         │         │         │        │              │
│        dcgw1     dcgw2     dcgw1     dcgw2      p2              │
│         dc1       dc1       dc2       dc2      (PE)             │
│                                              0.0.0.0/0          │
└─────────────────────────────────────────────────────────────────┘
         │         │                   │         │
    ┌────┴─────────┴──────┐     ┌──────┴─────────┴────┐
    │   DC1 Fabric        │     │   DC2 Fabric        │
    │   SR-Linux spine    │     │   SR-Linux spine    │
    │   SR-Linux leaf     │     │   SR-Linux leaf     │
    │   client1, client2  │     │   client3, client4  │
    └─────────────────────┘     └─────────────────────┘
```

`p2` is a minimal SR OS PE node connected to `p1` via IS-IS and LDP. It originates `0.0.0.0/0` as an aggregate route in VPRN 101 and exports it into VPNv4 with route target `target:65100:101`. `p1` reflects this to all datacenter gateways. Each gateway installs it as a `BGP VPN` route and re-advertises it as an EVPN type-5 route into the local fabric.

## Key design decisions

**p2 as dedicated PE** — SR OS does not allow a node configured as a BGP route reflector (`cluster { cluster-id ... }`) to simultaneously originate VPNv4 routes from a local VPRN. Separating the PE and RR roles onto distinct nodes (`p2` and `p1` respectively) resolves this platform constraint.

**Aggregate route on p2** — SR OS `bgp-ipvpn mpls` does not redistribute static blackhole routes into VPNv4 because it cannot bind an MPLS label to a null next-hop. An aggregate route with a blackhole forwarding action is redistributed correctly because SR OS treats it as a locally originated BGP route rather than a forwarding entry.

**VRF import policy on dcgws** — Without a `vrf-import` policy, datacenter gateways receive both EVPN-IFF routes from the local fabric and BGP VPN routes from `p1` for the same DC1 prefixes, causing asymmetric routing tables. The import policy restricts BGP VPN installation to `0.0.0.0/0` only, ensuring DC1 prefixes are always learned via EVPN-IFF from the local fabric.

**`default-preference ibgp 161` on dcgws** — By default, all iBGP-learned routes including both BGP VPN (from the `ipbb` group toward `p1`) and EVPN-IFF (from the `ibgp-evpn` group toward the spines) share the same iBGP preference. Setting `default-preference ibgp 161` lowers the preference of all iBGP-learned routes uniformly to 161, which is lower than the EVPN-IFF default of 169. However, since the `vrf-import` policy already filters out all BGP VPN routes except `0.0.0.0/0`, the only BGP VPN route that remains in the VPRN routing table is the default route from `p2`. For `0.0.0.0/0`, if `dcgw1` re-advertises it into the EVPN fabric as an EVPN-IFF route (preference 169), `dcgw2` would prefer it over the BGP VPN route (preference 170 by default). Setting `ibgp 161` ensures the BGP VPN-learned default route is always preferred over any EVPN-IFF re-advertisement, giving both gateways a consistent and independent path to `p2`.

## Prerequisites

- Docker and Containerlab installed
- Nokia SR OS license file at `/opt/nokia/sros/license-sros23.txt`
- Nokia SR OS image: `vrnetlab/vr-sros:24.10.R8`
- Nokia SR Linux image: `ghcr.io/nokia/srlinux:24.10`

## Deployment

Clone this repository and copy the unchanged config files from the original Nokia basic DCI lab into the `configs/` directory:

```bash
git clone https://github.com/<your-username>/nokia-dci-lab-extended
cd nokia-dci-lab-extended

# Copy unchanged configs from original repo
git clone https://github.com/srl-labs/nokia-basic-dci-lab original
cp original/configs/* configs/
# The files in this repo will overwrite the changed ones
cp -f p1.partial.cfg configs/
cp -f p2.partial.cfg configs/
cp -f dcgw1-dc1.partial.cfg configs/
cp -f dcgw2-dc1.partial.cfg configs/

# Deploy
sudo containerlab deploy -t dci.clab.yml
```

## Verification

After deployment, verify the corrected routing table on dcgw1-dc1:

```bash
docker exec -it clab-dci-dcgw1-dc1 bash
telnet localhost 5000
# login: admin / password: admin

show router 101 route-table
```

Expected output shows `0.0.0.0/0` via `BGP VPN` (next-hop `10.0.0.50`, tunneled via p1), DC1 prefixes via `EVPN-IFF`, and DC2 prefixes as backup routes `[B]`.

Verify cross-datacenter reachability:

```bash
docker exec -it clab-dci-client1-dc1 ping -c 5 20.0.0.4
```

## WAN link failure test

To reproduce the WAN link failure experiment, start a high-frequency ping from client1 to client4 and disable the WAN port on dcgw1-dc1:

```bash
# Terminal 1: start ping
docker exec -it clab-dci-client1-dc1 ping -i 0.01 20.0.0.4

# Terminal 2: disable WAN port on dcgw1-dc1
docker exec -it clab-dci-dcgw1-dc1 bash
telnet localhost 5000
# login: admin / password: admin
configure exclusive
    port 1/1/c1 admin-state disable
commit
```

With the corrected design, packet loss should be at most 1–2 packets (10–20 ms), compared to ~35% persistent loss with the original static blackhole configuration.

To restore:

```bash
configure exclusive
    port 1/1/c1 admin-state enable
commit
```

## Addressing scheme

| Node | Interface | Address |
|------|-----------|---------|
| p1 | system | 10.0.0.49/32 |
| p1 | dcgw1_dc1 | 10.31.49.49/24 |
| p1 | dcgw2_dc1 | 10.32.49.49/24 |
| p1 | dcgw1_dc2 | 10.41.49.49/24 |
| p1 | dcgw2_dc2 | 10.42.49.49/24 |
| p1 | p2 | 10.33.49.49/24 |
| p2 | system | 10.0.0.50/32 |
| p2 | p1 | 10.33.49.50/24 |
| p2 | lo0 (VPRN 101) | 10.255.101.1/32 |
| dcgw1-dc1 | system | 10.0.0.31/32 |
| dcgw1-dc1 | p1 | 10.31.49.31/24 |
| dcgw2-dc1 | system | 10.0.0.32/32 |
| dcgw2-dc1 | p1 | 10.32.49.32/24 |

## Reference

This work was produced as part of the thesis:

> Hilgert, L. (2026). *Decentralised datacenter switching for a resilient network fabric in mission-critical environments*. Master's thesis, University of Amsterdam / Royal Military Academy of Belgium.

Original topology: [srl-labs/nokia-basic-dci-lab](https://github.com/srl-labs/nokia-basic-dci-lab)

## License

Configuration files in this repository are provided for research and educational purposes. The Nokia SR OS and SR Linux images are subject to Nokia's licensing terms.
