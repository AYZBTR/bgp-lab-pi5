# 4-Router Multi-AS BGP Lab

A four-router topology across two autonomous systems demonstrating iBGP, eBGP, redundancy, and routing policy. Built on a Raspberry Pi 5 using FRRouting and Containerlab.

## Topology

AS 65001                            AS 65002
          ┌──────────────────┐               ┌──────────────────┐
          │   ┌────┐         │   eBGP        │         ┌────┐   │
          │   │ r1 │─────────┼───────────────┼─────────│ r3 │   │
          │   └────┘         │               │         └────┘   │
          │      │           │               │            │     │
          │  iBGP│           │               │       iBGP │     │
          │      │           │               │            │     │
          │   ┌────┐         │   eBGP        │         ┌────┐   │
          │   │ r2 │─────────┼───────────────┼─────────│ r4 │   │
          │   └────┘         │               │         └────┘   │
          └──────────────────┘               └──────────────────┘



Each AS has two routers connected via iBGP. The two ASes are connected via two redundant eBGP links (r1↔r3 and r2↔r4) so that traffic has automatic failover if one external link fails.

## Addressing plan

| Router | AS    | Loopback (advertised) | iBGP link IP    | eBGP link IP                |
|--------|-------|----------------------|-----------------|------------------------------|
| r1     | 65001 | 192.168.1.0/24       | 10.1.0.1/30     | 10.0.13.1/30 → r3            |
| r2     | 65001 | 192.168.2.0/24       | 10.1.0.2/30     | 10.0.24.1/30 → r4            |
| r3     | 65002 | 192.168.3.0/24       | 10.2.0.1/30     | 10.0.13.2/30 → r1            |
| r4     | 65002 | 192.168.4.0/24       | 10.2.0.2/30     | 10.0.24.2/30 → r2            |

## Files

- `lab.clab.yml` — Containerlab topology
- `r1/`, `r2/`, `r3/`, `r4/` — per-router FRR config (`daemons` + `frr.conf`)

## Run it

```bash
cd 4-router-multi-as-lab
sudo containerlab deploy -t lab.clab.yml
```

Wait ~60 seconds for all four BGP sessions to establish.

## Verify

```bash
docker exec -it clab-bgp-multi-as-r1 vtysh -c "show ip bgp summary"
docker exec -it clab-bgp-multi-as-r1 vtysh -c "show ip bgp"
```

Each router should see all four `192.168.X.0/24` networks. Networks within the local AS appear as iBGP-learned (`*>i`); networks in the other AS appear via both eBGP (preferred, `*>`) and a backup iBGP path (alternate, `* i`) — demonstrating path diversity.

End-to-end test from r1:
```bash
docker exec -it clab-bgp-multi-as-r1 ping -c 3 192.168.4.1
```

## Tear down

```bash
sudo containerlab destroy -t lab.clab.yml
```

## Concepts demonstrated

- **iBGP vs eBGP** in the same topology
- **Path diversity** — every router has multiple paths to remote networks
- **BGP best-path selection** — eBGP-learned routes are preferred over iBGP-learned routes by default
- **Redundancy** — when one eBGP link fails, BGP automatically converges to the backup path
- **Inbound prefix-list filtering** on eBGP sessions, validated by injecting a rogue route
- **Filter symmetry** — applying the same filter on all redundant eBGP sessions to a peer AS

## Lessons learned

1. **`Active` doesn't mean "working" in BGP.** A neighbor in `Active` state is *trying* to establish a session, not succeeding. The real progression is `Idle → Connect → Active → OpenSent → OpenConfirm → Established`. Counterintuitive but classic.

2. **Subnet math matters.** A `/30` only has two usable host IPs. `10.0.13.0/30` allows only `.1` and `.2`; `.0` and `.3` are network and broadcast. I spent a debugging session figuring out why an eBGP session was stuck in `Active` because a link IP had been set to the broadcast address by mistake.

3. **Filter symmetry is essential.** A prefix-list on one eBGP link doesn't protect against unwanted routes if there's a redundant link without the same filter. Demonstrated by injecting `1.1.1.0/24` on r3 and watching it sneak in via r4 → r2 → r1 even though it was correctly filtered on the direct r1↔r3 link. Fixed by applying the same prefix-list on r2's eBGP session to r4.

4. **iBGP doesn't re-advertise routes to other iBGP peers.** A loop-prevention rule built into BGP. In larger topologies this is solved with route reflectors or full-mesh iBGP. In this 2-router-per-AS lab the issue doesn't surface because each router has only one iBGP peer.
