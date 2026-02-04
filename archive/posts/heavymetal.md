---
title: "Heavy metal"
date: 2026-01-13
draft: false
tldr: "Built a Groth16 prover that runs on Apple Metal because I needed one and the alternatives aren't free."
---

I built a Groth16 prover that runs on Apple Metal, called it gunmetal for now.

If you work in zero-knowledge cryptography, your first reaction is probably *why?* And honestly, that's fair. ZK proofs run on servers. The entire ecosystem is built around it. [gnark](https://github.com/Consensys/gnark), [arkworks](https://github.com/arkworks-rs), [rapidsnark](https://github.com/iden3/rapidsnark), [snarkjs](https://github.com/iden3/snarkjs), [icicle](https://github.com/ingonyama-zk/icicle). These are the standard tools, and they all target server-class hardware. Nobody proves on a laptop.

But Apple Silicon has a property that server GPUs don't: unified memory. The CPU and GPU share the same physical memory, with no copying between them. In a prover, where you're constantly moving large amounts of data between CPU-side computation and GPU-side computation, eliminating that copy overhead felt like it could matter more than people assumed. I wanted to find out.

It does.

## The result

120ms for 134,000 constraints on an M3 Ultra. That's 2.9x faster than the same workload running on CPU alone. Not bad.

Here's how it works, and more importantly, *why* it's fast.

## Custom arithmetic, all the way down

The first decision was to build field arithmetic from scratch instead of relying on existing libraries. This sounds like unnecessary masochism, but it's where a huge chunk of the performance comes from. The key move is keeping everything on the stack. Fixed-size arrays instead of heap-allocated big integers. When you're doing millions of field multiplications, the difference between a stack allocation and a heap allocation adds up fast.

GLV endomorphism with zero-allocation Barrett reduction gets roughly 20x faster scalar decomposition than the standard approach. Montgomery multiplication is fully unrolled in Metal shaders, so the GPU spends its time doing math instead of managing memory.

## Never let the hardware sit idle

This is where the real gains live. A Groth16 proof involves two expensive categories of work: QAP computation (the polynomial math) and MSM execution (the elliptic curve math). The natural instinct is to do one, then the other. But if you pipeline them, running QAP computation on one piece of hardware while MSM runs on another, you can overlap them almost entirely.

In practice, the GPU handles G1 MSMs while the CPU handles G2 MSMs in parallel. Two of the most expensive operations in the entire proving pipeline, running concurrently. Zero idle time.

## The FFT that actually fits the GPU

Number-theoretic transforms are another major bottleneck, and the algorithm choice here matters enormously. I went with a Stockham FFT, which avoids the bit-reversal permutations that traditional FFT approaches require. That turns out to be a big deal on a GPU, because bit-reversal creates scattered memory access patterns that destroy throughput. Stockham keeps memory access coalesced, and the result is about 10x faster than the CPU baseline.

## Unified memory is the whole game

This is the part most people miss when comparing Apple Silicon to discrete GPUs. On a traditional setup with an NVIDIA card, you have to copy data from CPU memory to GPU memory across the PCIe bus. You need staging buffers. You pay transfer costs every time the GPU needs new data.

On Apple Silicon, there's none of that. The GPU reads directly from the same physical memory the CPU wrote to. No copies, no bus, no staging. The data is just *there*. Combine that with interleaved point layout for cache locality, and memory access patterns become essentially free performance. This architectural advantage is what makes the whole approach viable.

## Tuning for Metal specifically

The MSM implementation uses the Pippenger algorithm with 13-bit windows and local threadgroup atomics. One thing I found during profiling: 128 threads per threadgroup consistently outperforms the conventional 256-thread baseline on Metal hardware. This is the kind of thing you can only discover by measuring on actual hardware. The textbook answer isn't always right for a specific execution model.

## Prior art

The [zkmopro](https://github.com/zkmopro) team and EluAegis did some cool work on Metal MSM. What I needed, though, was a complete proving pipeline, not just MSM, and one that's architecturally ready to support multiple GPU backends over time.

Next up: CUDA and ROCm backends. Maybe Mali. Someday. The architecture is designed for it. First I have some other battles to overcome.
