---
simd: '0140'
title: Migrate Config to Core BPF
authors:
  - Joe Caulfield - Anza Technology
category: Standard
type: Core
status: Draft
created: 2024-04-02
feature: (fill in with feature tracking issues once accepted)
---

## Summary

Migrate the Config program to Core BPF.

## Motivation

BPF programs offer less complexity than native programs for other clients, such
as Firedancer, since developers will no longer have to keep up with program
changes in their runtime implementations. Instead, the program can just be
updated once.

In this spirit, Config should be migrated to Core BPF.

## Alternatives Considered

Config could instead remain a builtin program. This would mean each validator
client implementation would have to build and maintain this program alongside
their runtime, including and future changes.

## New Terminology

N/A.

## Detailed Design

The program will be reimplemented in order to be compiled to BPF and executed by
the BPF loader.

The reimplemented program's ABI will exactly match that of the original.

The reimplemented program's functionality will exactly match that of the
original, differing only in compute usage.

The program will be migrated to Core BPF using the procedure outlined in
[SIMD 0088](https://github.com/solana-foundation/solana-improvement-documents/pull/88).

The program's upgrade authority will be a multi-sig authority with keyholders
from Anza Technology and may expand to include contributors from other validator
client teams.
In the future, this authority could be replaced by validator governance.

## Impact

Validator client teams are no longer required to implement and maintain a Config
program within their runtime.

All validator client teams can work to maintain the single Config program
together.

## Security Considerations

The program will be upgradeable. The upgrade authority will be a multi-sig held
by core contributors.

The program's reimplementation poses no new security considerations compared to
the original builtin version.

## Backwards Compatibility

The Core BPF implementation is 100% backwards compatible with the original
builtin implementation.
