.. _formal-verification:

Formal Verification
###################

Formal verification (using mathematical proof to establish that software behaves correctly) is used in hardware design, avionics, and other fields where the cost of failure justifies the cost of proof. (Intel adopted formal verification for processor arithmetic after the `Pentium FDIV bug <https://en.wikipedia.org/wiki/Pentium_FDIV_bug>`_; the `DO-178C <https://en.wikipedia.org/wiki/DO-178C>`_ standard for airborne systems recognizes formal methods as a means of achieving the highest assurance levels.) Smart contracts have the same profile: they are autonomous, immutable after deployment, and operate in adversarial environments. A single bug is publicly exploitable by anyone, instantly, with no possibility of intervention. The case for formal verification here is at least as strong.


What Formal Verification Is
============================

Testing checks that a program produces the right output for specific inputs. `Formal verification <https://ethereum.org/developers/docs/smart-contracts/formal-verification/>`_ proves it produces the right output for *all* inputs. The difference is between "we tried a thousand cases and none failed" and "we have a mathematical proof that no case can fail."

More precisely: formal verification is the construction of a machine-checked proof that a program's behavior (as defined by a formal model of the language's semantics) satisfies a *specification*, a formal statement of what the program should do. The proof is checked by software, not by human judgment, and it covers every possible execution, including edge cases no one thought to test. Once a property is proven, it is settled permanently. It does not need to be re-checked, and it cannot be invalidated by a new test case or a new attacker.

Testing, fuzzing, auditing, and static analysis can find bugs. Formal verification can prove their absence. The guarantee is strictly stronger: a property established by formal proof holds universally, not just for the cases that happened to be checked. For a deeper treatment of this distinction, see :doc:`testing-vs-proving`.

.. tip::

   Testing shows the presence of bugs. Formal verification proves their absence.


Approaches to Smart Contract Verification
==========================================

When verifying a smart contract, bugs can hide at multiple levels. A proof at one level does not cover the others. Each is a distinct trust assumption:

.. list-table:: Levels of Verification
   :widths: 30 70
   :header-rows: 0

   * - **Contract properties**
     - Does this contract do what it should?
   * - **Language semantics**
     - Does the language mean what we think?
   * - **Compiler correctness**
     - Does the bytecode match the source?
   * - **Execution platform (EVM)**
     - Does the VM execute bytecode correctly?

Most verification tools today operate only at the top level, proving properties about a specific contract. But those proofs implicitly trust that the language semantics are correct, that the compiler preserved meaning during translation, and that the EVM executes as specified. Each unverified level is a gap where bugs can hide undetected. See :doc:`verification-gap` for why this matters.

The approaches in use today also differ in what they can prove at any given level, and in how much you can trust the result.

Automated Solver-Based Verification
------------------------------------

The most common approach translates contract code and specifications into logical formulas and uses `SMT solvers <https://en.wikipedia.org/wiki/Satisfiability_modulo_theories>`_ to check them automatically. We write properties like "the total supply is preserved by this function" and the tool either verifies them or finds a counterexample, typically in minutes. This is accessible, practical, and effective for a significant class of properties.

SMT solvers have fundamental limitations. They treat verification as a single-pass search over a formula and cannot perform `induction <https://en.wikipedia.org/wiki/Mathematical_induction>`_ (the technique for proving a property holds across an *unbounded* number of cases). A solver can check "this invariant holds after 5 transactions," maybe 100, but not "after any number of transactions." This is not an engineering limitation. Verification conditions for general smart contract properties routinely fall outside the `decidable fragments <https://en.wikipedia.org/wiki/Undecidable_problem>`_ these solvers handle. There is also a trust consideration: when a solver reports "verified," the claim is only as reliable as the solver implementation itself. For a detailed comparison of what SMT solvers can and cannot express, see :doc:`hol-vs-fol`.

Type-System-Based Approaches
-----------------------------

Some languages build verification into the type system, using techniques like linear types to enforce that assets cannot be duplicated or destroyed. The result is useful compile-time guarantees, produced automatically.

The limitation is expressiveness. Type systems enforce *structural* properties (resource safety, access control) but cannot express *functional* correctness: "this AMM implements the constant-product formula correctly" or "this voting mechanism is fair under all participation patterns."

Interactive Theorem Proving
----------------------------

In interactive theorem proving, the meaning of programs is defined as a mathematical object inside a `proof assistant <https://en.wikipedia.org/wiki/Proof_assistant>`_, and proofs of correctness are constructed with human guidance and checked by machine.

The trust model is different from solver-based tools. The proof assistant has a small, trusted *kernel* whose sole job is to verify that each proof step follows logically from the previous ones. Everything else in the system is untrusted: automation, tactics, even AI-generated proof attempts. If any component produces a faulty step, the kernel rejects it. No software is bug-free, but a kernel of a few thousand lines that has been scrutinized for decades is a categorically different trust proposition from a solver of hundreds of thousands of lines. Correctness depends on the smallest, most auditable component in the system. See :doc:`trusted-computing-base` for a deeper treatment of why this matters and how it works.

This approach can express and prove anything statable in `higher-order logic <https://en.wikipedia.org/wiki/Higher-order_logic>`_, a logical system that can quantify not just over values but over functions, predicates, and properties themselves, enabling unbounded induction, quantification over all possible inputs and contracts, full functional correctness, and compositional reasoning across interacting systems. Where a type system can ensure an AMM's liquidity tokens are not duplicated, interactive theorem proving can prove that the AMM correctly implements the constant-product invariant across any sequence of swaps, deposits, and withdrawals, including under adversarial reordering.

The cost is human effort. But a formal semantics and verified compiler are infrastructure. Built once, they benefit every contract written in the language. And the proofs they produce are permanent: they do not expire, do not depend on auditor judgment, and do not need to be re-done when the tool is updated.

.. note::

   Interactive theorem proving is the only approach that can address all four verification levels in the table above.


Formal Verification with Vyper
===============================

Formal verification makes demands on the language being verified. The simpler a language's semantics, the more tractable its proofs. A language that guarantees termination allows its semantics to be defined as a straightforward mathematical function. A language without escape hatches into raw bytecode can be modeled completely at the source level. A language without inheritance, overloading, or metaprogramming has less surface area for a formal model to cover.

Vyper meets these requirements. All loops are bounded. Recursion is prohibited. There is no inheritance, no inline assembly, no operator overloading, no function modifiers. Every construct has clear, predictable semantics. These design choices were made for security and readability, but they also make Vyper a strong target for formal verification. Attempting this for a language with unbounded recursion, inheritance hierarchies, and inline assembly would require a formal model of far greater complexity, which is why no such model exists for any general-purpose smart contract language.

The `Verifereum <https://verifereum.org>`_ project is building a full-stack verified pipeline for Vyper in the `HOL4 <https://hol-theorem-prover.org>`_ proof assistant. The work is in `vyper-hol <https://github.com/verifereum/vyper-hol>`_. The components:

**Formal language semantics.**
A precise mathematical definition of what every Vyper construct means. Without this, any verification of Vyper contracts is implicitly trusting an informal understanding of the language. With it, proofs are grounded in a machine-checked definition. The formal semantics is both *executable* and *provable*: it can be run on concrete inputs to validate it against the official Vyper test suite, and it can be reasoned about to prove theorems. It currently covers the core of Vyper's execution model and passes the ``functional/codegen`` section of the official test suite. We have a machine-checked proof that the Vyper interpreter terminates on all inputs, confirming Vyper's totality as a mathematical theorem, not a documentation claim. Development was supported by the Ethereum Foundation's `Ecosystem Support Program <https://esp.ethereum.foundation>`_.

**Contract-level verification.**
Proving properties about specific contracts: invariant preservation, functional correctness, access control enforcement, grounded in the formal semantics. See :doc:`correctness-proofs` for what a correctness proof is and how confidence in specifications is established.

**Compiler verification.**
Proving that the Vyper compiler correctly translates source to EVM bytecode. Any property proven about source code would then be guaranteed to hold in deployed bytecode. See :doc:`compiler-verification` for what end-to-end compiler verification means and why it matters.

**EVM formalization.**
A formal model of the Ethereum Virtual Machine, built in the main `Verifereum repository <https://github.com/verifereum/verifereum>`_ and already used by the Vyper formal semantics.

The formal semantics and EVM formalization are available now. The formal semantics is available today and provides the foundation for contract-level verification of Vyper contracts. Compiler verification is a long-term effort; see :doc:`compiler-verification` for the precedents and what it requires.

When connected, these layers form a chain of proofs from source-level properties to on-chain execution. No such pipeline exists for any smart contract language. We are building it, component by component. For a live roadmap, see the `vyper-hol issue tracker <https://github.com/verifereum/vyper-hol/issues>`_.


Why It Matters
===============

The value of formal verification scales with the cost of failure.

Protocols holding user funds face the most direct risk: a single vulnerability can drain an entire contract, and no amount of insurance or incident response can reverse an on-chain exploit. Formal verification is the only technique that can prove an invariant holds under all conditions, including adversarial conditions no one anticipated.

For security professionals, formal methods change the scope of what is achievable. An audit backed by formally verified language semantics and a verified compiler has a foundation that manual code review alone cannot provide. The formal semantics serves as the reference specification, the authoritative definition of what the code means, that rigorous auditing benefits from but rarely has.

For regulated institutions and compliance teams, a machine-checked proof is an auditable artifact. It records exactly what was proved, what was assumed, and what logic was used. It does not depend on the judgment or reputation of any individual. As regulators increasingly require evidence of software assurance, formal verification provides a standard of evidence already accepted in avionics and hardware design.

Every contract deployed without formal verification of its critical invariants is a contract whose correctness rests entirely on testing and human review. For high-value contracts, that is the status quo. It does not have to be.

.. seealso::

   `vyper-hol <https://github.com/verifereum/vyper-hol>`_, formal semantics of Vyper in HOL4

   `Verifereum <https://verifereum.org>`_, formal verification of Ethereum applications

   `HOL4 <https://hol-theorem-prover.org>`_, the proof assistant used


.. toctree::
   :maxdepth: 1
   :caption: Deep Dives

   testing-vs-proving
   correctness-proofs
   trusted-computing-base
   verification-gap
   compiler-verification
   hol-vs-fol
