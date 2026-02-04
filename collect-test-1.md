# Name Resolution Testing Infrastructure

## Current Test Infrastructure

### What Exists

| Layer | Test Support | Multi-file? |
|-------|--------------|-------------|
| **parser** | Excellent - `check()` with expect-test snapshots | No (single source string) |
| **hir_def** | `single_file_db()` + `check_item_tree()` | No |
| **ide** | `single_file_db()` in fixture.rs | No |
| **nameres** | Empty `tests.rs` file | - |

### The Gap: No Multi-File Test Support

The `DefMap` queries require a `PackageGraph` with `PackageId`, but:
- Current test helpers (`single_file_db`) don't set up a `PackageGraph`
- No fixture format for specifying multiple files with dependencies

## What's Needed to Test Name Resolution

To test the `todo!()` implementations, you'd need to:

```rust
// Hypothetical test helper needed:
fn multi_file_db(files: &[(&str, &str)]) -> (TestDatabase, PackageId) {
    // 1. Create FileIds for each file
    // 2. Build a PackageGraph with root file + dependencies
    // 3. Set up source roots
    // 4. Return database + package ID for querying DefMap
}

#[test]
fn resolve_simple_import() {
    let (db, pkg) = multi_file_db(&[
        ("lib.wesl", "import foo::bar; fn main() { bar(); }"),
        ("foo.wesl", "fn bar() {}"),
    ]);
    let def_map = db.package_def_map_query(pkg);
    // Assert bar is in scope in lib.wesl
}
```

## Difficulty Assessment

| Task | Effort |
|------|--------|
| Write `multi_file_db()` helper | Medium - need to understand PackageGraph construction |
| Test `collector.collect()` | Easy once helper exists - just snapshot `DefMap::dump()` |
| Test path resolution | Easy once collector works |

## Quick Win: Test ItemTree with imports

You can already test that imports are correctly parsed into the ItemTree:

```rust
#[test]
fn item_tree_with_imports() {
    check_item_tree(
        "import foo::bar;",
        expect![[/* snapshot of import in item tree */]],
    );
}
```

## Next Steps

Options:
1. Create the `multi_file_db()` test helper
2. Start implementing `collector.collect()` and see what breaks
