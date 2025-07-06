# Functions — Certora Prover Documentation 0.0 documentation
Certora Prover Documentation

CVL functions allow you to reuse parts of a specification, such as common assumptions, assertions, or basic calculations. Additionally they can be used for basic calculations and for function summaries.

Syntax
-----------------------------------------

The syntax for CVL functions is given by the following EBNF grammar:

```
function ::= [ "override" ]
             "function" id
             [ "(" params ")" ]
             [ "returns" type ]
             block

```


See Basic Syntax for the `id` production, Types for the `type` production, and Statements for the `block` production.

Examples
---------------------------------------------

*   Function with a return:
    
    ```
function abs_value_difference(uint256 x, uint256 y) returns uint256 {
    if (x < y) {
      return y - x;
    } else {
      return x - y;
    }
}

```

    
*   CVL function with no return
    
*   Overriding a function from imported spec
    

Using CVL functions
-------------------------------------------------------------------

CVL Function may be called from within a rule, or from within another CVL function.
