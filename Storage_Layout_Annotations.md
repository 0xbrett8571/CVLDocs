# Storage Layout Annotations — Certora Prover Documentation 0.0 documentation

# Storage Layout Annotations[](#storage-layout-annotations "Link to this heading")

## [Solidity and EIP-7201 namespace](https://eips.ethereum.org/EIPS/eip-7201)[](#solidity-and-eip-7201-namespace "Link to this heading")

Upgradeable contracts often need to add new state variables without interfering with existing storage layouts. To achieve this, they access storage by manually computing storage locations using hashing. EIP-7201 was proposed to standardize both the formula for deriving these storage locations and the documentation format for annotating them, making the process more reliable and interoperable.

 /\*\* @custom:storage-location erc7201:example.main \*/
 struct MainStorage {
     uint256 x;
     uint256 y;
 }

When storage is accessed using custom slot calculations—such as:

// keccak256(abi.encode(uint256(keccak256("example.main")) - 1)) & ~bytes32(uint256(0xff));
bytes32 private constant MAIN\_STORAGE\_LOCATION \=
  0x183a6125c38840424c4a85fa12bab2ab606c4b6d0e7cc73c0c06ba5300eab500;
function \_getMainStorage() private pure returns (MainStorage storage \_slot) {
  assembly {
    \_slot.slot := MAIN\_STORAGE\_LOCATION
  }
}

—the Certora Prover may not be able to automatically infer the correct storage layout, which can lead to analysis failures. By following the EIP-7201 proposal and using the automatic storage extension generation option, the Prover can accurately generate layout information for these annotated storage extensions. This improves reliability and reduces the risk of incorrect analysis when contracts use custom storage slots.

Not all contracts follow the EIP-7201 proposal. In such cases, the Certora Prover allows the user to explicitly specify storage layout information by writing an _extension harness_.

## Table of Contents[](#table-of-contents "Link to this heading")

*   [Storage extension flags](#storage-extension-flags)
    
*   [How it works (high level)](#storage-extension-how)
    
*   [User-provided harnesses](#storage-extension-harnesses)
    
*   [Quick example](#storage-extension-example)
    
*   [Troubleshooting](#storage-extension-troubleshooting)
    

* * *

## Storage Extension Flags[](#storage-extension-flags "Link to this heading")

To enable automatic storage extension, add one or both of the following flags to your `.conf` JSON:

Flag

Appearance in `.conf` file

Pass directly to CLI option

Purpose

storage\_extension\_annotation

`"storage_extension_annotation": true`

`--storage_extension_annotation`

Detects `@custom:storage-location erc7201:…` annotations and **automatically extends the storage layout** during compilation.

extract\_storage\_extension\_annotation

`"extract_storage_extension_annotation": true`

`--extract_storage_extension_annotation`

Dumps the generated harness Solidity file(s) to `<build_dir>/…` for inspection.

storage\_extension\_harnesses

`"storage_extension_harnesses": ["A=H", ...]`

\`–storage\_extension\_harnesses A=H …

Specifies that each contract `A`’s storage layout is extended by the layout specified in `H`

**Example:**

{
  // ...
    "storage\_extension\_annotation": true,
    "extract\_storage\_extension\_annotation": true,
    "storage\_extension\_harnesses" : \[ "A1=H1", "A2=H2" \]
  // ...
}

## How automatic generation works (high level)[](#how-automatic-generation-works-high-level "Link to this heading")

1.  **Scan ASTs** – while compiling each `.sol` file, the Prover looks for struct definitions with a preceding comment `@custom:storage-location erc7201:<namespace>`.
    
2.  **Generate a minimal harness** – for every unique namespace the tool emits an _extra_ contract containing **only**:
    
    *   Import statements for the original file,
        
    *   One dummy state variable per namespace with the prefix `ext_<namespace>_` and the original struct name.
        
    
     // auto-generated
    import "./VaultBridgeToken.sol";
    contract \_Auto\_VaultBridgeTokenHarness\_ {
      VaultBridgeTokenStorage ext\_agglayer\_vault\_bridge\_VaultBridgeToken\_storage;  // slot = keccak256("erc7201:agglayer.vault-bridge.VaultBridgeToken.storage")-1 & ~0xff
    }
    
3.  **Compile the harness** with _exactly_ the same `solc` flags that the main file is using.
    
4.  **Splice the fields** – the storage layout extracted from the harness is merged into the layout of every contract that _inherits_ the annotated struct’s declaring contract.
    
5.  **(optional) Dump the harness** into `.<build_dir>/` if `extract_storage_extension_annotation` is on.
    

## User-provided harnesses[](#user-provided-harnesses "Link to this heading")

Not all contracts follow EIP-7201:

contract A {
  struct MainStorage {
       uint256 x;
       uint256 y;
   }
  
  bytes32 private constant MAIN\_NONSTANDARD\_LOCATION \= 0x12345678;
  
  function \_getMainStorage() private pure returns (MainStorage storage \_slot) {
    assembly {
      \_slot.slot := MAIN\_NONSTANDARD\_LOCATION
    }
  }
}

Here there is neither an annotation on the `MainStorage` struct, nor is the storage slot calculated according to the formula specified in the EIP. In this case, the user can manually write a storage extension harness:

import {A} from "/path/to/A.sol";

contract H {
  /\*\* @custom:certoralink 0x12345678 \*/
  A.MainStorage m;
}

and provide the CLI option `--storage_extension_harnesses A=H` (or add `"storage_extension_harnesses": ["A=H"]` to the `.conf` file)

The extension harness may have several such state variables. Each variable must have a `@custom:certoralink` NatSpec annotation followed by a hexadecimal string. Each such

/\*\* @custom:certoralink s\_i \*/
T\_i x\_i;

annotation will instruct the Prover that storage slot `s_i` has the layout corresponding to `T_i`.

## Quick example[](#quick-example "Link to this heading")

contract VaultBridgeToken {
    /\*\* @custom:storage-location erc7201:agglayer.vault-bridge.VaultBridgeToken.storage \*/
    struct VaultBridgeTokenStorage {
        IERC20 underlyingToken;
        uint256 reservedAssets;
        // ...
    }
}

With

{
  // ...
  "storage\_extension\_annotation": true
  // ...
}

you can immediately assert over `currentContract.ext_agglayer_vault_bridge_VaultBridgeToken_storage.underlyingToken` in CVL:

rule no\_reserved\_assets\_leak(env e) {
    require currentContract.\_reservedAssets() \== 0;
    deposit(e, 100);
    assert currentContract.ext\_agglayer\_vault\_bridge\_VaultBridgeToken\_storage.reservedAssets \>= 100;
}

## Troubleshooting[](#troubleshooting "Link to this heading")

*   **“Slot already declared”** Two different annotations resolved to the same slot. Double-check the namespace strings and inheritance chain.
