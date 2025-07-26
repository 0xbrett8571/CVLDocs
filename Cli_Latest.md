

The Certora Prover - A formal verification tool for smart contracts


Usage: /home/brett/.local/bin/certoraRun <Files> <Flags>

Acceptable files for EVM projects are Solidity files (.sol suffix), Vyper files (.vy suffix), or conf files (.con
f suffix)                                                                                                        
Flag Types

1. boolean (B): gets no value, sets flag value to true (false is always the default)
2. string (S): gets a single string as a value, note also numbers are of type string
3. list (L): gets multiple strings as a value, flags may also appear multiple times
4. map (M): collection of key, value pairs


┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃Flag                           ┃Type┃Description                          ┃Default                  ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━┩
│address                        │L   │Set the address of a contract to a   │Assigns addresses        │
│                               │    │given address                        │arbitrarily              │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│auto_dispatcher                │B   │automatically add `DISPATCHER(true)` │                         │
│                               │    │summaries for all calls with         │                         │
│                               │    │unresolved callees                   │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│build_cache                    │B   │Enable caching of the contract       │Compiles contract source │
│                               │    │compilation process                  │files from scratch each  │
│                               │    │                                     │time                     │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│compilation_steps_only         │B   │Compile the spec and the code without│Sends a request after    │
│                               │    │sending a verification request to the│source compilation and   │
│                               │    │cloud                                │spec syntax checking     │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│compiler_map                   │M   │Map contracts to the appropriate     │Uses the same compiler   │
│                               │    │compiler in case not all contract    │version for all contracts│
│                               │    │files are compiled with the same     │                         │
│                               │    │compiler version.                    │                         │
│                               │    │                                     │                         │
│                               │    │CLI Example:                         │                         │
│                               │    │  --compiler_map                     │                         │
│                               │    │A=solc8.11,B=solc8.9,C=solc7.5       │                         │
│                               │    │                                     │                         │
│                               │    │JSON Example:                        │                         │
│                               │    │  "compiler_map": {                  │                         │
│                               │    │    "A": "solc8.11",                 │                         │
│                               │    │    "B": "solc8.9",                  │                         │
│                               │    │    "C": "solc7.5"                   │                         │
│                               │    │  }                                  │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│contract_extensions            │M   │Dictionary of extension contracts.   │                         │
│                               │    │Format:                              │                         │
│                               │    │{                                    │                         │
│                               │    │  "ExtendedContractA: [              │                         │
│                               │    │    {                                │                         │
│                               │    │      "extension": "ExtenderA",      │                         │
│                               │    │      "exclude": [                   │                         │
│                               │    │        "method1",                   │                         │
│                               │    │        ...,                         │                         │
│                               │    │        "methodN"                    │                         │
│                               │    │      ]                              │                         │
│                               │    │    },                               │                         │
│                               │    │    {                                │                         │
│                               │    │      ...                            │                         │
│                               │    │    }                                │                         │
│                               │    │  ],                                 │                         │
│                               │    │  "ExtendedContractB: [              │                         │
│                               │    │    ...                              │                         │
│                               │    │  ],                                 │                         │
│                               │    │  ...                                │                         │
│                               │    │}                                    │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│contract_recursion_limit       │S   │Specify the maximum depth of         │0 - no recursion is      │
│                               │    │recursive calls verified for Solidity│allowed                  │
│                               │    │functions due to inlining            │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│dynamic_bound                  │S   │Set the maximum amount of times a    │0 - calling              │
│                               │    │contract can be cloned               │create/create2/new causes│
│                               │    │                                     │havocs that can lead to  │
│                               │    │                                     │non-useful counter       │
│                               │    │                                     │examples                 │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│dynamic_dispatch               │B   │Automatically apply the DISPATCHER   │Contract method          │
│                               │    │summary on newly created instances   │invocations on newly     │
│                               │    │                                     │created instances causes │
│                               │    │                                     │havocs that can lead to  │
│                               │    │                                     │non-useful counter       │
│                               │    │                                     │examples                 │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│exclude_method                 │L   │Filter out methods to be verified by │Verifies all public or   │
│                               │    │their signature                      │external methods. In     │
│                               │    │                                     │invariants pure and view │
│                               │    │                                     │functions are ignored    │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│exclude_rule                   │L   │Filter out the list of               │Verifies all rules and   │
│                               │    │rules/invariants to verify. Asterisks│invariants               │
│                               │    │are interpreted as wildcards         │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│foundry                        │B   │Verify all Foundry fuzz tests in the │                         │
│                               │    │current project                      │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│hashing_length_bound           │S   │Set the maximum length of otherwise  │224 bytes (7 EVM words)  │
│                               │    │unbounded data chunks that are being │                         │
│                               │    │hashed                               │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│ignore_solidity_warnings       │B   │Ignore all Solidity compiler warnings│Treats certain severe    │
│                               │    │                                     │Solidity compiler        │
│                               │    │                                     │warnings as errors       │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│independent_satisfy            │B   │Check each `satisfy` statement that  │For each `satisfy`       │
│                               │    │occurs in a rule while ignoring      │statement, assumes that  │
│                               │    │previous ones                        │all previous `satisfy`   │
│                               │    │                                     │statements were fulfilled│
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│link                           │L   │Link a slot in a contract with       │The slot can be any      │
│                               │    │another contract.                    │address, often resulting │
│                               │    │                                     │in unresolved calls and  │
│                               │    │Format:                              │havocs that lead to      │
│                               │    │  <Contract>:<field>=<Contract>      │non-useful counter       │
│                               │    │                                     │examples                 │
│                               │    │Example:                             │                         │
│                               │    │  Pool:asset=Asset                   │                         │
│                               │    │                                     │                         │
│                               │    │The field asset in contract Pool is a│                         │
│                               │    │contract of type Asset               │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│loop_iter                      │S   │Set the maximum number of loop       │A single iteration for   │
│                               │    │iterations                           │variable iterations      │
│                               │    │                                     │loops, all iterations for│
│                               │    │                                     │fixed iterations loops   │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│max_concurrent_rules           │S   │Set the maximum number of parallel   │Number of available CPU  │
│                               │    │rule evaluations. Lower values (e.g.,│cores.                   │
│                               │    │1, 2, or 4) may reduce memory usage  │                         │
│                               │    │in large runs. This can sometimes    │                         │
│                               │    │help to mitigate out of memory       │                         │
│                               │    │problems.                            │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│method                         │L   │Filter methods to be verified by     │Verifies all public or   │
│                               │    │their signature                      │external methods. In     │
│                               │    │                                     │invariants pure and view │
│                               │    │                                     │functions are ignored    │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│msg                            │S   │Add a message description to your run│No message               │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│multi_assert_check             │B   │Check each assertion statement that  │Stops after a single     │
│                               │    │occurs in a rule, separately         │violation of any         │
│                               │    │                                     │assertion is found       │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│multi_example                  │S   │Show several counter examples for    │Shows a single example   │
│                               │    │failed `assert` statements and       │                         │
│                               │    │several witnesses for verified       │                         │
│                               │    │`satisfy` statements                 │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│nondet_difficult_funcs         │B   │Summarize as NONDET all value-type   │Tries to prove the       │
│                               │    │returning difficult internal         │unsimplified code        │
│                               │    │functions which are view or pure     │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│nondet_minimal_difficulty      │S   │Set the minimal `difficulty`         │50                       │
│                               │    │threshold for summarization by       │                         │
│                               │    │`nondet_difficult_funcs`             │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│optimistic_contract_recursion  │B   │Assume the recursion limit is never  │May show counter examples│
│                               │    │reached in cases of recursion of     │where the recursion limit│
│                               │    │Solidity functions due to inlining   │is reached               │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│optimistic_fallback            │B   │Prevent unresolved external calls    │Unresolved external calls│
│                               │    │with an empty input buffer from      │with an empty input      │
│                               │    │affecting storage states             │buffer cause havocs that │
│                               │    │                                     │can lead to non-useful   │
│                               │    │                                     │counter examples         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│optimistic_hashing             │B   │Bound the length of data (with       │May show counter examples│
│                               │    │potentially unbounded length) to the │with hashing applied to  │
│                               │    │value given in `hashing_length_bound`│data with unbounded      │
│                               │    │                                     │length                   │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│optimistic_loop                │B   │Assume the loop halt conditions hold,│May produce a counter    │
│                               │    │after unrolling loops                │example showing a case   │
│                               │    │                                     │where loop halt          │
│                               │    │                                     │conditions don't hold    │
│                               │    │                                     │after unrolling          │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│optimistic_summary_recursion   │B   │Assume the recursion limit of        │Can show counter examples│
│                               │    │Solidity functions within a summary  │where the recursion limit│
│                               │    │is never reached                     │was reached              │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│override_base_config           │S   │Path to parent conf                  │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│packages                       │L   │Map packages to their location in the│Takes packages mappings  │
│                               │    │file system                          │from `package.json`      │
│                               │    │                                     │`remappings.txt` if      │
│                               │    │                                     │exist, conflicting       │
│                               │    │                                     │mappings cause the script│
│                               │    │                                     │abnormal termination     │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│packages_path                  │S   │Look for Solidity packages in the    │Looks for the packages in│
│                               │    │given directory                      │$NODE_PATH               │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│parametric_contracts           │L   │Filter the set of contracts whose    │Verifies all functions in│
│                               │    │functions will be verified in        │all contracts in the file│
│                               │    │parametric rules/invariants          │list                     │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│precise_bitwise_ops            │B   │Show precise bitwise operation       │May report               │
│                               │    │counter examples. Models mathints as │counterexamples caused by│
│                               │    │unit256 that may over/underflow      │incorrect modeling of    │
│                               │    │                                     │bitwise operations, but  │
│                               │    │                                     │supports unbounded       │
│                               │    │                                     │integers (mathints)      │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│project_sanity                 │B   │Perform basic sanity checks on all   │                         │
│                               │    │contracts in the current project     │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│protocol_author                │S   │Add the protocol's author for easy   │The `package.json` file's│
│                               │    │filtering in the dashboard           │`author` field if found  │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│protocol_name                  │S   │Add the protocol's name for easy     │The `package.json` file's│
│                               │    │filtering in the dashboard           │`name` field if found    │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│prototype                      │L   │Set the address of the contract's    │Calls to the created     │
│                               │    │create code.                         │contract will be         │
│                               │    │                                     │unresolved, causing      │
│                               │    │Format:                              │havocs that may lead to  │
│                               │    │  <hex address>=<Contract>           │non-useful counter       │
│                               │    │                                     │examples                 │
│                               │    │Example:                             │                         │
│                               │    │  0x3d602...73                       │                         │
│                               │    │                                     │                         │
│                               │    │Contract Foo will be created from the│                         │
│                               │    │code in address 0x3d602...73         │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│prover_args                    │L   │Send flags directly to the Prover    │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│prover_version                 │S   │Use a specific Prover revision       │Uses the latest public   │
│                               │    │                                     │Prover version           │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│rule                           │L   │Verify only the given list of        │Verifies all rules and   │
│                               │    │rules/invariants. Asterisks are      │invariants               │
│                               │    │interpreted as wildcards             │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│rule_sanity                    │S   │Select the type of sanity check that │No sanity checks         │
│                               │    │will be performed during execution   │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│short_output                   │B   │Reduce verbosity                     │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc                           │S   │Path to the Solidity compiler        │Calling `solc`           │
│                               │    │executable file                      │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_allow_path                │S   │Set the base path for loading        │Only the Solidity        │
│                               │    │Solidity files                       │compiler's default paths │
│                               │    │                                     │are allowed              │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_evm_version               │S   │Instruct the Solidity compiler to use│Uses the Solidity        │
│                               │    │a specific EVM version               │compiler's default EVM   │
│                               │    │                                     │version                  │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_evm_version_map           │M   │Map contracts to the appropriate EVM │Uses the same Solidity   │
│                               │    │version                              │EVM version for all      │
│                               │    │                                     │contracts                │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_map                       │M   │Map contracts to the appropriate     │Uses the same Solidity   │
│                               │    │Solidity compiler in case not all    │compiler version for all │
│                               │    │contract files are compiled with the │contracts                │
│                               │    │same Solidity compiler version.      │                         │
│                               │    │                                     │                         │
│                               │    │CLI Example:                         │                         │
│                               │    │  --solc_map                         │                         │
│                               │    │A=solc8.11,B=solc8.9,C=solc7.5       │                         │
│                               │    │                                     │                         │
│                               │    │JSON Example:                        │                         │
│                               │    │  "solc_map: {"                      │                         │
│                               │    │    "A": "solc8.11",                 │                         │
│                               │    │    "B": "solc8.9",                  │                         │
│                               │    │    "C": "solc7.5"                   │                         │
│                               │    │  }                                  │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_optimize                  │S   │Tell the Solidity compiler to        │Uses the Solidity        │
│                               │    │optimize the gas costs of the        │compiler's default       │
│                               │    │contract for a given number of runs  │optimization settings    │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_optimize_map              │M   │Map contracts to their optimized     │Compiles all contracts   │
│                               │    │number of runs in case not all       │with the same            │
│                               │    │contract files are compiled with the │optimization settings    │
│                               │    │same number of runs.                 │                         │
│                               │    │                                     │                         │
│                               │    │CLI Example:                         │                         │
│                               │    │  --solc_optimize_map                │                         │
│                               │    │A=200,B=300,C=200                    │                         │
│                               │    │                                     │                         │
│                               │    │JSON Example:                        │                         │
│                               │    │  "solc_optimize_map": {             │                         │
│                               │    │    "A": "200",                      │                         │
│                               │    │    "B": "300",                      │                         │
│                               │    │    "C": "200"                       │                         │
│                               │    │  }                                  │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_via_ir                    │B   │Pass the `--via-ir` flag to the      │                         │
│                               │    │Solidity compiler                    │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│solc_via_ir_map                │M   │Map contracts to the appropriate     │do not set via_ir during │
│                               │    │via_ir value                         │compilation unless       │
│                               │    │                                     │solc_via_ir is set       │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│split_rules                    │L   │List of rules to be sent to Prover   │Verifies all rules and   │
│                               │    │each on a separate run               │invariants in a single   │
│                               │    │                                     │run                      │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│struct_link                    │L   │Link a slot in a struct with another │The slot can be any      │
│                               │    │contract.                            │address, often resulting │
│                               │    │                                     │in unresolved calls and  │
│                               │    │Format:                              │havocs that lead to      │
│                               │    │  <Contract>:<slot#>=<Contract>      │non-useful counter       │
│                               │    │Example:                             │examples                 │
│                               │    │  Bank:0=BankToken Bank:1=LoanToken  │                         │
│                               │    │                                     │                         │
│                               │    │The first field in contract Bank is a│                         │
│                               │    │contract of type BankToken and the   │                         │
│                               │    │second of type LoanToken             │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│summary_recursion_limit        │S   │Determine the number of recursive    │0 - no recursion is      │
│                               │    │calls we verify in case of recursion │allowed                  │
│                               │    │of Solidity functions within a       │                         │
│                               │    │summary                              │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│verify                         │S   │Path to The Certora CVL formal       │                         │
│                               │    │specifications file.                 │                         │
│                               │    │                                     │                         │
│                               │    │Format:                              │                         │
│                               │    │  <contract>:<spec file>             │                         │
│                               │    │Example:                             │                         │
│                               │    │  Bank:specs/Bank.spec               │                         │
│                               │    │                                     │                         │
│                               │    │spec files suffix must be .spec      │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│version                        │B   │Show the Prover version              │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│vyper                          │S   │Path to the Vyper compiler executable│Calling `vyper`          │
│                               │    │file                                 │                         │
├───────────────────────────────┼────┼─────────────────────────────────────┼─────────────────────────┤
│wait_for_results               │S   │Wait for verification results before │Sends request and does   │
│                               │    │terminating the run                  │not wait for results     │
└───────────────────────────────┴────┴─────────────────────────────────────┴─────────────────────────┘


You can find detailed documentation of the supported flags here: 
https://docs.certora.com/en/latest/docs/prover/cli/options.html


