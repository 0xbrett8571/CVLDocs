The “Listing Safe Assumptions” design pattern introduces a structured approach to document and validate essential assumptions. Let’s delve into the importance of this design pattern using the provided example.

```
function ecrecoverAxioms() {
  // zero value:
  require (forall uint8 v. forall bytes32 r. forall bytes32 s. ecrecover(to_bytes32(0), v, r, s) == 0);
  // uniqueness of signature
  require (forall uint8 v. forall bytes32 r. forall bytes32 s. forall bytes32 h1. forall bytes32 h2.
    h1 != h2 => ecrecover(h1, v, r, s) != 0 => ecrecover(h2, v, r, s) == 0);
  // dependency on r and s
  require (forall bytes32 h. forall uint8 v. forall bytes32 s. forall bytes32 r1. forall bytes32 r2.
    r1 != r2 => ecrecover(h, v, r1, s) != 0 => ecrecover(h, v, r2, s) == 0);
  require (forall bytes32 h. forall uint8 v. forall bytes32 r. forall bytes32 s1. forall bytes32 s2.
    s1 != s2 => ecrecover(h, v, r, s1) != 0 => ecrecover(h, v, r, s2) == 0);
}

rule zeroValue() {
    ecrecoverAxioms();
    bytes32 msgHash; uint8 v; bytes32 r; bytes32 s; 
    assert ecrecover(to_bytes32(0), v, r, s ) == 0;
}

rule ownerSignatureIsUnique () {
    ecrecoverAxioms();
    bytes32 msgHashA; bytes32 msgHashB;
    uint8 v; bytes32 r; bytes32 s; 
    address addr; 
    require msgHashA != msgHashB; 
    require addr != 0;
    assert isSigned(addr, msgHashA, v, r, s ) => !isSigned(addr, msgHashB, v, r, s );
}
```
Warning

The uniqueness of signature axiom is not sound. There are some rare cases where ecrecover(h2, v, r, s) will not return zero for the wrong hash. This is why you must always check that the address returned by ecrecover is the correct one.

Context:
In the example, we focus on the ecrecover function used for signature verification. The objective is to articulate and validate key assumptions associated with this function to bolster the security of smart contracts.

Importance of Listing Safe Assumptions:
Clarity and Documentation:

The design pattern begins by explicitly listing assumptions related to the ecrecover function. This serves as clear documentation for developers, auditors, and anyone reviewing the spec. Clarity in assumptions enhances the understanding of expected behavior.

Preventing Unexpected Behavior:

The axioms established in the example, such as the zero message axiom and uniqueness of signature axiom, act as preventive measures against unexpected behavior. They set clear expectations for how the ecrecover function should behave under different circumstances, neglect all the counter-examples that are not relevant to the function intended behavior.

Easy To Use:

By encapsulating assumptions within the CVL function, this design pattern allow us to easily use those assumptions in any rule or invariant we desire.

In conclusion, the “Listing Safe Assumptions” design pattern, exemplified through the ecrecover function in the provided example, serves a broader purpose in specs writing. It systematically documents assumptions, prevents unexpected behaviors, and offers ease of use throughout the rules and invariants.

Require Invariants: Proving inter-dependent invariants
TherequireInvariant statements can be used to establish crucial conditions that must persist throughout the execution of a smart contract. Let’s explore the usefulness of the requireInvariant statement and illustrate its application using the provided example.

```
invariant totalSharesLessThanDepositedAmount()
    totalSupply() <= depositedAmount();

invariant totalSharesLessThanUnderlyingBalance()
    totalSupply() <= underlying.balanceOf(currentContract)
    {
        preserved with(env e) {
            require e.msg.sender != currentContract;
            requireInvariant totalSharesLessThanDepositedAmount();
            requireInvariant depositedAmountLessThanContractUnderlyingAsset();
        }
    }

rule sharesRoundingTripFavoursContract(env e) {
    uint256 clientSharesBefore = balanceOf(e.msg.sender);
    uint256 contractSharesBefore = balanceOf(currentContract);

    requireInvariant totalSharesLessThanDepositedAmount();
    require e.msg.sender != currentContract;

    uint256 depositedAmount = depositedAmount();

    uint256 clientAmount = withdraw(e, clientSharesBefore);
    uint256 clientSharesAfter = deposit(e, clientAmount);
    uint256 contractSharesAfter = balanceOf(currentContract);
    assert (clientAmount == depositedAmount) => clientSharesBefore <= clientSharesAfter; 
    
    // all other states
    assert (clientAmount < depositedAmount) => clientSharesBefore >= clientSharesAfter;
    assert contractSharesBefore <= contractSharesAfter;
}
```
Importance of Require Invariants:
Ensuring Invariant Consistency:

The totalSharesLessThanUnderlyingBalance invariant expands the validation scope to include the underlying balance of the current contract. By utilizing the previously established totalSharesLessThanDepositedAmount invariant as a prerequisite, the specification writer ensures that the total shares in the contract remain within the limits of the underlying asset balance. This requireInvariant approach effectively eliminates counterexamples in states that have already been proven to be unattainable.

Validation Through Rules:

The sharesRoundingTripFavoursContract rule showcases the application of require invariants to verify the behavior of share transactions. Similarly to the invariant example, the requireInvariant statements enable the specification writer to disregard counterexamples in states that are not relevant to the intended behavior.

In conclusion, the “Require Invariants” design pattern, as demonstrated through the provided example, offers a systematic methodology to define, validate, and uphold critical conditions within smart contract specifications. for more information, please visit the documentation.
