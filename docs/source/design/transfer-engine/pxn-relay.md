---
orphan: true
---

# Legacy Transfer Engine: KV cache relay across isolated RDMA rails

## Goal

In the PD deployment, P writes KV cache to D. Each GPU has a nearby RNIC,
but RNICs on different rails cannot communicate through the leaf-only
fabric. Selecting the RNIC near either endpoint can therefore force the
other endpoint's GPUDirect RDMA data path across NUMA domains:

```
Before A: P GPU3 -- PIX/PCIe -- P mlx5_6/7 -- RDMA -- D mlx5_6/7
                                                    -- SYS -- D GPU5
Before B: P GPU3 -- SYS -- P mlx5_10/11 -- RDMA -- D mlx5_10/11
```

The primary goal is to avoid the `SYS` GPU-to-RNIC data path on **both** hosts.
The network transfer must remain within one reachable rail. A GPU-to-GPU
relay over direct NVLink is allowed even when the two GPUs have different
NUMA affinities; the previous draft's rule requiring every participating GPU
to share one NUMA node was too strict.

For this placement, the desired WRITE route is:

```
P GPU3 -- NV18/NVLink --> P GPU5 staging buffer
       -- PIX/PCIe ------> P mlx5_10/11
       -- InfiniBand/RDMA -> D mlx5_10/11
       -- PIX/PCIe ------> D GPU5 KV cache
```

The corresponding READ into P GPU3 reverses the stages: RDMA from D GPU5
through the same rail into a P GPU5 staging buffer, followed by a local
NVLink copy to P GPU3.

## Route contract

The feature is opt-in. Route configuration or discovery must identify the
reachable RNIC pairs, the GPU-to-RNIC PCIe path on each host, and the local
GPU-to-GPU relay path. Plan the entire route before sending any data. A route
is valid only when:

1. The chosen P and D RNICs are on the same reachable rail.
2. The GPU buffer used by RDMA on each host has a GPU-to-RNIC path that does
   not cross NUMA (`SYS` is forbidden). Prefer a nearby `PIX` path when one
   is available. Validate the actual topology path rather than inferring it
   from matching device names alone.
3. Each relay copy has a verified GPU P2P path. For the initial deployment,
   P GPU3 to P GPU5 must use NVLink; this hop is intentionally allowed even
   if their NUMA affinities differ.
4. The staging allocation is registered with the RNIC that performs RDMA,
   and it stays alive until both the copy and network stage finish.
5. A retry may use only a different **complete** route satisfying all three
   path checks. It cannot switch one RNIC independently and recreate a `SYS`
   GPU-to-RNIC hop. If no complete route exists, fail before sending.

Unknown GPU-to-RNIC or GPU-to-GPU topology is not proof of a valid route.
The operator can provide an explicit verified map when automatic discovery
cannot classify it.

## Execution and ownership

Each public task owns a staged state machine:

```
WRITE: source GPU -> relay GPU -> RDMA -> destination GPU -> completed
READ:  source GPU -> RDMA -> relay GPU -> destination GPU -> completed
```

In the stated PD WRITE, the source copy and relay buffer are on P. D receives
RDMA directly into its GPU5 KV cache, so D needs no staging buffer or extra
copy. The public task reports completion only after every required stage has
completed. Stage failures drain in-flight work before freeing intermediate
buffers. A bounded staging pool and chunked pipeline limit relay memory use.

For Legacy Transfer Engine, `MultiTransport` must associate a public task
with its relay stages and final status. The local GPU copy can use the
existing CUDA/NVLink machinery; `RdmaTransport` supplies the network stage
and must enforce the selected RNIC pair on initial submission and retries.
If a future route needs a copy on D, D must coordinate that copy before it
reports the KV cache ready to decode.

## Deployment inputs and validation

The transfer process sees GPU0 through GPU7, and GPU0/GPU1 CUDA P2P has been
confirmed. The GPU3/GPU5 NVLink path, GPU/RNIC topology, and allowed RNIC
pairs still need to be verified on the deployment hosts. The first version
can use an in-process staging pool if GPU3 and GPU5 are visible and P2P works
in the same transfer process.

Test the two current bad routes and the proposed relay route. Confirm the
planner rejects any `SYS` GPU-to-RNIC edge, an unreachable rail pair, an
unavailable NVLink/P2P hop, and an invalid retry. Cover READ and WRITE stage
order, buffer lifetime, chunk backpressure, cancellation, and completion
semantics. Measure KV cache throughput and time to first token with the
same load before and after relay; the extra NVLink copy must cost less than
the avoided cross-NUMA RDMA path for the deployment to benefit.
