---
title: "Verify everything"
date: 2026-01-15
draft: false
tldr: "Three layers of verification for a cryptography library: Lean proofs, property-based testing, and differential testing."
---

Most zero-knowledge bugs don't announce themselves.

Your prover generates a proof. The verifier accepts it. Your tests pass. Everything looks correct. And it might be. Right up until someone discovers a soundness hole that lets them forge a proof for a false statement. You never saw it coming because none of your tests could have caught it. The proof was structurally broken in a way that functional testing can't reach.

This is the fundamental problem with testing cryptographic systems: the thing you're most afraid of, a soundness failure, is exactly the thing that's hardest to detect. A wrong answer looks identical to a right one. Both pass the verifier. Both look fine in your CI pipeline. The difference only shows up when an adversary exploits it.

## Why multi-backend makes this worse

When I started building [gunmetal](/posts/heavymetal), I knew it wouldn't stay Metal-only for long. CUDA comes next, then Vulkan, then whatever else makes sense. But here's the thing: every new backend is a fresh surface for subtle bugs. Not the obvious kind that crash your program. The quiet kind that only show up on specific hardware, with specific inputs, in specific edge cases that nobody would think to test by hand.

A GPU kernel might be 10x faster than its CPU equivalent but introduce a one-in-a-million case where the result differs by a single bit. In most software, a single wrong bit is a rounding error. In cryptography, a single wrong bit can break soundness entirely. Your proof is now forgeable, and you have no idea.

So I built a verification framework for the prover itself. Three layers, each catching a different class of problem.

## Formal proofs for the math itself

The foundation is Lean proofs for core arithmetic. These verify that field operations actually form a field, that curve addition is associative, that pairing equations hold. If Lean accepts the proof, the math is correct. Not probably correct. Not correct for all inputs we tested. *Definitively* correct, in the mathematical sense. This was a monumental undertaking tbh, to have a complete Lean for Groth16.

This matters because everything else in the prover builds on these operations. If your field multiplication has a subtle bug, every layer above it (MSM, FFT, the entire Groth16 protocol) inherits that bug. Getting the foundation right isn't optional.

## Property-based testing at scale

Above the formal proofs, there are 193 proptest properties covering MSM, NTT/FFT, QAP/R1CS, and Groth16 end-to-end. The idea is simple but powerful: instead of checking five carefully chosen inputs, you verify mathematical properties across thousands of randomly generated ones.

For example, rather than testing "does this specific multiplication produce this specific result," you test "is multiplication commutative" and "is multiplication associative" and "does multiplying by one give back the original value", across thousands of random field elements. This is how you catch the edge cases that deterministic tests miss: the off-by-one in a field reduction, the carry propagation error that only triggers on a specific bit pattern.

## Differential testing across hardware

The third layer runs the same computation on GPU and CPU and compares the results bit-for-bit. Any divergence is a bug, full stop.

This layer exists because GPU arithmetic can be subtly wrong in ways that are nearly impossible to catch otherwise. The CPU implementation serves as a reference. If the GPU disagrees with it, something is broken, even if both results look plausible on their own. If my GPU implementation had been audited or even formally verified externally, I could have trusted the implementation. But since I'll be working on more backends, its best to have a proven truth (arkworks).

## Scanning for known attack patterns

The framework also scans for broken primitives, insufficient rounds, and FreeLunch attack patterns (ePrint 2024/347). If you're not familiar with FreeLunch: certain prover optimizations create soundness holes that let an attacker forge proofs for false statements. The proof passes every functional test you could write. It completely breaks your security model. And it comes from an *optimization*, something that was supposed to make things faster, not less secure.

This is exactly the class of bug that you cannot find by testing outputs. You have to verify the structure of the computation itself.

## The point

Implement a new backend. Run the verification suite. Pass everything. Now you have a cryptographically sound implementation. Not by hope, not by spot-checking a handful of test vectors, but by construction.

At least that was the point when I did all that.
