.. _hol-vs-fol:

Higher-Order Logic vs. First-Order Logic
#########################################

The logic you use determines what you can express and prove. First-order logic (FOL) quantifies over values. Higher-order logic (HOL) quantifies over functions, predicates, and properties themselves. SMT solvers operate in decidable fragments of first-order logic. Proof assistants like HOL4 work in higher-order logic. This difference determines the ceiling of what each approach can verify.


What First-Order Logic Can Say
================================

First-order logic quantifies over *individuals*: values, numbers, addresses, amounts. Statements like these are first-order:

- "For all amounts *a*, if *a* is a positive integer, then *a* + 1 > *a*."
- "For all amounts *a*, if *a* <= ``balance[sender]``, then ``transfer(sender, receiver, a)`` decreases ``balance[sender]`` by *a*."

The quantifier "for all" ranges over values. The functions and predicates (``balance``, ``transfer``, ``<=``) are fixed; only their arguments vary.

`SMT solvers <https://en.wikipedia.org/wiki/Satisfiability_modulo_theories>`_ operate in decidable fragments of first-order logic, enriched with specific theories: integers, bit-vectors, arrays, and other data types. Within these fragments, solvers check properties fully automatically, typically in seconds or minutes.


What First-Order Logic Cannot Say
===================================

First-order logic with set-theoretic axioms (such as ZFC) can encode quantification over functions by representing them as sets of ordered pairs. But this encoding takes the resulting formulas outside the decidable fragments that SMT solvers rely on. The first-order logic fragments used by SMT solvers cannot quantify over properties, functions, or predicates. This limits what solver-based verification can express.

**Unbounded induction.** The statement "for all properties P, if P(0) holds and P(n) implies P(n+1) for all n, then P(n) holds for all n" quantifies over the property P (a second-order statement). In first-order logic, induction is an axiom *schema*: a separate axiom for each specific property. An SMT solver can check that a particular invariant holds after a bounded number of steps, but it cannot perform unbounded induction: it cannot prove the invariant holds after *any* number of steps.

**Quantification over functions.** "There exists a function *f* such that *f* is a simulation relation between these two programs" requires quantifying over functions. Simulation relations are central to compiler verification (see :doc:`compiler-verification`). They establish that compiled code behaves like its source. SMT solvers cannot express this claim.

**Quantification over contracts.** "For all contract implementations satisfying this interface, composing them with this contract preserves invariant X" requires quantifying over implementations, which are functions. SMT solvers can check the property for specific known implementations, but not for all possible ones.


What Higher-Order Logic Adds
==============================

Higher-order logic allows direct quantification over functions, predicates, sets, and properties.

**Induction becomes a single axiom.** "For all P, if P(0) and (for all n, P(n) implies P(n+1)), then for all n, P(n)." This one statement covers all properties. No schema needed. We state and prove it once; it applies universally.

**Universal quantification over programs and contracts.** Higher-order logic can express:

- "For all contract implementations satisfying this interface, composing them with this contract preserves invariant X." Quantification over implementations (over functions).
- "For all invariants preserved by individual operations, the invariant is preserved by any sequence of operations." Quantification over invariants (over predicates).
- "There exists a simulation relation between the source-level semantics and the compiled bytecode." Quantification over relations (over functions of two arguments).

These are the natural formulations of the properties that matter most for smart contract security and compiler correctness.


Why This Matters for Smart Contracts
======================================

The properties that matter most for high-value contracts are often higher-order in nature. First-order logic as used by SMT solvers can express specific instances. Higher-order logic can express the general claims.

**Reentrancy safety.** "This contract is safe against reentrancy from *any* external contract" requires quantifying over all possible calling contracts: all possible functions that could be invoked during execution. An SMT solver can check reentrancy safety against specific known contracts, but not against all possible ones.

**Invariant preservation across unbounded sequences.** "This invariant holds after *any* sequence of valid transactions" requires unbounded induction. An SMT solver can check that the invariant holds after 5 transactions, or 100, but not after an arbitrary number.

**Compiler correctness.** "This compiler preserves semantics for *all* source programs" requires quantifying over programs, structured data (abstract syntax trees) that can be represented as functions. See :doc:`compiler-verification`.

**Compositional reasoning.** "If modules A and B each satisfy their specifications, then their composition satisfies the system specification" requires quantifying over specifications and modules. This is how large systems are verified: components are verified separately and composition is proved correct.


Concrete Example: Invariant Preservation
==========================================

**What an SMT solver can check:** "After calling ``transfer``, then ``deposit``, then ``withdraw``, the total supply is unchanged." This checks one specific sequence of three calls.

**What we can prove in HOL4:** "For all states *s* and all sequences of valid transactions *T*, if ``total_supply_invariant(s)`` holds, then ``total_supply_invariant(apply(T, s))`` holds." This covers all sequences of any length. It is proved once and holds permanently.

The higher-order statement subsumes the first-order check and every other specific sequence. It is not a stronger test. It is a different kind of result: a mathematical theorem, not a checked instance.


The Trade-Off
==============

The expressiveness of higher-order logic comes at a cost.

The decidable fragments of first-order logic used by SMT solvers permit fully automated proof search: state a property, get a result. Higher-order logic is strictly more expressive, but full automation is not possible in general. This follows from `Gödel's incompleteness theorems <https://en.wikipedia.org/wiki/G%C3%B6del%27s_incompleteness_theorems>`_ and the undecidability of higher-order unification (a mathematical fact, not an engineering limitation). Proofs in higher-order logic require human guidance: structuring the argument, choosing the right induction, stating intermediate lemmas.

No system can have both full automation and full expressiveness.

.. list-table::
   :widths: 30 35 35
   :header-rows: 1

   * - Property type
     - SMT solver (FOL fragments)
     - Proof assistant (HOL)
   * - Bounded, specific properties
     - Fully automated
     - Possible but unnecessary
   * - Unbounded invariants
     - Cannot express
     - Requires human guidance
   * - Compiler correctness
     - Cannot express
     - Requires human guidance
   * - Compositional properties
     - Cannot express
     - Requires human guidance

The approaches are complementary: for bounded, specific properties ("does this function revert on invalid input?"), SMT-based tools are practical and effective. For universal, compositional, unbounded properties ("does this invariant hold forever, against all callers, under all conditions?"), higher-order logic is necessary.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`testing-vs-proving`, the distinction between checking instances and proving universals

   :doc:`trusted-computing-base`, why the proof assistant's kernel makes higher-order proofs trustworthy
