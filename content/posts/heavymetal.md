---
title: "heavy metal"
date: 2026-01-13
draft: false
---

built a groth16 prover for mac metal.

this perhaps makes zero sense. proofs run on servers. [gnark](https://github.com/Consensys/gnark), [arkworks](https://github.com/arkworks-rs), [rapidsnark](https://github.com/iden3/rapidsnark), [snarkjs](https://github.com/iden3/snarkjs), [icicle](https://github.com/ingonyama-zk/icicle) - that's the stack. nobody's proving on laptops. but i had the tickle & was curious how the M-series Ultra would hold up with its unified memory.

120ms for 134k constraints on M3 Ultra. 2.9x faster than CPU.

here's why.

custom field arithmetic instead of relying on arkworks. GLV endomorphism with zero-allocation Barrett reduction gives ~20x faster scalar decomposition - arkworks uses heap-allocating BigInt, we use stack arrays. CIOS Montgomery multiplication fully unrolled in Metal shaders.

pipelining is where it gets interesting. QAP computation (the polynomial math) overlaps with MSM execution (the elliptic curve math). while the GPU grinds through G1 MSMs, the CPU handles G2 MSMs in parallel. two expensive operations, zero waiting.

stockham FFT for NTT computation - 10x faster than CPU. avoids bit-reversal permutations and enables coalesced memory access.

apple silicon's unified memory architecture means zero-copy between CPU & GPU. no PCIe bottleneck. no staging buffers. the GPU just... sees the data. interleaved point layout for cache locality means memory access patterns are basically free performance. this is a massive advantage most people sleep on.

pippenger algorithm with 13-bit windows & local threadgroup atomics. 128 threads per threadgroup hit the sweet spot after profiling - beats the 256 baseline. classic MSM acceleration but tuned specifically for metal's execution model.

PRIOR ART

the [zkmopro](https://github.com/zkmopro) team and EluAegis were onto something with their metal MSM work. but i needed something holistic to support future platforms, and icicle's GPU backends are closed source commercial stuff anyway.

now i'm tickling to add cuda, rocm, maybe mali. who knows.
