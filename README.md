# BGP Labs on Raspberry Pi 5

Hands-on BGP networking labs running on a Raspberry Pi 5 (Ubuntu Server 24.04 ARM64) using FRRouting and Containerlab.

## Stack

- Raspberry Pi 5 Model B, 16GB RAM
- Ubuntu Server 24.04 LTS (ARM64)
- Docker + Containerlab 0.74
- FRRouting 10.1.1

## Labs in this repo

### `2-router-lab/`
A minimal eBGP lab between two ASes. Demonstrates BGP session establishment, FRR's default-deny policy, and the relationship between the `network` statement and the local routing table. See its README for details.

### `4-router-multi-as-lab/`
A four-router topology across two ASes with iBGP, redundant eBGP links, prefix-list filtering, and route-map policy. Shows path diversity, best-path selection, and filter symmetry across redundant peers.

## Built incrementally

Each lab here was built hands-on, with real debugging along the way (subnet math errors, FRR's default-deny defaults, filter symmetry across redundant links). The commit history reflects the journey, not a polished tutorial.
