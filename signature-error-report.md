# Signature Error Handling Bug - Completion Report

**Date:** 2026-02-04
**Branch:** `wesl-imports`
**Related:** signature-error-1.md

## Completed

Fixed the signature error handling bug per signature-error-1.md.

### Changes

1. Added `source: ExpressionStoreSource` to three `InferenceDiagnostic` variants:
   - `UnexpectedTemplateArgument`
   - `WgslError`
   - `ExpectedLoweredKind`

2. Updated `any_diag_from_infer_diagnostic` to select correct source map based on the `source` field

3. Added tests at hir_ty and IDE levels (using expect_test for exact message verification)

### Design Decisions

- **Per-variant source field** (not wrapper struct) - matches existing `InvalidType` pattern
- **Only fixed definitely-affected variants** - investigation confirmed "possibly affected" variants are body-only

### Commits

```
00de728 Fix signature diagnostics being silently dropped
5f97196 Add IDE-level tests for signature error handling
206021f Use expect_test for IDE diagnostic tests
```

## Investigation of "Possibly Affected" Diagnostics

All six diagnostics listed as "possibly" occurring in signatures were investigated and found to be **body-only**:

| Diagnostic | Finding |
|------------|---------|
| TypeMismatch | Body-only (assignments, function calls, constructors) |
| UnresolvedName | **Dead code** - defined but never emitted |
| InvalidConstructionType | Body-only (type constructors) |
| FunctionCallArgCountMismatch | Body-only (function validation, constructors) |
| NoBuiltinOverload | Body-only (unary/binary operators) |
| NoConstructor | Body-only (type constructors) |

### Why These Don't Need `source`

Signature processing uses `TypeLoweringContext` which:
1. Only emits `TypeLoweringError` → converted to `InvalidType` (already had `source` field)
2. For template argument evaluation (`evaluate_template_argument`), complex expressions like function calls return `None` rather than going through full expression inference

The three diagnostics we fixed are the **only ones** that can occur in signature contexts.

## Should `source` Be Added to All Variants?

**Recommendation: No**, for architectural reasons.

The codebase has a deliberate separation:
- **Signature processing** → `TypeLoweringContext` → emits `TypeLoweringError` → `InvalidType`
- **Body inference** → `infer_expression` and friends → emits the other diagnostics

The "possibly affected" diagnostics (`TypeMismatch`, `NoBuiltinOverload`, etc.) are emitted from functions like `infer_statement`, `validate_function_call`, `call_builtin` - these are called from body inference, not signature processing.

For a diagnostic like `TypeMismatch` to occur in a signature context, you'd need to route signature expressions through `infer_expression`. That would be a significant architectural change, and at that point adding `source` would be a small part of the work.

If WGSL/WESL evolves to support more complex const-expressions in signatures (like const-fn calls that need full type inference), the architecture might need to change. But that's speculative and would require deliberate code changes - at which point the developer would need to handle the `source` field anyway.

**If desired**, a comment could be added near the `InferenceDiagnostic` enum to document this assumption:

```rust
// NOTE: Variants with `source: ExpressionStoreSource` can occur in both
// body and signature contexts. Other variants only occur in body contexts
// because signature processing uses TypeLoweringContext, not full inference.
// If this architectural assumption changes, add `source` to affected variants.
```

## Housekeeping Note

`UnresolvedName` variant in `InferenceDiagnostic` is dead code - it's defined but never emitted anywhere. Consider removing it in a future cleanup.

## Files Modified

- `crates/hir_ty/src/infer.rs` - Added `source` field to variants, updated all emission sites
- `crates/hir/src/diagnostics.rs` - Updated conversion to use correct source map
- `crates/hir_ty/src/tests/simple.rs` - Updated hir_ty level test
- `crates/ide/src/diagnostics.rs` - Added IDE-level test module
