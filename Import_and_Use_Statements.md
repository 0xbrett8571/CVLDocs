# Import and Use Statements — Certora Prover Documentation 0.0 documentation

# Import and Use Statements[](#import-and-use-statements "Link to this heading")

Contents of additional spec files can be imported using the `import` command. Some parts of the imported spec files are implicitly included in the importing spec file, while others such as rules and invariants must be explicitly `use`d. Functions, definitions, filters, and preserved blocks of the imported spec can be overridden by the importing spec. If a spec defines a function and uses it (e.g. in a rule or function), and another spec imports it and overrides it, uses in the imported spec use the new version.

## Examples[](#examples "Link to this heading")

```solidity
import "base.spec"; // Imports all the elements defined in "base.spec", excluding rules and invariants;

use rule threeParametricRuleInBase filtered { f -> isPlusSevenSomeUInt(f), // Override the filter of f
                                            g -> filterDef(g) // Add a filter to g (using an overridden definition)
                                            // The filter of h is inherited from the base spec
                                          }

use invariant invInBase // Imports the invariant invInBase
{
    preserved minusSevenSomeUInt() { // Overriding the preserved block in base.spec
      plusSevenSomeUInt();
    }

    preserved plusSevenSomeUInt() { // Adding a preserved block
      minusSevenSomeUInt();
	  }

    preserved { // Overriding the generic preserved block in base.spec
    }
}

methods { // The methods in the methods block in base.spec are automatically added to this block
  function plusSevenSomeUInt() external returns (bool) envfree;
}

use rule ruleInBase; // Imports the rule ruleInBase

use rule parametricRuleInBase; // Imports the rule parametricRuleInBase

definition isPlusSevenSomeUInt(method h) returns bool = h.selector == sig:plusSevenSomeUInt().selector;

use rule twoParametricRuleInBase filtered { f -> isPlusSevenSomeUInt(f), // Override the filter of f
                                            g -> filterDef(g) // Add a filter to g (using an overridden definition)
                                          }

override definition filterDef(method f) returns bool = f.isFallback;

override function callF(env eF, calldataarg args, method f) {  // Should make ruleInBase fail
  require eF.msg.value > 0;
  f@withrevert(eF, args);
  assert !lastReverted;
}
```
        

## Syntax[](#syntax "Link to this heading")

The syntax for `import` and `use` statements is given by the following [EBNF grammar](overview.html#ebnf-syntax):

import ::= "import" string

use ::= "use" "rule" id
        \[ "filtered" "{" id "->" expression { "," id "->" expression } "}" \]
      | "use" "builtin" "rule" id
      | "use" "invariant" id \[ "filtered" "{" id "->" expression "}" \] \[ "{" { preserved\_block } "}" \]

See [Basic Syntax](basics.html) for the `string` and `id` productions, [Expressions](expr.html) for the `expression` production, and [Invariants](invariants.html) for the `filtered` and `preserved_block` production.
