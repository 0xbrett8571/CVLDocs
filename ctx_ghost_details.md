# Tracking changes with ghosts and hooks — Certora Prover Documentation 0.0 documentation
Sometimes it is useful for a rule to observe changes to a particular part of storage, or otherwise track the behavior of a contract while it is executing. For example, while verifying an ERC20 contract, we may want to observe each update to any user balance so that we can keep our own running tally of the total supply.

Ghosts and hooks are designed for this purpose. Ghosts are additional variables that you can add to use during verification. They are similar to contract storage variables — they are rolled back when a contract function reverts or when the storage is reset.

Hooks are blocks of CVL code that get executed when a contract performs a certain instruction. For example, a store hook will be run whenever the contract updates a storage variable, while a call hook will be run whenever the contract makes an external call.

Together, hooks and ghosts let you keep track of what the contract does during execution.

