# Maximum Packets Per Second (PPS) Through a Network Interface

## Theoretical Maximum PPS by Link Speed (64-byte frames)

| Link Speed | Max PPS (theoretical) | With overhead* |
|------------|----------------------|----------------|
| 1 Gbps | ~1,488,095 pps | ~1.49M pps |
| 10 Gbps | ~14,880,952 pps | ~14.88M pps |
| 25 Gbps | ~37,202,380 pps | ~37.2M pps |
| 40 Gbps | ~59,523,809 pps | ~59.5M pps |
| 100 Gbps | ~148,809,523 pps | ~148.8M pps |

*Ethernet overhead: 64-byte frame + 20 bytes (preamble + inter-frame gap) = 84 bytes per packet on the wire*

## Formula

```
Max PPS = Link Speed (bps) / ((Frame Size + 20) × 8)
```

Examples:

```
1 Gbps, 64-byte frames:
1,000,000,000 / ((64 + 20) × 8) = 1,488,095 pps

1 Gbps, 1500-byte frames (standard MTU):
1,000,000,000 / ((1500 + 20) × 8) = 81,274 pps
```

## Real-World Limits

In practice, PPS is lower due to system bottlenecks:

| Bottleneck | Typical impact |
|------------|---------------|
| CPU / interrupt handling | Biggest limiter on software path |
| NIC hardware queue depth | Drops if ring buffer fills |
| Kernel network stack | Context switches, memory copies |
| Driver efficiency | Some drivers handle RSS/multiqueue better |
| Offloading settings | TSO/GRO/GSO reduce CPU but aggregate packets |

### Realistic Numbers on Linux

| Method | PPS achievable |
|--------|---------------|
| Kernel stack (single core) | ~1–2M pps |
| Kernel stack (multi-queue, RSS) | ~5–10M pps |
| DPDK / XDP bypass | ~20–80M pps per core |
| Hardware (smart NIC) | Line rate |

## Check Your NIC's Actual Limits

```bash
# See link speed
ethtool eth0 | grep Speed

# Ring buffer sizes (how many packets can queue)
ethtool -g eth0

# Check for drops (rx_missed, rx_dropped = hitting limits)
ethtool -S eth0 | grep -i "drop\|miss\|error"

# See interrupt coalescing settings
ethtool -c eth0

# Check multi-queue / RSS configuration
ethtool -l eth0
```

## Key Takeaways

- At **1 Gbps with small packets (64 bytes)**: max ~1.49M pps
- At **1 Gbps with standard MTU (1500 bytes)**: max ~81K pps
- Smaller packets = more PPS needed to saturate bandwidth
- Real bottleneck is usually CPU, not the wire
- Use `ethtool -S` to check if you're dropping packets at the NIC level
