## Meta
- Std formats lead to standard tools. We can't have tools like wgsl-analyzer and wgsl-test or sharable libraries if every developer hacks their own bespoke approach to shader assembly.
- We want to deliver native browser ergonomics with minimal tools, not just use WGSL as an output format. ergonomics isn't about enabling users. We can already do this kind of thing with enough tooling. It's about making things easy and universal in the browser.

## Background from WESL
- We reported [many examples in 2024](https://docs.google.com/presentation/d/e/2PACX-1vSNZ0Y634bhYbVnFjNWg5X4qbmAxWvgHOUFdMZM3H4K6fRcYyMwG-SPPW5BDR1ChHnLb0V9IONCndR8/pub?start=false&loop=false&delayms=3000#slide=id.g30bc7b45552_0_15)
  of [community ad-hoc workarounds](
   https://docs.google.com/presentation/d/e/2PACX-1vSNZ0Y634bhYbVnFjNWg5X4qbmAxWvgHOUFdMZM3H4K6fRcYyMwG-SPPW5BDR1ChHnLb0V9IONCndR8/pub?start=false&loop=false&delayms=3000#slide=id.g2f924c2f0a7_0_10).
- Module system was the #1 user problem when we surveyed users and open source projects.
  Solving that was WESL's first goal in 2024.
- Conditions was #2 problem. Likely that something lands in WGSL eventually, std is needed for common tooling.
- [WESL's home page](https://wesl-lang.dev/) has a simple playable example of WESL module system.
- WESL has 3 implementations (wesl-rs, wesl-js, wgsl-analyzer), shared test suite, spec.
- Module system has evolved a bit since introduction, we added library support, simplified to avoid network probing, and then built user requested features on top of it  (e.g. Bevy wanted visibility controls and wildcard imports for preludes).
- The module system doesn't depend on anything else in WESL, AFAIK. (But we build several other features atop modules.)
- Major design challengers still open but growing less likely: 
  ocaml modules instead of `impl` might make namespaces on steroids more tempting. 
  Relative urls for imports (zig, JavaScript) might make for tighter web integration.
- Bevy and Lygia have shipped WESL. Those are new and not much spread into their user communities. We've dozens, but not hundreds of WESL users.
- Designing the WebGPU interface here is new exercise.

## Basic Idea
```ts
createShaderModule({ sources }); // keys are :: separated paths, values are WGSL
```

```wgsl
import package::util::{foo};     // WGSL gets import statements

fn bla() { foo(); }
```

## Benefits of a module system:
- trivial mapping to files, basically just `/` <=> `::`
  - no transpilation of wgsl text required, browser native ergonomics
  - source maps are mostly unnecessary, column and lines are correct already (sourcemaps are still good for WESL support tho :-))
- straightforward extension to libraries (w/o source-rewriting linkers)
    ```wgsl
    import lygia::math::nyquist;    // import from libraries
    ```
- extends to naturally skip unused module that says `enable f16` (or `@if`)
- extends to support ESM style loading of modules. 
    ```ts
    import loaded from "./main.wgsl" with { type: "wgsl"};  // loads recursively from main
    createShaderModule({...loaded});
    ```
- [gpuweb#5456](https://github.com/gpuweb/gpuweb/issues/5456) - users want code sharing standards, not home grown.
- (also provides a spelling for how to reach WGSL builtins, `wgsl::min`)

## Stage 1:
- Add `sources` option to `createShaderModule`
- Add `source` field to `GPUCompilationMessage`
- Add `import` statement to WGSL.
- (no fetching, no library package dependencies)

## Stage 2: 
- Add dependencies.
  Each entry has its own sources and its own view of what dependency names mean.
  ```ts
  createShaderModule({ code, packages: { lygia: { sources: lygiaSources } } });
  ```
  ```ts
  createShaderModule({
    code,                              // imports lygia::... and rend::...
    packages: {
      lygia:       { sources: lygiaSources },
      "lygia-old": { sources: lygiaSourcesOld },
      rend:        { sources: rendSources, dependencies: { lygia: "lygia-old" } },
    },
  });
  ```

## vs. namespaces
- Name outside the text vs. inside the text
  - filenames vs. in-text names lead to drift / mismatch rules (sometimes an error, sometimes an IDE lint).
  - library versioning via in-text paths requires rewriting. 
  - library multi-version support requires some per importer binding mechanism like the [namespaces + dependencies](https://github.com/gpuweb/gpuweb/pull/7310#issuecomment-5632757314) sketch in 7310. But that proposal's module names on the inside, package names on the outside is adopting the essential machinery for modules anyway.
- Closed vs reopenable code units
  - reopenable namespaces hurts tools like wgsl-analyzer (need every module to analyze anything).
    and hurts fetching (need to fetch every module, can't fetch only mobile relevant ones),
  - reopenable namespaces make things difficult for visibility. `private` can't mean private to the namespace when any file can reopen it. 
    Languages with reopenable namespaces end up needing another unit for visibilty control.
- Other issues
  - 7310's {} does not create a lexical scope, unlike other uses of `{}` in WGSL.
    (but lexical nesting would have other problems, e.g. wouldn't map to files well, and there'd be spooky actions at a distance when an outer file introduces a colliding declaration name)
  - `using` flattens names into scope which can conflict. We considered this when we looked wildcard imports for WESL. For libraries, adding a new declaration can break consumers when they upgrade. Many host languages restrict wildcards. Because we want WESL shaders to fit in host package systems, we restrict wildcards in [WESL](https://wesl-lang.dev/spec/ImportsDesign#wildcards-in-wesl-when-to-allow-when-to-gate) too.
  - namespaces & modules can be done together, but adds complexity. Many languages have modules but not namespaces, we may not need both.

## Future stages
- ESM style loader (syntax noted above). Browser fetcher reads the first lines in each module to find import statements, fetches modules recursively from the net. No full WGSL parsing/binding required.
- (WESL) compound import statements (WESL allows `import foo::{a, b::c};` )
- (WESL) wildcard imports (WESL allows `import foo::*;` with limitations)
- (WESL) visibility (`private` `public`, for libraries and larger projects)
- (WESL) `super::` (like `../`)
- (WESL) select a 'main' module; only modules it reaches are compiled. A `mobile_main` never reaches the module with `enable subgroups`, so one set of sources works everywhere without conditions. (`@if` conditions can use this reachability later too for finer grain control).

## Things to change in current WESL to make module fetching easy
- require import statements, and limit inline module paths to two segments. (no `lygia::bar::zap();`, instead first `import lygia::bar;`, then `bar::zap();`)
- require brackets on element imports `import lygia::bar::{zap};`

## Questions for making the proposal
- for stage 1, allow element imports, or module imports, or both? WESL allows both
- for stage 2 dependencies, dictionaries or objects? see alt below (presuming dictionaries)
- Do we understand the restrictions on a future ESM style fetch mode? 
- Do we understand restrictions from security boundary for the content process?

## Questions for the committee
- Do we want relative URLs in import statements (like JS or zig) instead of `::` or `.`.
- If the committee likes the direction, what additional incubation / learning would we like first?
- Do we understand the restrictions on a future ESM style fetch mode/browser process?
- Ought we syntactically or semantically validate every string passed in or only reachable modules?
  (leaving unreachable modules unvalidated is nice for e.g. unused `enable f16` module.)
- Do we want qualified names (with `::`) in the WebGPU API for entrypoints and overrides? WESL uses the main module to define the host code visible API (enforcing modularity, nested libraries can't expose to the host api). The main module uses `public import` to pull names from other modules into the host visible API. This allows us to show only simple names (with no `::` inside) to the host for overrides and entry points.
So simple-names-only is possible, what's preferable?

### `::` vs `.` vs `/` 
WESL debated in 2024, [notes](https://hackmd.io/ljkByEcnQa2NdNLWed2M6Q). Both work. WESL went with `::`, but would change to match WGSL decision. 

Claude driven language survey suggests switching to `.` would be a bit better:
- `.` is the majority across surveyed languages; Carbon #989 rejected a `::` split for lack of evidence it helps.
- Swift's 2026 `Module::name` (SE-0491) fixed a module hiding its own members behind a same-named type.
  WESL avoids that: import bindings share the declaration namespace, conflicts are errors.
- No lexical ambiguity; template discovery already works after any identifier.
- Grammar cost: a `path` nonterminal (`ident ('.' ident)*`) shared by call, type, and lhs positions so LALR(1) holds; `a.b(x)` and `a.b<T>` are not WGSL today.
- Semantics: module vs. member boundary is decided in name resolution; local vars shadow import names (same as consts today).
- Use `.` in imports too for consistency; `.` <-> `/` maps to files the way Java/Python do.
- "require imports, max two-segment inline paths" preserves easy fetching - fetcher only reads import statements.
- Interestingly slang allows _both_ `.` and `::` for namespace paths, see the [user guide](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/03-convenience-features.html#namespaces).. 
- Benefit: one membership concept, familiar to JavaScript/Python/Swift/C#/Java users. (also saves a few characters)
- Cost: larger grammar change, a little further from Rust/C++.

## Cross language survey

- Several languages with modules considered but rejected namespaces. [carbonlang](https://github.com/carbon-language/carbon-lang/blob/trunk/proposals/p000107-code-and-name-organization.md#scoped-namespaces), es6 considered by dropped in 2013, TS added but [discourages namespaces](https://github.com/microsoft/TypeScript/issues/30994), swift, etc. 
  - most cite unnecessary complexity, carbon argues harder to read (need search from closing `}`), 
- `.` is the most popular as a separator e.g. C#, Java/Scala/Kotlin, Swift, Js/Ts, Carbon, Python, Go, OCaml. C++ and Rust use `::`
- Only a few languages have used library-version-in-name:
  go uses [major version only suffixes](https://go.dev/ref/mod#major-version-suffixes), but it's controversial. 
  c++ did use it (`inline namespace`), but e.g. [google style](https://google.github.io/styleguide/cppguide.html#Namespaces) guide says avoid.
- Namespaces solve naming; libraries also need composition (include-once,
  dependencies, a visibility and version boundary). Every language that started
  with namespaces added a composition unit later and ended up with two
  mechanisms. Slang has both, and its bugs and its 2026 proposals sit at the
  seam between them. Modules do both with one mechanism.
- Slang has tried the complex route of both modules and namespaces. They've seen some problems:
  - Slang's `import` mixes a module's global declarations into the importer
    unqualified, as with wildcard imports or `using` in 7310.
    Their [language guide](https://docs.shader-slang.org/en/latest/external/slang/docs/language-guide.html) noted: "collisions between different imported files ... are possible. This is a bug." 
    The workaround is to wrap things in `namespace lib {}`, but then in 2026 they are trying to drop the workaround with [aliased imports](https://github.com/shader-slang/slang/issues/12972) and [semantic modules](https://github.com/shader-slang/slang/issues/12985), dropping the workaround by making the module the thing for qualifying names, which is the WESL design.
  - Namespaces can be reopened from any module, but this has been a problem area: 
    e.g. a [breaking change](https://github.com/shader-slang/slang/issues/11443) in July 2026 to address `using namespace` in one module from traveling into imported modules, 
    reopened-namespace lookup had an [ordering bug](https://github.com/shader-slang/slang/issues/11531) until June 2026, 
    and reopened + nested + imported member is [still open](https://github.com/shader-slang/slang/issues/11442). 
    `import` inside a namespace was [closed as not planned](https://github.com/shader-slang/slang/issues/2208). 
    Mixing the namespace and module axes hasn't been trivial for Slang.
  - Visibility lives on the 
    [modules](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/04-modules-and-access-control.html#access-control), not namespaces. 
  - The In-text module names drift from file names. You import a module by its file
    name, but helper files join the module by the in-text name (`implementing foo;`).
    `module foo;` in `bar.slang` imports only as `bar`. Doc examples always match the two names, 
    but it's not enforced.
  - No multi-version libraries allowed yet, and the 
    [package manager proposal](https://github.com/shader-slang/slang/pull/12656) keeps one package name per node.

## Future WESL designs 
- module parameters (WESL proposal, completes conditions)
  ```ts
  createShaderModule({ ...loaded, parameters: { MOBILE: true } });
  ```
  ```wgsl
  fn foo() {
    @if(!MOBILE) doExpensive();
  }
  ```
  ```wgsl
  import foo::a<DEBUG = true>;
  ```
- reflection (WIP, pre-proposal)
  ```ts
  const m = createShaderModule({ ...loaded, parameters: { MOBILE: true } });
  const info = await m.getReflection();
  ```
----

## Notes

- `code` in `createShaderModule` is equivalent to `sources: { "package": "..." }`.
  Contained declarations in `code` are referenced at the root as `import package::decl;` 
  like rust's `lib.rs`. 

  This leaves room for libraries later to allow `import mylib::decl;` w/o an extra level in the path.
  passing `code` and `sources: { package: "..."} ` is an error.

### Stage 2 via package objects 
- Add dependencies:
  ```ts
  const lygia = createShaderPackage({sources: lygiaSources});
  createShaderModule({code, packages: { lygia }})
  ```
  ```ts
  const lygia = createShaderPackage({ sources: lygiaSources });
  const lygia_old = createShaderPackage({ sources: lygiaSourcesOld });
  const rend = createShaderPackage({ sources, packages: { lygia: lygia_old }});
  createShaderModule({code, packages: { lygia, rend }});
- benefit: more efficient internally for many createShaderModule() calls with same packages?.
- cost: fetcher returns data, doesn't make webgpu calls, so converter is needed.