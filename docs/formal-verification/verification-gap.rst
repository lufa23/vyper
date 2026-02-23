.. _verification-gap:

The Verification Gap
####################

When we verify a smart contract at the source level but deploy compiled bytecode, there is a gap between what was verified and what actually runs. Any bug in the compiler can invalidate the verification. This is the verification gap, and it is the default state of all source-level verification tools today.


What the Verification Gap Is
==============================

We write a contract in Vyper. We verify properties about the source code, with an SMT solver, a theorem prover, or both. We compile it and deploy the bytecode to Ethereum.

What runs on-chain is the **bytecode**, not the source code. The verification was about the source code.

The compiler is the bridge. If it translates correctly, the verified properties carry over to the bytecode. If it has a bug (even a subtle one in an edge case), the bytecode may behave differently from the source, and the verification is meaningless for the deployed contract.

The verification gap is the space between "verified source" and "deployed bytecode" where unverified compilation happens. Every source-level verification result implicitly trusts the compiler. That trust is usually unexamined.

The gap applies equally to audits. An auditor reviews source code, not deployed bytecode. If a compiler bug changes the semantics of the compiled output, the auditor's conclusions about the source are invalidated. The gap is not specific to formal verification; it is inherent in any source-level analysis.


Why This Is Not a Theoretical Concern
=======================================

Compilers have bugs. All compilers. This is an inherent reality of complex software.

Smart contract compilers have had bugs that changed the semantics of compiled code: bugs that could have invalidated source-level verification of the affected contracts. The Vyper compiler has had bugs. GCC and LLVM, with decades of development and enormous user bases, have bugs. Compilers are among the most carefully engineered software in existence, and they still contain defects.

The more effort we put into source-level verification, the more the compiler becomes the weakest link. If we have proved source code correct relative to the formal semantics, the only remaining source of semantic bugs is the compiler. The proof effort produces a result contingent on an unverified translation step.


How the Gap Manifests
======================

Compiler bugs do not announce themselves. The source code looks correct. The verification says it is correct. The bytecode does something different. There is no mechanism to detect the discrepancy short of verifying the compiler itself or independently analyzing the bytecode.

Several areas of compilation are particularly prone to subtle bugs:

**Optimization passes.** An optimization that is correct for most programs but wrong for a specific edge case can silently change behavior. The optimization may have been tested extensively, but testing cannot cover every possible input program.

**Code generation for complex features.** Dynamic arrays, ABI encoding and decoding, storage layout computation, and memory management involve intricate logic where off-by-one errors and boundary conditions are common sources of defects.

**Interactions between compiler passes.** Each pass may be individually correct, but the composition of passes can produce incorrect code. An optimization that is sound in isolation can interact with a subsequent pass in a way neither was designed to handle.

The gap is invisible to the developer. The contract compiles, the tests pass, and the source-level verification succeeds. The only way to discover a compiler-introduced bug is through independent bytecode analysis, a compiler bug report, or, in the worst case, an exploit.


The Current State
==================

All source-level verification tools today (SMT-based, type-system-based, and most theorem-prover-based) operate above the verification gap. They verify source-level properties and trust the compiler.

Some tools work at the bytecode level instead, analyzing deployed EVM bytecode directly. This eliminates the compiler trust assumption but introduces different problems: bytecode-level reasoning is harder, less readable, and disconnected from the source-level abstractions that make contracts understandable.

Neither approach alone is complete. Source-level verification is expressive but trusts the compiler. Bytecode-level verification is compiler-independent but loses source-level reasoning.


Closing the Gap
================

The only way to close the verification gap without losing source-level reasoning is to **verify the compiler itself**: prove that for any valid source program, the compiled bytecode has the same observable behavior as the source according to the formal semantics.

This has been done for other languages. CompCert (verified C compiler, proved correct in Coq) and CakeML (verified ML compiler, proved correct in HOL4) demonstrate that compiler verification is achievable for realistic languages. The Csmith study (Yang et al., 2011) showed that CompCert's verified middle-end produced zero miscompilation bugs, while bugs were found in every unverified compiler tested, and in CompCert's own unverified front-end and back-end. See :doc:`compiler-verification` for a detailed treatment.

With a verified compiler, the proof chain is complete: a property proved about source code, grounded in a formal semantics, is guaranteed to hold in the deployed bytecode.

Vyper's design makes compiler verification more tractable than it would be for a language with unbounded recursion, inheritance hierarchies, and inline assembly. Each restriction reduces the number of cases the compiler must handle and the complexity of the correctness proof. This is the goal of the `Verifereum <https://verifereum.org>`_ project's compiler verification work.

Before full compiler verification is complete, translation validation offers a partial mitigation. Translation validation checks, for a specific compilation run, that the output bytecode is consistent with the source semantics. It does not prove the compiler correct for all inputs. It can catch compiler bugs that affect a specific contract. We can automate translation validation and integrate it into the deployment pipeline as an additional check between source-level verification and deployment.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`compiler-verification`, what it means to verify a compiler and how it closes the gap

   :doc:`trusted-computing-base`, why the size of the trusted base matters for confidence in results
