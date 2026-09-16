```ts
createShaderModule({ sources }); // keys are :: separated paths, values are WGSL
```

```wgsl
import package::util::{foo};     // WGSL gets import statements

fn bla() { foo(); }
```

## Benefits of a module system:
- (also provides a spelling for how to reach WGSL builtins, e.g. `wgsl::min`)
- trivial mapping to files, basically just "/" <=> "::"
  - no transpilation of wgsl text required, browser native ergonomics
  - source maps are mostly unnecessary, column and lines are correct already 
- straightforward extension to libraries (w/o rewriting linkers)
    ```wgsl
    import lygia::math::nyquist;    // import from libraries
    ```
- extends to naturally skip unused module that says `enable f16` (or `@if`)
- extends to support ESM style:
    ```ts
    import loaded from "./main.wgsl" with { type: "wgsl"};  // loads recursively from main
    createShaderModule({...loaded});
    ```
- gpuweb#5456 - users want code sharing standards, not home grown 

## Stage 1:
- Add `sources` option to `createShaderModule`
- Add `source` field to `GPUCompilationMessage`
- Add `import` statement to WGSL.

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
- Not lexical nesting is different from other uses of `{}` in WGSL, and unusual across other languages.
  (but lexical nesting would have other problems, e.g. wouldn't map to files well, spooky)
- reopenable namespaces hurts tools like wgsl-analyzer (need every module to analyze anything).
  and hurts fetching (need to fetch every module, can't fetch only mobile relevant ones), and breaks future visibility.
- library versioning via in-text paths requires rewriting.
- namespaces & modules can be done together, but adds complexity. Many languages have modules but not namespaces, may not need both.
- namespaces + dependencies is halfway between module and namespace designs (module names on the inside, package names on the outside), but both on the outside (module) is better

## Future
- ESM style loader (noted above)

## Later enhancement to consider taking from WESL
- compound import statements (WESL allows `import foo::{a, b::c};` )
- wildcard imports (WESL allows `import foo::*;` with limitations)
- visibility (`private` `public`, for libraries and larger projects)
- `super::` (like `../`)
- 'main' module, acts as root for selecting modules. `enable f16` modules
  not used from `mobile_main`.

## Questions for making the proposal
- for stage 1, allow element imports, or module imports, or both? WESL allows both
- for stage 2 dependencies, dictionaries or objects?

## What have we learned from the WESL experience so far
- Module system was #1 user problem when we surveyed users and open source projects.
  It was WESL's first feature.
  (conditions was #2)
  ref earlier slide where I found a dozen different open source WGSL preprocessors.
- 3 implementations (wesl-rs, wesl-js, wgsl-analyzer), shared test suite, spec.
- evolution added some features for libraries, simplifications to avoid probing for fetching
- the module system doesn't depend on anything else in WESL, but we build other features atop it (visibility, wildcards, re-exporting, etc.)
- Bevy has shipped WESL, Lygia has shipped WESL. But those are new and not much used by their users yet. We've dozens not hundred of users.
- major design challengers still open but growing less likely: ocaml modules instead over `impl` might make namespaces on steroids more tempting. 
  Relative urls for imports (zig, JavaScript) might be interesting for tighter web integration.

## Meta

TBD 
- std formats lead to standard tools. We can't have tools like wgsl-analyzer and wgsl-test or sharable libraries if every developer hacks their own bespoke approach to shader assembly.
- we want to deliver native browser ergonomics with minimal tools, not just use WGSL as an output format. ergonomics isn't about _enabling_. We can already do this with enough tooling. It's about making it easy and universal in the browser.

## Things we might change in current WESL for easier fetching
- require import statements, and limit inline module paths to two segments. (no `lygia::bar::zap();`, first `import lygia::bar;`, then `bar::zap();`)
- require brackets on element imports `import lygia::bar::{zap};`

## Questions for the committee
- `::` vs `.` vs `/` 
  WESL debated in 2024, [notes](https://hackmd.io/ljkByEcnQa2NdNLWed2M6Q). Both work, WESL went with `::`, but would change to match WGSL decision. 
- Same separator for enums? 
- If we like the direction, what additional incubation / learning would we like first?
- do we understand the restrictions on a future fetch mode?
- ought we syntactically or semantically validate every string passed in or only reachable modules?
  (leaving unreachable modules unvalidated is nice for unused `enable f16` module..)

## Cross language survey

TBD

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
- reflection (WIP, pre-propsoal)
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

## Stage 2 alt via package objects 
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
  ```