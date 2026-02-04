# Signature Error Handling Bug

**Date:** 2026-01-22
**Branch:** `wesl-imports`
**Related:** Issue #632, PR #700

## Summary

Diagnostics originating from signature contexts (function parameters, return types, global variable templates) are silently dropped during conversion from `InferenceDiagnostic` to `AnyDiagnostic`. Only `InvalidType` correctly handles both signature and body contexts.

## Background

The branch author (stefnotch) noted:
> hir-ty, the type checker, has some error handling, however it incorrectly assumes that all errors happen in the body. But thanks to wgsl allowing arbitrary expressions in types, errors can also happen in signatures. So that bit of info needs to be passed along with every error.

## Architecture

WGSL-analyzer uses two separate `ExpressionStore` instances per definition:

```
┌─────────────────────────────────────────────────────────┐
│  Definition (e.g., function, global variable)           │
├─────────────────────────────────────────────────────────┤
│  Signature Store (ExpressionStoreSource::Signature)     │
│  - Parameter types                                      │
│  - Return type                                          │
│  - Template arguments (e.g., var<storage>)              │
├─────────────────────────────────────────────────────────┤
│  Body Store (ExpressionStoreSource::Body)               │
│  - Function body expressions                            │
│  - Initializer expressions                              │
└─────────────────────────────────────────────────────────┘
```

Each store has its own `ExpressionSourceMap` for mapping HIR `ExpressionId` back to AST nodes.

## The Bug

### Location
`crates/hir/src/diagnostics.rs:any_diag_from_infer_diagnostic()`

### Root Cause
Most `InferenceDiagnostic` variants only store an `ExpressionId`, not which `ExpressionStore` it came from. The conversion function always looks in the body source map:

```rust
// CORRECT - InvalidType checks the source
InferenceDiagnostic::InvalidType { source, error } => {
    let source_map = match source {
        ExpressionStoreSource::Body => source_map,
        ExpressionStoreSource::Signature => signature_map,
    };
    // ... uses correct map
}

// BUG - Always uses body map
InferenceDiagnostic::UnexpectedTemplateArgument { expression } => {
    let pointer = source_map.expression_to_source(*expression).ok()?.clone();
    //            ^^^^^^^^^^ Always body map - fails for signature expressions!
}
```

### Affected Diagnostics

| Diagnostic | Can occur in signature? |
|------------|------------------------|
| `UnexpectedTemplateArgument` | Yes - var templates |
| `WgslError` | Yes - type expressions |
| `ExpectedLoweredKind` | Yes - type contexts |
| `AssignmentNotAReference` | No - body only |
| `TypeMismatch` | Possibly - needs investigation |
| `NoSuchField` | No - body only |
| `ArrayAccessInvalidType` | No - body only |
| `UnresolvedName` | Possibly |
| `InvalidConstructionType` | Possibly |
| `FunctionCallArgCountMismatch` | Possibly |
| `NoBuiltinOverload` | Possibly |
| `NoConstructor` | Possibly |
| `AddressOfNotReference` | No - body only |
| `DerefNotAPointer` | No - body only |
| `CyclicType` | No - uses TextRange directly |

### Trigger Case

```wgsl
var<function> bad_global: u32;  // Error: function address space not allowed at module level
```

The error is created in `infer_global_variable`:
```rust
self.push_diagnostic(InferenceDiagnostic::UnexpectedTemplateArgument {
    expression: variable.template_parameters[0],  // From SIGNATURE store
});
```

But `UnexpectedTemplateArgument` doesn't track which store the expression came from. The conversion tries to find it in the body map, fails, returns `None`, and the diagnostic is silently dropped.

### Evidence

A warning is logged but easy to miss:
```rust
None => {
    tracing::warn!("could not create diagnostic from {:?}", diagnostic);
},
```

## Proposed Fix

Add `ExpressionStoreSource` to all `InferenceDiagnostic` variants containing `ExpressionId`:

```rust
// Before
pub enum InferenceDiagnostic {
    UnexpectedTemplateArgument {
        expression: ExpressionId,
    },
    // ...
}

// After
pub enum InferenceDiagnostic {
    UnexpectedTemplateArgument {
        source: ExpressionStoreSource,
        expression: ExpressionId,
    },
    // ...
}
```

Then update `any_diag_from_infer_diagnostic` to select the correct source map for each diagnostic.

### Files to Modify

1. `crates/hir_ty/src/infer.rs` - Add `source` field to affected variants
2. `crates/hir_ty/src/infer.rs` - Update all `push_diagnostic` calls to include source
3. `crates/hir/src/diagnostics.rs` - Update conversion to use correct map

## Testing Strategy

### Current Test Infrastructure

| Crate | Tests? | Notes |
|-------|--------|-------|
| hir_ty | Yes | `tests/simple.rs` - checks raw diagnostics |
| hir | No | Conversion untested |
| ide | No | Has `fixture.rs` helper but no tests |

### Recommended Approach: Focused Spot Tests

**1. hir_ty level** (already added):
```rust
#[test]
fn global_var_function_address_space_error() {
    check_infer(
        "var<function> bad_global: u32;",
        expect![[r#"
            23..33 'bad_global': ref<u32>
            UnexpectedTemplateArgument { expression: Idx::<Expression>(0) }
        "#]],
    );
}
```

**2. IDE level** (new test module in `crates/ide/src/diagnostics.rs`):
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::fixture::single_file_db;
    use hir::diagnostics::DiagnosticsConfig;

    fn check_diagnostics(source: &str, expected_count: usize) {
        let (analysis, file_id) = single_file_db(source);
        let config = DiagnosticsConfig {
            enabled: true,
            type_errors: true,
            ..Default::default()
        };
        let diagnostics = analysis.diagnostics(&config, file_id).unwrap();
        assert_eq!(diagnostics.len(), expected_count, "diagnostics: {diagnostics:?}");
    }

    #[test]
    fn signature_error_not_dropped() {
        // Error in signature context - verifies conversion works
        check_diagnostics("var<function> x: u32;", 1);
    }

    #[test]
    fn body_error_still_works() {
        // Regression test - body errors should still work
        check_diagnostics("fn f() { let x: u32 = 1.0; }", 1);
    }
}
```

### Test Coverage

| Test | Layer | Purpose |
|------|-------|---------|
| `global_var_function_address_space_error` | hir_ty | Document diagnostic is created |
| `signature_error_not_dropped` | ide | Verify signature→IDE conversion |
| `body_error_still_works` | ide | Regression test |
| 2-3 more spot tests | ide | Cover other signature error types |

### Why Minimal Tests?

- Fix is mechanical: add `source` field to each variant
- Once pattern established, all variants work identically
- 4-6 focused tests sufficient for confidence
- Over-testing creates maintenance burden

## Implementation Checklist

- [ ] Add `source: ExpressionStoreSource` to affected `InferenceDiagnostic` variants
- [ ] Update `push_diagnostic` calls to pass correct source
- [ ] Update `any_diag_from_infer_diagnostic` to use source field
- [ ] Add IDE-level test module
- [ ] Add 4-6 spot tests covering signature error scenarios
- [ ] Verify existing tests still pass
- [ ] Run full test suite
