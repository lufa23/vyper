.. _testing-vs-proving:

Testing vs. Proving
###################

Testing and auditing sample a program's behavior. Theorem proving characterizes it completely. These are not points on a spectrum. They are categorically different kinds of knowledge.


What Testing Tells You
=======================

A test checks that a program produces the expected output for a specific input. A test suite checks many inputs. But the input space of a smart contract (all possible states, all possible call sequences, all possible interacting contracts) is effectively infinite. A test suite, no matter how large, covers a vanishingly small fraction of it.

Even 100% code coverage does not mean 100% of behaviors are checked. Code coverage measures which *lines* execute, not which *states* are reached or which combinations of conditions occur. A function can have full line coverage while entire categories of behavior go unexercised.

Consider a token contract that passes all unit tests for transfers, approvals, minting, and burning. Every line is covered. But a specific combination of transfer amounts and approval sequences (one the test authors never considered) allows a double-spend. The test suite did not include that combination. An attacker did.

`Fuzzing <https://en.wikipedia.org/wiki/Fuzzing>`_ (property-based testing) improves on handwritten tests by generating random inputs and checking that stated properties hold. It explores more of the input space and finds bugs faster. But it is still sampling. A fuzzer that runs for a week without finding a violation has not proved the property holds. It has failed to find a counterexample within its time budget.


What an Audit Tells You
========================

An audit is a human reading code and reasoning about it. Experienced auditors catch classes of bugs that automated tools miss: subtle economic exploits, flawed incentive designs, logic errors arising from misunderstanding the problem domain.

But auditing is inherently non-exhaustive. The auditor examines scenarios they think of, constrained by experience, attention, and time budget. Scenarios they do not think of go unexamined.

Audits do not compose. Auditing contract A and contract B separately does not audit their interaction. A property that holds for each contract in isolation can fail when they are composed. Cross-contract interactions are where many exploits occur.

Consider a lending protocol that integrates with an AMM for liquidations. The lending contract is audited. The AMM is audited. But the interaction (a liquidation that triggers a swap that triggers a callback) is not covered by either audit.

An audit report is an opinion, backed by expertise and diligence. A formal proof is a mathematical artifact that can be machine-checked by anyone, at any time, with no dependence on the prover's judgment or reputation.

These three limitations are connected. Audits are non-exhaustive because human attention is finite. They do not compose because the auditor's mental model of one contract does not extend to its interactions with another. The result is an opinion because there is no mechanical check on the reasoning. Formal proofs address all three: they cover all inputs, they compose mathematically, and they are checked by machine.


What a Theorem Tells You
==========================

A theorem proved in a system like `HOL4 <https://hol-theorem-prover.org>`_ is a statement that holds for *all* inputs, *all* states, and *all* execution paths. It is not a sample. It is a complete characterization of the property in question.

The proof is checked by machine, by the proof assistant's kernel (see :doc:`trusted-computing-base`). The kernel verifies that every step follows from the axioms of the logic. Human judgment is not involved in checking.

Theorems compose. If we prove property P about module A, property Q about module B, and that P and Q together imply R about the combined system, then R is proven. The proof of the whole follows from the proofs of the parts.

Theorems are permanent. They do not need to be re-done when the team changes, when the auditor's contract expires, or when a new attack vector is discovered. A proven property holds until the specification or the code changes. (If a new attack falls outside the specification, that is a specification problem (see :doc:`correctness-proofs`), not a limitation of the proof.)

The difference, concretely: a test suite checks "transfer works correctly for amounts 0, 1, 100, and ``MAX_UINT256``." A theorem proves "for all amounts and all states satisfying the precondition, transfer preserves the total supply invariant." The theorem subsumes every test case, including the ones no one wrote.


The Asymmetry
==============

Edsger Dijkstra stated the core point: "Program testing can be used to show the presence of bugs, but never to show their absence."

No amount of testing investment can produce the guarantee that a single theorem provides. More tests, longer fuzzing campaigns, and additional auditors increase confidence. But the result remains "we looked hard and found nothing." A formal proof produces a categorically different result: "nothing exists to be found."

For high-value contracts, the question is not "how much testing is enough." There is no amount that is enough to guarantee the absence of a specific class of bug. The question is whether we have proved the properties that matter.

.. list-table::
   :widths: 20 40 40
   :header-rows: 1

   * - Method
     - What it establishes
     - Limitation
   * - Testing
     - Correct behavior for specific inputs
     - Cannot cover the full input space
   * - Fuzzing
     - No violation found under random sampling
     - Absence of evidence is not evidence of absence
   * - Auditing
     - Expert judgment that code appears correct
     - Non-exhaustive, does not compose, not machine-checkable
   * - Formal proof
     - Correct behavior for all inputs, all states, all paths
     - Only as strong as the specification (see :doc:`correctness-proofs`)

Testing and auditing remain essential: for finding bugs before investing in proofs, and for exploring behaviors in areas not yet covered by a specification. They cannot replace proofs for the properties that matter most. Testing finds bugs. Proving eliminates them.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`correctness-proofs`, what a specification is and how to build confidence that it captures the right properties

   :doc:`hol-vs-fol`, why the logic used determines what can be proved
