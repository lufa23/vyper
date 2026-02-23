.. _compiler-verification:

Compiler Verification
#####################

A verified compiler comes with a mathematical proof that it preserves semantics: the compiled bytecode behaves identically to what the source code means according to the formal semantics. This closes the :doc:`verification gap <verification-gap>` and makes source-level proofs valid for deployed bytecode. Compiler verification has been achieved for other languages and is a goal for Vyper.


What Compiler Verification Means
==================================

A compiler is a function from source programs to target programs (in Vyper's case, from Vyper source to EVM bytecode).

A *verified* compiler comes with a proof: for all valid source programs P, ``compile(P)`` on the EVM preserves the observable behavior of P according to the formal semantics. "Observable behavior" is defined precisely: return values, storage writes, events emitted, ether transferred. Internal details like stack layout and memory allocation can differ, as long as externally visible behavior is preserved.

This is a **universal** statement. It holds for all programs the compiler accepts, not just programs that were tested. If the compiler is verified, any property proved about source code automatically holds for the deployed bytecode. The verification gap is closed by construction.


Precedents: CompCert and CakeML
=================================

**CompCert** (2006–present, INRIA) is a formally verified C compiler, proved correct in the `Coq <https://coq.inria.fr/>`_ proof assistant. It covers most of ISO C99 and targets multiple architectures (PowerPC, ARM, x86). CompCert is the first verified compiler for a major production language.

The Csmith study (Yang et al., 2011) tested production C compilers by compiling randomly generated programs and checking for miscompilations. Every tested compiler (including GCC and LLVM) exhibited bugs. In CompCert, the verified middle-end (the compilation passes covered by the correctness proof) produced zero miscompilation bugs. Bugs were found in CompCert's unverified front-end and back-end (the parts not covered by the proof). This result demonstrates both the power of verification where it is applied and the cost of leaving any part of the pipeline unverified.

**CakeML** (2014–present) is a formally verified compiler for a functional programming language (ML), proved correct in `HOL4 <https://hol-theorem-prover.org>`_, the same proof assistant we use in the `Verifereum <https://verifereum.org>`_ project. CakeML covers the full compilation pipeline from source to machine code and includes a verified garbage collector and runtime. The infrastructure, techniques, and libraries developed for CakeML include proof strategies for compiler passes, verified data structures, and automation for semantic preservation. The methodology is applicable to verifying a Vyper compiler in HOL4.

The key lesson from both projects: compiler verification requires significant effort, but the result is permanent. Once the compiler is verified, every program it compiles benefits from the proof.


What End-to-End Means
======================

End-to-end verification means the proof covers the **entire** compilation pipeline (every pass, every transformation, every optimization) from source to target. No intermediate step is left unverified. The CompCert Csmith result illustrates why this matters: verification of the middle-end eliminated bugs there, but bugs survived in the unverified front-end and back-end.

Partial verification (verifying individual compiler passes in isolation) leaves gaps between passes. An optimization that is individually correct can interact with a subsequent pass to produce incorrect code. The standard approach, pioneered by CompCert, is compositional: prove each pass correct separately, then compose the proofs. If pass A preserves semantics and pass B preserves semantics, then A followed by B preserves semantics. The chain extends from the first pass to the last.

For Vyper, the compilation pipeline includes parsing, type-checking, IR generation (Venom), optimization, and bytecode generation. End-to-end verification covers all of these stages, ensuring that no pass introduces a discrepancy between source-level semantics and deployed bytecode.

Even before end-to-end verification is complete, partial verification has value. Verifying individual compilation passes eliminates bugs in those passes. The Csmith result illustrates this: CompCert's verified middle-end was bug-free, even though the unverified parts were not. Each verified pass narrows the space where compiler bugs can hide. The goal is end-to-end, but every step toward it improves the baseline.


Why Vyper Is Tractable
========================

Compiler verification difficulty scales with the complexity of the source language and the compilation pipeline.

**No unbounded recursion.** Recursive function calls create complex control flow that the compiler must translate correctly. Vyper prohibits recursion, eliminating this class of compilation complexity.

**Bounded loops.** Every loop in Vyper has a statically known bound. The compiler does not need to handle arbitrary loop structures, and the verification does not need to reason about unbounded iteration in compiled code.

**No inheritance.** Inheritance introduces method dispatch, linearization, and storage layout interactions that complicate compilation and its verification. Vyper's module system avoids these.

**No inline assembly.** Inline assembly bypasses the compiler's normal translation pipeline. Verifying a compiler that supports it requires reasoning about arbitrary low-level code embedded within high-level constructs. Vyper does not allow it.

**Simpler IR.** Vyper's Venom IR is designed for clarity, reducing the complexity of the IR-to-bytecode verification.

Each restriction reduces the number of cases the compiler handles and the number of feature interactions the proof must cover. No compiler verification exists for any general-purpose smart contract language with unbounded recursion, multiple inheritance, and inline assembly. Vyper's design makes the problem tractable.


What This Enables
==================

With a verified compiler, the full verification pipeline connects without gaps:

1. **Source-level properties** are proved about the Vyper contract.
2. **The formal semantics** guarantees these properties mean what the proof claims.
3. **The verified compiler** guarantees the bytecode matches the source semantics.
4. **The EVM formalization** guarantees the bytecode executes as specified.

No trust gaps. No unverified translation steps. The proof chain is complete from specification to on-chain execution.

A developer writes a Vyper contract, has critical properties formally verified at the source level, compiles, and deploys. The verification guarantees that the deployed bytecode satisfies those properties. Compiler verification is the component that extends source-level guarantees to deployed code.


Current Status
================

The `Verifereum <https://verifereum.org>`_ project has compiler verification as a stated goal. The prerequisites are in place:

- The formal semantics of Vyper (`vyper-hol <https://github.com/verifereum/vyper-hol>`_) covers the core of the language and passes the ``functional/codegen`` section of the official test suite.
- The EVM formalization is built in the main `Verifereum repository <https://github.com/verifereum/verifereum>`_ and is already used by the Vyper formal semantics.

Compiler verification is a long-term effort. CompCert and CakeML each required years of sustained work. They demonstrate that the goal is achievable, and Vyper's design makes it more tractable than it would be for languages with more complex semantics.

For current progress, see the `vyper-hol issue tracker <https://github.com/verifereum/vyper-hol/issues>`_.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`verification-gap`, why compiler verification matters and what happens without it

   :doc:`trusted-computing-base`, why the proof assistant's kernel is the right foundation for compiler proofs
