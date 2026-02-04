# wesl-js Name Binding Analysis for wgsl-analyzer

Analysis of wesl-js codebase to identify name binding algorithms that could be adapted for wgsl-analyzer.

## Key Files and Their Purposes

| File | Purpose |
|------|---------|
| `tools/packages/wesl/src/BindIdents.ts` | Main binding pass - core name resolution algorithm |
| `tools/packages/wesl/src/Scope.ts` | Scope data structures (`LexicalScope`, `PartialScope`) |
| `tools/packages/wesl/src/ModuleResolver.ts` | Module resolution interface and implementations |
| `tools/packages/wesl/src/ModulePathUtil.ts` | Path resolution (`package::`, `super::` handling) |
| `tools/packages/wesl/src/FlattenTreeImport.ts` | Import tree flattening |

## Core Algorithm: `bindIdents()`

```
1. findValidRootDecls(rootScope, conditions) - Filter declarations based on @if/@else
2. initRootDecls(validRootDecls) - Initialize global names set and known declarations
3. Create LiveDecls map for tracking visible declarations
4. For each valid root decl: processDependentScope() - Follow references within declarations
5. bindIdentsRecursive() - Walk scope tree depth-first
   - For each ref ident: handleRef() which:
     a. findDeclInModule() - Search local scopes (up through parents)
     b. findQualifiedImport() - Match against import statements
     c. findExport() - Resolve to external module declaration
     d. Check if stdWgsl() - Built-in identifier
```

## Key Data Structure: LiveDecls (Scoped Symbol Table)

```typescript
interface LiveDecls {
  decls: Map<string, DeclIdent>;  // name -> declaration
  parent?: LiveDecls | null;       // parent scope (for lookup chain)
}
```

Simple scoped symbol table with parent links for lexical scoping. Maps directly to wgsl-analyzer's `ItemScope`.

## Path Resolution Algorithm

From `ModulePathUtil.ts` - `resolveModulePath()`:

```typescript
function resolveModulePath(parts: string[], srcModuleParts: string[]): string[] {
  // Handle package:: - replace with actual package name from source module
  const resolved = parts[0] === "package"
    ? [srcModuleParts[0], ...parts.slice(1)]
    : parts;

  // Handle super:: - climb up the module path hierarchy
  const lastSuper = resolved.lastIndexOf("super");
  if (lastSuper === -1) return resolved;
  const base = srcModuleParts.slice(0, -(lastSuper + 1));
  return [...base, ...resolved.slice(lastSuper + 1)];
}
```

## Import Flattening

From `FlattenTreeImport.ts`:

```typescript
// Input: import foo::bar::{baz, cat as neko}
// Output: [{importPath: ["baz"], modulePath: ["foo", "bar", "baz"]},
//          {importPath: ["neko"], modulePath: ["foo", "bar", "cat"]}]
```

Already implemented in wgsl-analyzer as `ImportTree::expand()`.

## Clever Techniques

### 1. Lazy Module Loading

Modules parsed on-demand via `ModuleResolver.resolveModule()`. Similar to salsa queries.

### 2. Conditional Filtering (`@if/@elif/@else`)

```typescript
function* validItems(scope: Scope, conditions: Conditions) {
  let elseValid = false;
  for (const item of scope.contents) {
    const cond = validateConditional(getCondAttr(item), elseValid, conditions);
    elseValid = cond.nextElseState;
    if (cond.valid) yield item;
  }
}
```

### 3. Deduplication via `knownDecls` Set

Prevents re-processing the same declaration when imported through multiple paths.

### 4. `accumulateUnbound` Mode for IDE Features

Collect unresolved references without throwing errors - useful for partial/broken code:

```typescript
if (!bindContext.unbound)
  failIdent(ident, `unresolved identifier '${ident.originalName}'`);
// else: silently add to unbound list for IDE to show
```

## Edge Case Handling

### Circular Imports

wesl-js handles this naturally because:
- Each module is parsed independently
- `knownDecls` set prevents infinite recursion during binding
- Test case "circular import" covers: file1 imports file2, file2 imports file1

### Forward References

No special handling needed - root-level declarations are collected first, then binding happens:

```typescript
const validRootDecls = findValidRootDecls(rootAst.rootScope, conditions);
// All declarations visible before processing any references
```

### Error Recovery

`accumulateUnbound` mode for IDE tolerance of broken code.

## Adaptation for wgsl-analyzer

### For `resolve_path_fp_with_macro()`:

Based on wesl-js's `findQualifiedImport()` + `findExport()`:

```rust
pub(super) fn resolve_path_fp_with_macro(
    &self,
    db: &dyn DefDatabase,
    original_module: FileId,
    path: &ModPath,
) -> ResolvePathResult {
    let mut current_module = original_module;

    // Handle path kind (package::, super::, plain)
    match path.kind() {
        PathKind::Package => {
            current_module = self.crate_root();
        }
        PathKind::Super(levels) => {
            for _ in 0..levels {
                if let Some(parent) = self.containing_module(current_module) {
                    current_module = parent;
                } else {
                    return ResolvePathResult {
                        resolved_def: ModuleDefinitionId::ERROR,
                        segment_index: Some(0),
                    };
                }
            }
        }
        PathKind::Plain => {
            // First segment might be:
            // 1. An import alias
            // 2. A direct declaration name
            // 3. An external package name
        }
    }

    // Resolve each segment
    for (idx, segment) in path.segments().iter().enumerate() {
        if let Some(def) = self[current_module].scope.get(segment) {
            if let Some(next_mod) = def.as_module() {
                current_module = next_mod;
            } else if idx == path.segments().len() - 1 {
                return ResolvePathResult {
                    resolved_def: def,
                    segment_index: None,
                };
            } else {
                return ResolvePathResult {
                    resolved_def: def,
                    segment_index: Some(idx + 1),
                };
            }
        } else {
            return ResolvePathResult {
                resolved_def: ModuleDefinitionId::ERROR,
                segment_index: Some(idx),
            };
        }
    }

    ResolvePathResult {
        resolved_def: ModuleDefinitionId::Module(current_module),
        segment_index: None,
    }
}
```

## Test Cases from wesl-js

The test suite at `wesl-testsuite/src/test-cases/ImportCases.ts` covers:

- Basic imports: `import package::bar::foo`
- Aliasing: `import foo as bar`
- Collections: `import pkg::{foo, bar}`
- Transitive imports (diamond patterns)
- Circular imports
- Name conflicts and mangling
- `super::` relative paths
- Inline qualified references: `package::file::fn()`

## Differences from wesl-rs

| Aspect | wesl-js | wesl-rs | wgsl-analyzer |
|--------|---------|---------|---------------|
| Approach | Single-pass, lazy | Fixed-point iteration | Fixed-point + incremental |
| Phases | Merged collect+resolve | Separate phases | Separate phases |
| Errors | Throw exception | Result type | Diagnostics |
| Partial state | `accumulateUnbound` mode | N/A | Via Salsa queries |

## Key Takeaways

1. **Single-pass works for non-incremental**: wesl-js doesn't need fixed-point because it resolves lazily during the walk
2. **`accumulateUnbound` pattern**: Good for IDE tolerance - collect errors without failing
3. **`knownDecls` deduplication**: Essential for avoiding infinite loops with re-exports
4. **Conditional filtering**: `@if/@else` handling is a generator pattern that could be adapted
5. **Path resolution is simple**: The `resolveModulePath()` function is straightforward to port
