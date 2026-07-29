---
simd: 'XXXX'
title: 'Loader V3: Relax CPI Constraints'
authors:
    - Joe Caulfield (Anza)
category: Standard
type: Core
status: Draft
created: 2026-07-29
feature: (fill in with feature key and github tracking issues once accepted)
supersedes: (optional - fill this in if the SIMD supersedes a previous SIMD)
superseded-by: (optional - fill this in if the SIMD is superseded by a subsequent
 SIMD)
extends: (optional - fill this in if the SIMD extends the design of a previous
 SIMD)
---

## Summary

Loader V3 currently blocks four of its eight instructions from being invoked
via CPI. This SIMD proposes lifting that blanket restriction and replacing it
with a narrower rule enforced by the loader itself: these instructions fail if
the account they operate on is the program that invoked them.

## Motivation

The CPI restriction is not enforced by the loader. It lives in the runtime's
CPI syscall, which holds an allowlist of Loader V3 instructions keyed on the
first byte of the instruction data.

The following instructions are currently allowed via CPI:

- `Upgrade`
- `SetAuthority`
- `SetAuthorityChecked`
- `Close`

The following instructions are currently rejected via CPI:

- `InitializeBuffer`
- `Write`
- `DeployWithMaxDataLen`
- `ExtendProgram`

This is far coarser than the hazard it guards against: changing the size of
the data region backing an ELF that may be loaded and verified during the
same execution, which can change the result of a verifier run. `Upgrade` is
exempt because it cannot resize — it rewrites a region fixed at deployment.

The hazard can therefore be mitigated by focusing on the account an
instruction operates on, rather than the instruction itself, since it arises
only when that account backs a currently executing program.

The instructions blocked today are precisely the ones a program would need to
manage other programs on-chain:

- Program factories that deploy programs from a PDA authority
- Programs that assemble buffers on a user's behalf
- PDA authorities that extend the programs they administer

## New Terminology

No new terminology is introduced by this proposal.

## Detailed Design

After this proposal's feature gate is activated, the CPI syscall MUST NOT
filter Loader V3 instructions. All eight instructions become invocable via
CPI.

In the Loader V3 program, the four instructions that were previously
disallowed in CPI will instead enforce a check on the caller program ID and
the target account the instruction operates on.

Each instruction with its associated "operated on" target account is listed
below:

| Instruction            | Target account         | Index |
|------------------------|------------------------|-------|
| `InitializeBuffer`     | Buffer                 | 0     |
| `Write`                | Buffer                 | 0     |
| `DeployWithMaxDataLen` | Program account        | 2     |
| `ExtendProgram`        | Program account        | 1     |

When one of these instructions is invoked via CPI, it MUST fail with
`InvalidArgument` if the target account's address equals the address of the
calling program.

The calling program is the program executing one level below the current
instruction on the instruction stack. A top-level instruction has no caller
and is therefore never subject to this check.

The check MUST be performed immediately after the instruction's account count
check and before any account is borrowed or deserialized.

`Upgrade`, `SetAuthority`, `SetAuthorityChecked`, and `Close` are unchanged.
They remain invocable via CPI and MAY target the calling program.

### Edge Cases

**Unknown or malformed instruction data.** The allowlist matches the first
byte of the instruction data, so instruction data whose first byte is not one
of the four permitted discriminants — including empty data — is rejected at
the CPI boundary today. After activation such data reaches the loader, which
rejects it during deserialization instead. The resulting error changes.

**Indirect self-reference.** A program invoked by the caller may target the
caller. Program `A` invokes `B`, and `B` invokes `ExtendProgram` against `A`.
`A` is executing, but it is not the calling program, so the check does not
apply and the instruction proceeds. See Alternatives Considered.

### Feature Activation

Both halves of this change — removing the allowlist entry from CPI and adding
the self-reference check in the Loader V3 program — MUST activate under a
single feature gate.

## Alternatives Considered

### Lift the restriction entirely, as with `Upgrade`

Remove the allowlist and add no check at all, leaving the four instructions to
the loader's existing authority and signer constraints, as `Upgrade` is today.

`Upgrade` cannot resize the region it writes to, so it cannot change a
verifier result mid-execution. `ExtendProgram` can, so the exemption does not
extend to it. Dropping the check later is a relaxation; adding it later is a
breaking change.

### Reject any program on the instruction stack

Walk the instruction stack and reject the instruction if the target matches
any program currently executing, not just the caller. This also covers the
indirect case described above.

This is more thorough, at the cost of a walk bounded by the maximum invocation
depth instead of a single lookup. It is also potentially more secure against
verifier edge cases, since any program on the stack may have its region loaded
and verified during the same execution. It remains open for consideration.

### Retain existing constraints and make no changes

On-chain program management remains impossible.

## Impact

Programs can deploy, write, and extend other programs via CPI. This enables
on-chain program factories, buffer management on behalf of users, and PDA
authorities that administer the programs they own.

No existing on-chain program is affected. The four instructions cannot
currently be invoked via CPI at all, so no deployed code depends on their
behavior in that context. Top-level invocations are unchanged.

Tooling is unaffected. No instruction gains or loses an account, and no
account's signer or writable requirements change.

## Security Considerations

The restriction changes from a total block in CPI to a block on the calling
program only. A program executing more than one frame below the loader is no
longer protected. If program `A` invokes `B`, then `B` may extend `A` while
`A` is still executing, since the check only compares against `B`. Rejecting
any program on the instruction stack, described in Alternatives Considered,
would close this.

Only `ExtendProgram` can act on the calling program. `DeployWithMaxDataLen`
requires an `Uninitialized` program account, and `InitializeBuffer` and `Write`
require a `Buffer`, so all three would fail their own state checks first. The
check covers all four so the rule holds for each on its own.

No authority or signer check is relaxed. These apply the same at the top level
and via CPI.

`ExtendProgram` remains permissionless, so a program may extend any other
program via CPI, invalidating its cache entry for the slot. This is already
possible at the top level, and SIMD-0431 governs its cost.

## Backwards Compatibility

This change relaxes a consensus-relevant restriction on Loader V3 and
therefore requires a feature gate. Transactions rejected before activation may
succeed after it.

The change is backwards compatible for CLI and tooling.
