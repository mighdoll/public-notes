## Background from last meeting

- WebGPU should eventually provide APIs to guide standard use of files, libraries, urls, and conditional compilation.
- users ask for it, std formats lead to std tools. helps the ecosystem, etc.
- browser native ergonomics is the win here, invasive tooling can treat WGSL as an output format, but that's not the goal.
- WESL module system has been pretty stable, will get more vetting over time.
- WebGPU api for module style is new here.

## Stage 1 what to ship first
```ts
createShaderModule({ sources }); // keys are :: separated paths, values are WGSL
```

```wgsl
import package::util::{foo};     // WGSL gets import statements

fn bla() { foo(); }
```
- Add `sources` option to `createShaderModule`
- Add `source` field to `GPUCompilationMessage`
- Add `import` statement to WGSL.
- Compile the main module and the modules it reaches through imports, transitively.
  Unreached entries in `sources` are not examined. An import naming a module missing from `sources` is a creation error.
  (Everything a pipeline can use is validated at creation; what goes unvalidated is exactly what no pipeline can reach.)
- (no fetching, no library package dependencies)

## Stage 2: 
- Add dependencies:
  Each package has its own sources and its own view of what dependency names mean.
  ```ts
  const lygia = new GPUShaderPackage({ sources: lygiaSources });
  createShaderModule({ code, packages: { lygia } });
  ```
  ```ts
  const lygia = new GPUShaderPackage({ sources: lygiaSources });
  const lygia_old = new GPUShaderPackage({ sources: lygiaSourcesOld });
  const rend = new GPUShaderPackage({ sources: rendSources, packages: { lygia: lygia_old } });
  device.createShaderModule({ code, packages: { lygia, rend } });
  ```
- Packages are not tied to a device, they can be passed to multiple createShaderModule calls.
- Add `package` field to `GPUCompilationMessage`.

## Future stages
- (stage E) ESM style loader (syntax noted above). 
  Browser fetcher reads the first lines in each module to find import statements, fetches modules recursively from the net. 
  No full WGSL parsing/binding required.
- (WESL) compound import statements (WESL allows `import foo::{a, b::c};` )
- (WESL) wildcard imports (WESL allows `import foo::*;` with limitations)
- (WESL) visibility (`private` `public`, for libraries and larger projects)
- (WESL) `super::` (like `../`)
- (WESL) select a 'main' module; only modules it reaches are compiled. 
  A `mobile_main` never reaches the module with `enable subgroups`, so one set of sources works everywhere without conditions. 
  (`@if` conditions can use this reachability later too for finer grain control).

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

## vs. namespaces
Modules are a bit better than namespaces in stage 1 - they map more closely to files, work w/o source mapping, and don't have lexical nesting, reopenability, or name collision pitfalls.
For stage 2 and beyond, modules are the right base. Libraries, visibility and conditions work well with modules and not so well with namespaces.

Details:
- Name outside the text vs. inside the text
  - filenames vs. in-text names lead to drift mismatch rules and tooling in other languages which creates unnecessary maintenance.
  - library versioning via in-text paths requires rewriting. 
  - library multi-version support requires some per importer binding mechanism 
    like the [namespaces + dependencies](https://github.com/gpuweb/gpuweb/pull/7310#issuecomment-5632757314) sketch in 7310. 
    But that proposal is module names on the inside, package names on the outside, adopting the essential machinery for modules anyway. (Stage 2 in this proposal)
- Closed vs reopenable code units
  - reopenable namespaces hurts tools like wgsl-analyzer (need every module to analyze anything).
    and hurts fetching (need to fetch every module, can't fetch only mobile relevant ones),
  - reopenable namespaces make things difficult for visibility. `private` can't mean private to the namespace when any file can reopen it. 
    Languages with reopenable namespaces end up needing another organizational unit for visibilty control.
- 7310's {} does not create a lexical scope, unlike other uses of `{}` in WGSL.
  - lexical nesting would have other problems, e.g. wouldn't map to files well, 
    and there'd be spooky actions at a distance when an outer file introduces a colliding declaration name
- `using` flattens names into scope which can conflict. This is risky for libraries,
  because adding a new declaration can break consumers when they upgrade. 
  For that reason, many host languages restrict wildcards. 
  To enable working with a variety of host package systems,
  we restrict wildcards in [WESL](https://wesl-lang.dev/spec/ImportsDesign#wildcards-in-wesl-when-to-allow-when-to-gate).
  (wildcards are allowed, but intended mostly for preludes)
- namespaces & modules can be done together, but that adds complexity. 
  Many languages have modules but not namespaces, we may not need both.

## Javascript considered some of these issues too:
- ES6 drafts had inline `module "foo" { }` beside file modules, and TC39 cut it in
  [Sept 2013](https://github.com/tc39/notes/blob/main/meetings/2013-09/modules.pdf):
  "Biggest simplification: eliminate inline modules", "controversial and complex feature".
  Rossberg's [objection](https://esdiscuss.org/topic/module-naming-and-declarations) was basically the vs. namespaces arguments. 
  The lesson was to separate the internal name from the external names (urls) is also relevant for us.
- The JS 2011 [design rationale](https://web.archive.org/web/20130619151033id_/http://wiki.ecmascript.org/doku.php?id=harmony:modules_rationale) also visited
  the same issue of versioning and naming.
  "It also makes versioning easy: multiple versions of a library can be separately loaded,
  without the library writer having to provide them with explicitly distinct names."
- ES4 actually had something [very similar](https://archives.ecma-international.org/2008/TC39/tc39-2008-021.pdf) to 7310:
  `ns::x` qualification, a `use namespace` opener (flattening), and packages
  "defined across disjoint pieces of text" (reopening).
  TC39 removed packages in April 2008 because reopening made them non-scoping:
  "packages appear to be scoped entities but are not".
  ES4's `use namespace` opener went out with namespaces in 2008.
- JavaScript deliberately has no `using namespace` that flattens names into scope. `import * as ns` binds 
   one name only.

## Things we'd change from current WESL to make module fetching easy
- require import statements, and limit inline module paths to two segments. 
  (no `lygia::bar::zap();`, instead first `import lygia::bar;`, then `bar::zap();`)
- require brackets on element imports `import lygia::bar::{zap};`
- `@if` on an import stays legal, but a loader ignores conditions and fetches anyway. 
  No condition evaluation in the fetcher.
  Fetched but conditionally unreferenced modules are ignored in createShaderModule.
  - The fetcher skips modules unreferenced from the main module, no change. 
    main_desktop.wesl and main_mobile.wesl can fetch a different recursive set of modules.

## Questions to discuss 
- for stage 1, allow element imports, or module imports, or both? WESL allows both
- for stage 2, `new GPUShaderPackage` ?. A spec object holding bulk text outside any device is new for WebGPU..
- Do we understand restrictions from security boundary for the content process?
- Alt direction - do we want relative URLs in import statements (like JS or zig) instead of `::` or `.`.
- If the committee likes the direction, what additional incubation / learning would we like first? 
  Should we have a proposal status where we create a polyfill?
- Do we understand the restrictions on a future ESM style fetch mode/browser process?
- Do we want to keep the door open for native loading in the future?
  TC39 JavaScript wanted native loading/import maps, even though bundling was and is popular (see the ESM story below).
  Suggests we will likely someday too want to native loading.
  But it leads us to minor restrictions to the language, like requiring brackets on import statements, which we would otherwise not do.
- Stage 1 proposes compiling only reached modules; an unreached `enable f16` module is never examined.
  The 7310 noted that currently we validate everything passed in. Objections to leaving unreached strings unvalidated?
  - JS precedent: an ES module nobody imports is never fetched, let alone parsed; no runtime loader validates unreached units.
    Whole-project checking is a toolchain job (TypeScript's program is [include-based](https://www.typescriptlang.org/tsconfig/), editors and CI check globs).
    For us that's a separate validate-everything API or wgsl-analyzer, not `createShaderModule`.
- Do we want qualified names (with `::`) in the WebGPU API for entrypoints and overrides too? 
  Seems nice, but note that it's not strictly necessary, WESL doesn't currently use that.
  WESL instead uses the main module to define the host code visible API (enforcing modularity, nested libraries can't expose to the host api). 
  The main module uses `public import` to pull names from other modules into the host visible API. 
  This allows us to show only simple names with no `::` inside to the host for overrides and entry points.
  So simple-names-only is possible, what's preferable?

## Options

### `::` vs `.` vs (import `/` and inline `.`)
WESL debated in 2024, [notes](https://hackmd.io/ljkByEcnQa2NdNLWed2M6Q), 
mostly considering `::` vs.  `/` on imports plus `.` inline. 
Both work. 
WESL went with `::`, but would change to match WGSL decision. 

Language survey suggests switching to `.` everywhere (import foo.bar, enum.enumerant, module.fn()):
- Joke: '4x fewer dots' -lee. 'good point' -stefan.
- Benefit: one membership concept. familiar to JavaScript/Python/Swift/C#/Java users. a little conciser. 
- Cost: larger grammar change. a little further from Rust/C++.
- `.` is the majority across surveyed languages, Carbon was explicit in #989: rejected a `::` split for lack of evidence it helps.
- Parsing is ok
  - No lexical ambiguity; template discovery already works after any identifier.
  - Grammar cost: a `path` nonterminal (`ident ('.' ident)*`) shared by call, type, and lhs positions so LALR(1) holds; `a.b(x)` and `a.b<T>` are not WGSL today.
- Interestingly slang allows _both_ `.` and `::` for namespace paths, 
  see the [user guide](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/03-convenience-features.html#namespaces).. 
  Probably we'd do that in WESL but only for a transition period if we switch from `::` to `.`.

- (Swift's 2026 `Module::name` (SE-0491) fixed a module hiding its own members behind a same-named type.
  WESL avoids that: import bindings share the declaration namespace, conflicts are errors.)

## Cross language survey

- Several languages with modules considered but rejected namespaces. 
  [carbonlang](https://github.com/carbon-language/carbon-lang/blob/trunk/proposals/p000107-code-and-name-organization.md#scoped-namespaces), es6 considered but dropped in 2013, 
  TS added but [discourages namespaces](https://github.com/microsoft/TypeScript/issues/30994), 
  Kotlin had namespace blocks and [removed them in 2012](https://blog.jetbrains.com/kotlin/2012/01/the-great-syntactic-shift/), 
  Swift considered namespaces repeatedly (2015, 2016, 2018 pitches) and never accepted them, etc. 
  most cite unnecessary complexity. 
  carbon also argues they're hard to read (need search from closing `}`), 
- `.` is the most popular as a separator e.g. C#, Java/Scala/Kotlin, Swift, Js/Ts, Carbon, Python, Go, OCaml. C++ and Rust use `::`
- Only a few languages have used library-version-in-name:
  go uses [major version only suffixes](https://go.dev/ref/mod#major-version-suffixes), but it's controversial. 
  c++ did use it (`inline namespace`), but e.g. [google style](https://google.github.io/styleguide/cppguide.html#Namespaces) guide says avoid.
- Namespaces solve naming; libraries also need composition 
  (include-once, dependencies, a visibility and version boundary). 
  Pretty much every language that started with namespaces added a composition unit later and ended up with two mechanisms. 
  Modules do both with one mechanism.
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
  - In-text module names drift from file names. You import a module by its file
    name, but helper files join the module by the in-text name (`implementing foo;`).
    `module foo;` in `bar.slang` imports only as `bar`. Doc examples always match the two names, 
    but it's not enforced.
  - No multi-version libraries allowed yet, and the 
    [package manager proposal](https://github.com/shader-slang/slang/pull/12656) keeps one package name per node.

## The ESM story: standard syntax first, native loading later, bundling throughout

- Before ES6, JS organized large programs with in-text namespaces (YUI, Dojo, Closure's `goog.provide`):
  object paths plus a string that had to match a file path by convention, rebuilt by every library.
  Two competing module systems (CommonJS, AMD) split server from browser.
  Bundling was already the norm: "Every serious application development environment in the world has a build step"
  ([Dale, 2012](https://tomdale.net/2012/01/amd-is-not-the-answer/)).
- TC39 standardized the syntax and static linking first. The 2011
  [rationale](https://web.archive.org/web/20130619151033id_/http://wiki.ecmascript.org/doku.php?id=harmony:modules_rationale)
  argued static scoping, early errors for unresolvable imports, and non-blocking loading.
  Tooling was not the argument: tree shaking appears nowhere in 2011-2014. It was a retrofit that then confirmed the design
  ([v8.dev](https://v8.dev/features/modules): "Static `import` and `export` are more than just syntax; they are a critical tooling feature!").
- The loader was the unfinished half. Out of time in [Sept 2014](https://github.com/tc39/notes/blob/main/meetings/2014-09/sept-25.md),
  TC39 moved the loader pipeline to a separate WHATWG spec and kept one host hook:
  "there's a request for a module name (from referrer), give me the source code".
  ES2015 shipped June 2015 with module syntax "but without any way to actually run them"
  ([Denicola](https://blog.whatwg.org/js-modules)).
- HTML specc'd loading `<script type="module">` in [Jan 2016](https://github.com/whatwg/html/pull/443).
  Bare specifiers (`import "jquery"`) were reserved as errors, and resolved only when import maps reached
  the last engine in [2023](https://caniuse.com/import-maps). 
- The platform built native loading and import maps despite the popularity of bundling before and after.
  Native loading makes the no-build path work: small apps, dev servers, examples, playgrounds.
- The key part is the layering: the language owns syntax and linking, the host owns fetching.
  That boundary has held 11 years across `import()`, import attributes, CSS/JSON modules, import maps, and the Node and Deno loaders,
  without one change to `import`/`export` or to linking.

Lessons:
- Standardize syntax and linking first. That layer is what the ecosystem (bundlers, analyzers, tree shaking, shared libraries) builds on.
- Expect to want native loading later even though bundling dominates. The descriptor is the bundle format; a fetcher is the dev and small-app path.
- Reserve the specifier space, never give it a provisional meaning. HTML's "fails for now" is why import maps landed five years later without breaking a page.

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