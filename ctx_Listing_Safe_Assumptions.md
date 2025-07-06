# Listing Safe Assumptions — Certora Prover Documentation 0.0 documentation
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

The _uniqueness of signature_ axiom is not sound. There are some rare cases where `ecrecover(h2, v, r, s)` will not return zero for the wrong hash. This is why you must always check that the address returned by `ecrecover` is the correct one.

Context:
--------------------------------------------

In the example, we focus on the `ecrecover` function used for signature verification. The objective is to articulate and validate key assumptions associated with this function to bolster the security of smart contracts.

Importance of Listing Safe Assumptions:
----------------------------------------------------------------------------------------------------------

1.  **Clarity and Documentation:**
    
    *   The design pattern begins by explicitly listing assumptions related to the `ecrecover` function. This serves as clear documentation for developers, auditors, and anyone reviewing the spec. Clarity in assumptions enhances the understanding of expected behavior.
        
2.  **Preventing Unexpected Behavior:**
    
    *   The axioms established in the example, such as the zero message axiom and uniqueness of signature axiom, act as preventive measures against unexpected behavior. They set clear expectations for how the `ecrecover` function should behave under different circumstances, neglect all the counter-examples that are not relevant to the function intended behavior.
        
3.  **Easy To Use:**
    
    *   By encapsulating assumptions within the CVL function, this design pattern allow us to easily use those assumptions in any rule or invariant we desire.
        

In conclusion, the “Listing Safe Assumptions” design pattern, exemplified through the `ecrecover` function in the provided example, serves a broader purpose in specs writing. It systematically documents assumptions, prevents unexpected behaviors, and offers ease of use throughout the rules and invariants.
