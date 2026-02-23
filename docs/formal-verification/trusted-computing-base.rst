.. _trusted-computing-base:

The Trusted Computing Base
##########################

Every verification system has a trusted computing base: the set of components that must be correct for the result to be valid. The size and complexity of this base determines how much confidence the result deserves. Interactive theorem provers have the smallest trusted computing base of any verification approach.


What the Trusted Computing Base Is
====================================

The trusted computing base (TCB) is everything that must be correct for a verification result to hold. If any component in the TCB has a bug, the result could be wrong: a property reported as "verified" might not actually hold.

This concept applies to all verification approaches. An audit's TCB includes the auditor's judgment, attention, and domain knowledge. A test suite's TCB includes the test harness, the compiler, and the runtime. A type checker's TCB includes the type-checking algorithm and its implementation.

For formal verification, the question is precise: what software must be correct for a proof to be valid?


The TCB of an SMT Solver
==========================

When an SMT solver reports "verified," the claim is only as reliable as the solver implementation. The entire solver is in the TCB.

Major SMT solvers are hundreds of thousands of lines of highly optimized C++ code. They perform search, simplification, theory-specific reasoning, and heuristic optimization. The theories cover integers, bit-vectors, arrays, and other data types. A bug in any of these components could cause the solver to report "verified" when the property does not hold.

SMT solvers have had soundness bugs, cases where the solver returned "unsatisfiable" (meaning "the property holds") for formulas that were in fact satisfiable. The bug trackers of major solvers document these. The bugs are discovered and fixed, but their existence means the entire solver is a real component of the TCB, not just a theoretical one.

Some solvers support proof certificates (such as LFSC or Alethe) that can be independently checked by a smaller verifier. This shifts part of the trust from the solver to the certificate checker, reducing the effective TCB. Certificate support is not universal, certificate formats are still evolving, and the certificate checkers themselves become part of the TCB.


The LCF Architecture
=====================

`HOL4 <https://hol-theorem-prover.org>`_, Isabelle, and related systems follow the **LCF architecture**, named after Robin Milner's Logic for Computable Functions system from the 1970s. This architecture is what gives interactive theorem provers a fundamentally smaller TCB.

The key idea: a small *kernel* (a few thousand lines of code) is the **only** component that can produce theorems. The kernel implements a small number of primitive inference rules (in HOL4, roughly eight rules of higher-order logic). A theorem is a value of an abstract type that can only be constructed by applying these rules. The type system of the implementation language (Standard ML) enforces this boundary at the language level.

Everything outside the kernel (tactics, automation, decision procedures, proof search) is **untrusted**. These components suggest proof steps, decompose goals, and orchestrate reasoning. But they cannot produce theorems directly. They can only build theorems by calling the kernel's primitive inference rules, and the kernel checks each step independently. If an automation tactic has a bug, the worst outcome is that it fails to find a proof or produces a proof attempt the kernel rejects. It **cannot** produce a false theorem.

An analogy: the kernel is a judge who only admits evidence following strict rules of admissibility. The lawyers (tactics, decision procedures, automation) can be creative, disorganized, or adversarial. The judge catches invalid arguments regardless. Correctness depends on the judge, not on the lawyers.


Comparing TCB Sizes
=====================

.. list-table::
   :widths: 25 25 50
   :header-rows: 1

   * - Approach
     - TCB size
     - What must be trusted
   * - SMT solver
     - Hundreds of thousands of lines
     - Entire solver implementation
   * - Interactive theorem prover
     - ~5,000 lines (kernel only)
     - Kernel + axioms of the logic
   * - Audit
     - N/A (human judgment)
     - Auditor's reasoning and attention

A 5,000-line kernel that has been publicly available and scrutinized for decades is a different trust proposition from a solver of hundreds of thousands of lines under active development. The kernel is small enough to read in its entirety, simple enough to understand completely, and stable enough that changes are rare and carefully reviewed. This is one of the reasons we build our verification pipeline in HOL4: every proof we produce is checked by a kernel of this size.


Can the Kernel Have Bugs?
==========================

Yes. No software is provably bug-free. Proving a kernel correct requires another verified system, which has its own TCB. But the kernel is small enough, simple enough, and stable enough that the probability of a soundness bug is extremely low. HOL4's kernel has been stable for decades. Soundness bugs have occurred historically; they are exceedingly rare and quickly fixed when discovered.

Multiple independent implementations of the same logic provide cross-validation. HOL4, `HOL Light <https://www.cl.cam.ac.uk/~jrh13/hol-light/>`_, and Isabelle/HOL all implement higher-order logic with independent kernels. A theorem proved in one system can be cross-checked against the others. A soundness bug would need to exist simultaneously in multiple independent implementations to go undetected.


What About AI-Generated Proofs?
=================================

AI systems can generate proof steps: suggesting tactics, filling in lemma statements, performing proof search. The LCF architecture is exactly why this is safe. The AI is untrusted: it suggests steps, and the kernel checks them. If the AI produces an invalid step, the kernel rejects it. The AI is just another untrusted component, subject to the same gatekeeping as any tactic or decision procedure.

This is a different situation from using AI to generate code, where no kernel exists to catch mistakes. In code generation, a subtle error propagates silently. In theorem proving, every step passes through the kernel. Checking is independent of generation, and the checker is the smallest, most auditable component in the system.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`hol-vs-fol`, what higher-order logic can express and why the choice of logic matters

   :doc:`verification-gap`, why the TCB extends beyond the prover to the compiler and execution platform
