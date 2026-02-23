.. _correctness-proofs:

Correctness Proofs
##################

A correctness proof establishes that code matches a specification. The proof is machine-checkable. The hard question is whether the specification captures what you actually need, and there are systematic ways to build confidence in that.


What a Correctness Proof Is
=============================

A correctness proof has three components:

**A specification:** a formal statement of what the program should do, written in the logic of the proof assistant (higher-order logic in HOL4).

**A program:** the code being verified. For Vyper, this is the source code represented within the formal semantics defined in `vyper-hol <https://github.com/verifereum/vyper-hol>`_.

**A proof:** a machine-checked derivation showing that the program's behavior, as defined by the formal semantics, satisfies the specification under all possible inputs and states.

The proof establishes correctness *relative to a formal model* of the language's semantics. It covers all inputs and states within that model. It does not directly prove properties about the deployed contract. It proves properties about the model of the contract. The model's fidelity to the real system is a separate question, addressed by validating the formal semantics against the language's test suite and, ultimately, by verifying the compiler (see :doc:`compiler-verification` and :doc:`verification-gap`).

A specification for a transfer function might state: "for all states *s* and amounts *a*, if ``balance(sender, s) >= a``, then after ``transfer(sender, receiver, a)``, the sender's balance decreases by *a*, the receiver's balance increases by *a*, and the total supply is unchanged." The proof shows that the implementation satisfies this for every state and amount, not by testing examples, but by mathematical derivation. The proof assistant's kernel checks every step (see :doc:`trusted-computing-base`).


What Kinds of Properties Can Be Proved
========================================

**Safety properties: "nothing bad happens."** Invariants that hold across all executions. Examples: total supply is preserved by every function, only the owner can call administrative functions, no withdrawal exceeds the caller's balance. Safety properties are the most common starting point. They are simple to state and directly address the most damaging classes of bugs.

**Functional correctness: "this function computes exactly the right result."** The strongest form of specification: it fully describes a function's input-output behavior. For an AMM, functional correctness means the swap function implements the constant-product formula correctly for all inputs: the output satisfies ``x * y = k`` after every trade, accounting for fees, rounding, and edge cases.

**Relational properties: "these two implementations produce the same result."** Useful for compiler verification (source and bytecode behave identically; see :doc:`compiler-verification`) and for proving that a contract upgrade preserves behavior. The specification relates two programs rather than describing one in absolute terms.

**Liveness properties: "something good eventually happens."** Harder to state for smart contracts, since contracts are reactive (they respond to calls rather than running continuously). But certain behaviors have liveness aspects: an auction must eventually settle, a timelock must eventually release funds after its delay period.


The Specification Problem
==========================

A proof is only as meaningful as its specification. If we prove the wrong property, the proof is mathematically correct but practically useless.

This is the most common sophisticated critique of formal verification, and it is legitimate. Consider: we prove that ``transfer`` preserves total supply. But the specification does not account for a reentrancy path through a callback in a different function. The total supply invariant holds, and yet the contract can be drained. The proof is correct. The specification was incomplete.

This is not a weakness unique to formal verification. An auditor can miss the same reentrancy path. A test suite can fail to include the right sequence of calls. Every verification approach depends on correctly identifying what needs to be checked.

The difference is that formal verification makes this dependency *explicit*. The specification is a written document. We can read it, review it, and ask: "does this cover the behavior we care about?" With testing, the coverage question is harder to even pose: "which inputs did we not try?" has no tractable answer. With a formal specification, the question is concrete: "which behaviors are not specified?"


Building Confidence in Specifications
=======================================

There is no way to mechanically guarantee that a specification is "right." Rightness is a question about the relationship between a formal object and human intent, and that relationship is inherently informal. But there are systematic ways to build confidence.

**Specification review.** A specification in higher-order logic is a readable formal document, shorter and more abstract than the implementation. Domain experts and formal methods specialists review it the way they review code, checking that each stated property matches the intended behavior. Writing the right specification requires understanding both formal methods and the application domain; the best specifications are written collaboratively.

**Testing against the specification.** Before investing in proofs, run the specification against concrete test cases to validate it matches expected behavior. When the formal semantics is executable (as in vyper-hol), we can evaluate the specification on specific inputs and compare results. This catches errors in the specification early, before proof effort is spent.

**Incremental strengthening.** Start with safety invariants (total supply is preserved, access control is enforced), then progress toward full functional correctness. Each level catches a broader class of bugs. Gaps in weaker specifications are often revealed during the process of strengthening them.

**Specification patterns.** Common contract patterns (ERC-20 tokens, AMMs, governance mechanisms) could over time develop standard specifications reviewed and validated by the community. A well-tested ERC-20 specification could serve as a starting point for any token contract.


How to Know You've Proved Enough
=================================

There is no mechanical answer to "have we specified everything?", just as there is no mechanical answer to "have we tested everything?" Any specification selects which behaviors to constrain.

But the explicit nature of formal specifications makes gaps *visible*. We can enumerate what has been proven and what has not. A specification that covers total supply preservation but not access control makes that gap apparent to anyone who reads it. An audit that omitted a check for the same issue would not make the gap visible: it would be a scenario the auditor did not consider.

The strongest approach combines formal proofs with testing and fuzzing. Proofs cover the properties we have identified as critical, the invariants whose violation would be catastrophic. Testing and fuzzing explore the broader behavior space, finding issues in areas not yet specified. When fuzzing discovers a violation, it may indicate a property worth formalizing and proving. The approaches are complementary.

Formal proofs require more upfront effort than testing or auditing. Writing a specification and constructing a proof takes time proportional to the complexity of the property and the contract. But the cost profile is different from auditing: the proof is permanent, it does not need to be repeated when the team changes, and it composes with other proofs. For a contract holding significant value over a long time horizon, the one-time cost of a formal proof is small relative to the cost of a single exploit.

.. tip::

   Formal verification makes assumptions explicit. That is strictly better than leaving them implicit, even when the assumptions turn out to be incomplete.

.. seealso::

   :doc:`index`, overview of formal verification for Vyper

   :doc:`testing-vs-proving`, why proofs provide a categorically stronger guarantee than testing

   :doc:`verification-gap`, why the level at which you prove matters
