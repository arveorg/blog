---
title: "verify everything"
date: 2026-01-15
draft: false
---

most zk bugs are silent. your prover generates something that looks valid, verifier accepts it, everything feels fine until someone exploits a soundness hole you never knew existed.

when i started building gunmetal, i kinda knew i wouldn't stop at just one backend. Metal today, CUDA tomorrow, Vulkan after that. but every new backend is a fresh opportunity to introduce subtle bugs that only manifest on specific hardware, specific inputs, specific cosmic ray alignments. adding automated test cases for this is true shit and easy to mess up.

so i built gunmetal-verify, to formally verify the prover. three tiers of paranoia:

lean proofs for core arithmetic - field operations actually form a field, curve addition is associative, pairing equations hold. if lean accepts it, it's *definitively* correct. 193 proptest properties covering MSM, NTT/FFT, QAP/R1CS, Groth16 - instead of checking 5 handpicked inputs we're verifying mathematical properties across thousands of random ones. catches edge cases deterministic tests miss. then differential testing: same computation on GPU vs CPU, compare bit-for-bit. any divergence is a bug.

the framework also scans for broken primitives, insufficient rounds, and FreeLunch attack patterns (ePrint 2024/347). if you're not familiar: certain prover optimizations create soundness holes letting you forge proofs for false statements. passes every functional test, completely breaks your security model.

backend-specific bugs are a fascinating failure mode. your cuda kernel might be 10x faster but introduce a 1-in-million case where the result is wrong by a single bit. that single bit can break soundness entirely.

implement, run suite, pass = cryptographically correct by construction. that's the whole point.
