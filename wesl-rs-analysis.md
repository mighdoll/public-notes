# wesl-rs Code Analysis for Name Resolution

Analysis of wesl-rs codebase to identify code that could be adapted for wgsl-analyzer's name resolution implementation.

## Key Files in wesl-rs to Reference

| File | Purpose |
|------|---------|
| `crates/wesl/src/import.rs` | Core import resolution - `flatten_imports()`, `resolve_decl()`, `resolve_inline_path()` |
| `crates/wesl/src/resolve.rs` | `Resolver` trait, `VirtualResolver` (for tests) |
| `crates/wgsl-parse/src/syntax_impl.rs` | `ModulePath::join_path()` for handling `super::`, `package::` (lines 136-184) |

## Concept Mapping

| wesl-rs | wgsl-analyzer | Notes |
|---------|---------------|-------|
| `Module` | `ModuleData` | wgsl-analyzer uses `FileId` as key instead of `ModulePath` |
| `Resolutions` | `DefMap` | `DefMap.modules` is the equivalent container |
| `Imports` (flattened) | `FlatImport` | Already done in `ImportTree::expand()` |
| `ModulePath` | `ModPath` | Similar structure, different names |
| `PathOrigin` | `PathKind` | Equivalent enum variants |
| `Resolver` trait | `SourceDatabase` + `PackageGraph` | Salsa queries replace trait methods |
| `VirtualResolver` | Needs test helper | For multi-file tests |

## Key Functions in wesl-rs

### `import.rs`

1. **`flatten_imports()`** (lines 352-411): Converts nested `ImportStatement` trees into flat `Imports` map. Analogous to wgsl-analyzer's `ImportTree::expand()`.

2. **`resolve_lazy()`** (lines 271-313): Lazy resolution - only loads modules actually used. Good for dead code elimination.

3. **`resolve_eager()`** (lines 316-349): Eager resolution - loads all imported modules. Simpler for IDE use.

4. **`load_module()`** (lines 131-143): Gets or loads a module by path from the resolver.

5. **`resolve_decl()`** (lines 173-206): Resolves a declaration by name, handling:
   - Local declarations (in `module.idents`)
   - Re-exported imports (`@publish import`)
   - Recursively loading external modules

6. **`resolve_inline_path()`** (lines 417-439): Handles inline paths like `foo::bar::baz()` where `foo` might be an import alias.

### `resolve.rs`

- `Resolver` trait (lines 33-55): Interface for loading modules by path
- `FileResolver` (lines 106-167): Loads from filesystem
- `VirtualResolver` (lines 173-221): In-memory modules (for tests)
- `Router` (lines 276-348): Dispatches to sub-resolvers by path prefix

## Key Insight: Fixed-Point Iteration

wesl-rs uses **fixed-point iteration** because imports can form cycles (module A imports from B, B imports from A). The `collector.collect()` needs this pattern:

```rust
fn collect(&mut self) {
    loop {
        let mut progress = false;
        let mut i = 0;

        while i < self.unresolved_imports.len() {
            let (file_id, import) = &self.unresolved_imports[i];

            match self.resolve_import(*file_id, import) {
                ResolveResult::Resolved(def) => {
                    // Add to scope
                    let scope = &mut self.def_map[file_id.file_id].scope;
                    if let Some(name) = import.leaf_name() {
                        scope.push_item(name.clone(), def);
                    }
                    self.unresolved_imports.swap_remove(i);
                    progress = true;
                }
                ResolveResult::NeedLoadModule(module_path) => {
                    self.load_module(&module_path);
                    progress = true;
                    i += 1;
                }
                ResolveResult::Unresolved => {
                    i += 1;
                }
            }
        }

        if !progress || self.unresolved_imports.is_empty() {
            break;
        }
    }
}
```

## Path Resolution Strategy

Based on wesl-rs's `resolve_inline_path()` and path logic:

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
            current_module = self.root;
        }
        PathKind::Super(levels) => {
            for _ in 0..levels {
                if let Some(parent) = self[current_module].parent {
                    current_module = parent;
                } else {
                    // Error: too many super::
                    return ResolvePathResult {
                        resolved_def: ...,
                        segment_index: Some(0)
                    };
                }
            }
        }
        PathKind::Plain => {
            // Check if first segment is an import alias
            // ...
        }
    }

    // Walk path segments through module tree
    for (idx, segment) in path.segments().iter().enumerate() {
        let scope = &self[current_module].scope;
        // ... resolve each segment
    }
}
```

## Test Infrastructure from wesl-rs

wesl-rs's `VirtualResolver` pattern for multi-file tests:

```rust
// From wesl-rs test setup
let mut resolver = VirtualResolver::new();
for (path, file) in &case.wesl_src {
    resolver.add_module(path, file.into());
}
```

For wgsl-analyzer, similar approach:

```rust
#[test]
fn test_import_resolution() {
    let mut fixture = Fixture::new();
    fixture.add_file("/main.wesl", "import foo::bar; fn main() { bar(); }");
    fixture.add_file("/foo.wesl", "fn bar() {}");

    let db = fixture.build();
    let def_map = db.package_def_map_query(fixture.package_id);

    let main_scope = &def_map[fixture.file_id("/main.wesl")].scope;
    assert!(main_scope.items.contains_key(&Name::from("bar")));
}
```

## Key Differences to Handle

1. **Incremental vs. Batch**: wesl-rs mutates modules in place; wgsl-analyzer needs immutable `DefMap` via Salsa queries.

2. **Module Identity**: wesl-rs uses `ModulePath`; wgsl-analyzer uses `FileId`. Need mapping via `SourceRoot::resolve_path()`.

3. **Package Dependencies**: wesl-rs's `Router` pattern maps to wgsl-analyzer's `PackageGraph.dependencies`.

4. **Re-exports (`@publish`)**: wesl-rs supports this. Can be skipped initially if not needed.

## Files to Modify in wgsl-analyzer

1. `crates/hir_def/src/nameres/collector.rs` - Implement `collect()`
2. `crates/hir_def/src/nameres/path_resolution.rs` - Implement `resolve_path_fp_with_macro()`
3. `crates/hir_def/src/item_scope.rs` - May need import tracking
4. `crates/hir_def/src/nameres/tests.rs` - Add multi-file test infrastructure
