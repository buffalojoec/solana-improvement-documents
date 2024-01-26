---
simd: 'XXXX'
title: Feature Gate Threshold Automation
authors:
  - Tyera Eulburg
  - Joe Caulfield
category: Standard
type: Core
status: Draft
created: 2024-01-25
feature: (fill in with feature tracking issues once accepted)
---

## Summary

This SIMD outlines a proposal for automating the feature activation process
based on a stake-weighted support threshold, rather than manual human action.

With this new process, contributors would no longer have to assess stake support
for a feature before activation. Instead, this would be done by the runtime.

## Motivation

Feature gates wrap new cluster functionality, and typically change the rules of
consensus. As such, a feature gate needs to be supported by a strong majority
of cluster stake when it is activated, or else it risks partitioning the
network. The current feature-gate system comprises two steps:

1. An individual key-holder queues a feature gate for activation
2. The runtime automatically activates the feature on the next epoch boundary

The key-holder is the one who *manually* (with the help of the solana-cli)
assesses the amount of stake that recognizes the feature and decides whether
it is safe to activate. This is obviously brittle and subject to human error.

If instead the runtime was to assess this stake support for activating a
feature, this would eliminate the key-holder's responsibility to asses stake
support, reducing the risk of human error.

## New Terminology

- **Feature Gate program:** The Core BPF program introduced in
  [SIMD 0089](https://github.com/solana-foundation/solana-improvement-documents/pull/89)
  that will own all feature accounts.
- **Staged Features PDA:** The PDA under the Feature Gate program used to track
  features submitted for activation for a particular epoch.
- **Support Signal PDA:** The PDA under the Feature Gate program used to store
  a bit mask of the staged features a node supports.
- **Feature Tombstone PDA:** The PDA under the Feature Gate program used to
  assign accounts to, effectively "archiving" them and removing them from the
  Feature Gate program's owned accounts.

## Detailed Design

The proposed new process would be comprised of the following steps:

1. **Authorized Feature Submission:** In some epoch 0, the multi-signature
   authority submits features for activation.
2. **Signaling Support for Staged Features:** During the next epoch (epoch 1),
   validators signal which of the staged feature-gates they support in their
   software.
3. **Activation and Garbage Collection:** On the next epoch boundary, the
   runtime activates the feature-gates that have the necessary stake support.
   At the same time, the runtime also removes spam and archives activated
   feature accounts.

### Step 1: Authorized Feature Submission

A multi-signature authority shall be created to submit features for activation.
This multi-signature will comprise key-holders from Solana Labs and possibly
from other validator client teams in the future. In the future, this authority
could be replaced by validator governance.

The transaction to submit a feature activation shall contain the necessary
System Program instructions to fund, allocate, and assign the feature account
to `Feature111111111111111111111111111111111111`, as well as the Feature Gate
program's instruction `SubmitFeatureForActivation`.

The `SubmitFeatureForActivation` instruction verifies the signature of the
multi-signature authority, then adds the submitted feature ID to the **next
epoch's** Staged Features PDA.

The Staged Features PDA for a given epoch stores a list of all feature IDs that
were submitted prior to that epoch. To start, this list shall have a maximum
length of 5.

### Step 2: Signaling Support for Staged Features

With an on-chain reference point to determine the features staged for activation
for a particular epoch, nodes can signal their support for the staged features
supported by their software.

A node signals its support for staged features by invoking the Feature Gate
program's `SignalSupportForStagedFeatures` instruction, which requires a  
`[u8; 5]` bit mask of the staged features. Each element of the bit mask is
either a `1` - representing a supported feature - or a `0`. The program stores
this provided bit mask in a Support Signal PDA derived from the node's vote
address.

An example of this instruction is defined below.

```rust
pub enum FeatureGateInstruction {
    /// Signal support for staged features.
    ///
    /// This instruction submits a bit mask representing the current epoch's
    /// staged features.
    ///
    /// Validators submit these bit masks to describe feature-gates supported
    /// by their current software version.
    ///
    /// A `1` value represents support for the feature at that index of the
    /// bit mask, while a `0` represents a lack of support (or rejection).
    ///
    /// Accounts expected by this instruction:
    ///
    ///   0. `[w]`      Support Signal PDA
    ///   1. `[s]`      Vote account
    SignalSupportForFeatureSet {
        bit_mask: [u8; 5],
    },
}
```

Nodes shall send this instruction at some arbitrary point during the epoch at
least 128 slots before the end of the epoch and on startup after any reboot.

Note: If a feature is revoked, the list of staged features will not change, and
nodes may still signal support for this feature. However, the runtime will not
activate this feature if its corresponding feature account no longer exists
on-chain.

### Step 3: Activation and Garbage Collection

During the epoch rollover, the runtime uses the validator support signals to
determine which staged features to activate.

To do this, the runtime walks all of the vote accounts, derives their Support
Signal PDA to read their bit mask, and tallies up the total stake support for
each staged feature. The runtime will also zero-out each bit mask, resetting
each Support Signal PDA for the next epoch.

Only features whose stake support meets the required threshold are activated.
This threshold shall be set to 95% initially, but future iterations on the
process could allow feature key-holders to set a custom threshold per-feature.

If a feature is not activated, either because it has been revoked or it did not
meet the required stake support, it must be resubmitted according to Step 1.

To ensure this new process doesn't overload the Feature Gate program's owned
accounts, during the activation stage, garbage collection will:

- Archive any activated feature accounts
- Archive this epoch's Staged Features PDA
- Delete any spam feature accounts with no state

The runtime "archives" an account by assigning it to the Feature Tombstone PDA.

### Conclusion

This new process provides additional safeguards for submitting features for
activation as well as an automated stake support threshold check before
activating them. This greatly reduces the risk of human error when enabling new
functionality on the network, as well as minimizes the risk of cluster
partitioning.

Additionally, this system makes use of on-chain data and a client-agnostic BPF
program to perfom its duties, further decentralizing the feature activation
process.

These enhancements will be especially useful in a world where multiple clients
will aim to push features and seek to agree on their activation.

## Alternatives Considered

## Impact

This new process for activating features directly impacts core contributors and
validators.

Core contributors will no longer bear the responsibility of ensuring the proper
stake supports their feature activation. However, it will change the process
by which they can override this stake requirement to push a feature through.

Validators will be responsible for signaling their vote using a transaction
which they've previously not included in their process. They also will have a
more significant impact on feature activations if they neglect to upgrade their
software version.

## Security Considerations

This proposal increases security for feature activations by removing the human
element from ensuring the proper stake supports a feature.

This proposal could also potentially extend the length of time required for
integrating feature-gated changes, which may include security fixes. However,
the feature-gate process is relatively slow in its current state, and neither
the current process or this proposed process would have any implications for
critical, time-sensitive issues.

