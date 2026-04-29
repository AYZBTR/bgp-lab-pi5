# 2-Router eBGP Lab — FRR on Containerlab (Raspberry Pi 5)

A minimal eBGP lab running on a Raspberry Pi 5 (Ubuntu Server 24.04 LTS, ARM64). 
Two FRRouting containers in separate ASes establish a BGP session over a virtual 
link and exchange prefixes.

## Topology

```
[r1 lo: 192.168.1.1/24]            [r2 lo: 192.168.2.1/24]
       |                                    |
       | eth1: 10.0.0.1/30 ----eBGP---- eth1: 10.0.0.2/30 |
       |                                    |
     AS 65001                            AS 65002
```


## Stack

- Raspberry Pi 5, 16GB RAM
- Ubuntu Server 24.04 LTS (ARM64)
- Docker + Containerlab 0.74.3
- FRRouting 10.1.1

## Lessons learned

1. **FRR's strict-policy default**: Modern FRR refuses to exchange routes with 
   external peers unless explicit route-maps are defined. For this lab I used 
   `no bgp ebgp-requires-policy` to disable the requirement, but in production 
   I would always configure explicit inbound and outbound route-maps as a safety 
   measure against route leaks.

2. **`network` statement requires an existing route**: BGP's `network` directive 
   only advertises a prefix if it already exists in the routing table. I added 
   loopback interfaces with the advertised prefixes so FRR could resolve them 
   as valid routes. Without this, FRR marks them `inaccessible` and silently 
   drops them.

3. **Soft reconfiguration inbound**: Without `soft-reconfiguration inbound`, 
   `show ip bgp neighbor X received-routes` cannot show pre-policy received 
   routes. Useful for diagnostics.

## Verification

After deploy, the BGP session reaches Established state within ~30 seconds 
and prefixes are exchanged in both directions. End-to-end connectivity 
between loopbacks confirms the BGP-learned path is operational.

## Next steps

- Expand to a 4-router topology across 2 ASes (iBGP within, eBGP between)
- Add route-maps, prefix-lists, and AS-path prepending
- Apply for DN42 membership and run as a live BGP node
