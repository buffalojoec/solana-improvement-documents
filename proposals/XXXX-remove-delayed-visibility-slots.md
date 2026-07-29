---
simd: 'XXXX'
title: Remove Delayed Visibility Slots
authors:
    - Joe Caulfield (Anza)
category: Standard
type: Core
status: Draft
created: 2026-07-29
feature: (fill in with feature key and github tracking issues once accepted)
---

## Summary

A program that is modified (deployed, upgraded, or extended) in a given slot is
rendered unavailable for the remainder of the slot. This SIMD proposes removing
this restriction and instead only disabling the program for the remainder of
the transaction in which it was modified.

## Motivation

Rendering a program unavailable for an entire slot is unnecessarily
conservative. A program can only change underneath a caller during the
transaction which modifies it. Every transaction after that one observes the
fresh program bytes in its account data.

The cost of the current rule grows with how heavily a program is used. Every
transaction which invokes the program for the remainder of the slot fails with
`UnsupportedProgramId`. Those transactions are not deferred. They are failed,
and each one has to be rebuilt and resubmitted by whoever sent it. A routine
upgrade of a popular program therefore fails user transactions en masse.

The migration from SPL Token to P-Token is one such example. The token program
is invoked by a large share of the transactions in any given block, and
modifying it rendered it unavailable for the remainder of the slot the
migration landed in.

Restricting the program to the transaction which modifies it keeps the property
that matters and confines the disruption of an upgrade to only that transaction.

## New Terminology

No new terminology is introduced by this proposal.

## Detailed Design

The following Loader V3 instructions currently modify a program:

- `DeployWithMaxDataLen`
- `Upgrade`
- `ExtendProgram`

A Core BPF migration, as described in [SIMD-0088], also modifies a program. It
is performed by the runtime at the start of the slot in which its feature
activates, rather than by a transaction.

[SIMD-0088]: https://github.com/solana-foundation/solana-improvement-documents/blob/main/proposals/0088-enable-core-bpf-programs.md

After this proposal's feature gate is activated, a program modified in slot `N`
MUST be invocable in slot `N`, rather than becoming invocable in slot `N + 1`.

A program modified by the currently executing transaction MUST NOT be invocable
for the remainder of that transaction. Invoking it MUST fail with
`UnsupportedProgramId`, as it does today. This applies both to top level
instructions and to instructions invoked via CPI.

A program becomes invocable again at the start of the next transaction which
follows the one that modified it.

Rolling back a modification is unchanged. A transaction which modifies a
program and then fails has that modification discarded, and the program
continues to run the code it had before the transaction.

Closing a program is unchanged. It takes effect in the slot it occurs in, and
the closed program is not invocable for the remainder of the transaction which
closed it.

### Epoch Boundaries

A program is verified when it is modified, against the rules of the epoch in
which it will first execute. The first slot a program can execute in is
currently the slot after the one it was modified in, so a program modified in
slot `N` is currently verified against the rules of the epoch containing slot
`N + 1`.

After activation a program modified in slot `N` first executes in slot `N`, and
so it MUST be verified against the rules of the epoch containing slot `N`.

This is only observable in the last slot of an epoch, and only when the two
epochs differ in a feature which changes how a program is verified, such as the
set of registered syscalls or the permitted SBPF versions.

### Edge Cases

**Modification and invocation in one transaction.** A transaction which
modifies a program and then invokes it fails, as it does today.

**Programs created in the transaction which deploys them.** A program account
created by the same transaction which deploys it is not loadable by a top level
instruction of that transaction, because it is not yet a program account when
the transaction's accounts are loaded. That is independent of this proposal and
is unchanged by it.

**Two versions in one transaction.** A transaction may invoke a program before
modifying it, and that invocation observes the version which preceded the
transaction. After the modification the program is not invocable. No
transaction executes two versions of the same program.

**At most one modification per slot.** `DeployWithMaxDataLen` requires an
uninitialized program account, and `Upgrade`, `ExtendProgram`, and `Close`
reject the instruction when the program's recorded deployment slot equals the
current slot. A program can therefore be modified at most once in a slot, so at
most one transaction per slot is subject to the restriction above. This is
unchanged by this proposal.

**Core BPF migrations.** A migration is not performed by a transaction, so
there is no transaction to restrict. The migrated program MUST be invocable by
every transaction in the slot the migration occurred in, where today it is
invocable by none of them.

### Feature Activation

Removing the delay and scoping the restriction to the transaction MUST activate
under a single feature gate.

## Alternatives Considered

### Make the program invocable for the remainder of the transaction

Remove the restriction entirely and let the transaction which modified a
program invoke the new version immediately. This is the simplest rule to state
and it permits deploying a program and using it in one transaction.

A transaction could then execute two versions of the same program, and a
program could replace its own code and continue executing under the
replacement.

### Restrict the program for the remainder of the instruction only

Restrict the program for the rest of the instruction which modified it, rather
than the rest of the transaction. A later top level instruction in the same
transaction could then invoke the new version. This permits deploying a program
and using it in one transaction, while still preventing a program from
replacing its own code while executing.

A transaction could then execute two versions of the same program.

### Retain the current behavior

Every deploy, upgrade, and extend continues to cost a slot of unavailability
for the program, and the transactions which invoke it during that slot continue
to fail.

## Impact

Programs are usable in the slot they are modified in. An upgrade of a popular
program no longer fails the transactions which invoke it for the remainder of
that slot, and deployment tooling no longer has to wait a slot for a program to
respond.

Transactions which fail with `UnsupportedProgramId` today because they invoke a
program in the slot it was modified in will succeed after activation, provided
they do not invoke it in the transaction which modified it.

No instruction gains or loses an account, and no account's signer or writable
requirements change. Tooling is otherwise unaffected.

## Security Considerations

An upgrade authority can replace program code and have the replacement take
effect in the same slot. The window between an upgrade landing and the new code
executing narrows from one slot to zero. Anything which relies on that slot as
a margin to observe an upgrade before it takes effect loses it.

No transaction executes two versions of the same program, and no program
executes under code which was replaced while it was running. Both properties
hold today by virtue of the delay, and are preserved by restricting the program
for the remainder of the transaction which modified it.

No authority or signer check is relaxed. Which accounts may modify a program is
unchanged, as is the set of instructions which can do so.

## Backwards Compatibility

This change is consensus-relevant and therefore requires a feature gate.
Transactions rejected before activation may succeed after it.

The change is backwards compatible for CLI and tooling.
