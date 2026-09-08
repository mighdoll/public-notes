Naming and importing code units is a fine direction for WebGPU/WGSL!

I want to discuss one design choice: where to declare the name of a code unit.

namespaces: 
- units of code are named inside the code e.g. `createShaderModule({ code: "namespace foo { fn bar() {} }" })`

modules:
- units of code are named outside the code e.g. `createShaderModule({ sources: { "foo": "fn bar() {}" } })`

Either way, WGSL gets a way to address code in other code units, e.g. `foo::bar()`, and a `wgsl::` prefix for builtins. The implementations inside WebGPU should be similar overall. Modules need an addition to `createShaderModule()` to accept labeled code strings. Namespaces need new WGSL syntax for the namespace blocks.

They're similar.. but start with modules. Moving the labels to the outside leads to a better place for users: a browser native standard for code sharing, with no external rewriting/linking tools required. Namespaces alone aren't as good.

For bigger projects that want to organize their code:
- The directory tree already gives every file a unique name (`render/util.wgsl` vs `physics/util.wgsl`). With modules that path is the label, attached by whatever tool turns files into strings. With namespaces the shader text itself carries a second copy of the label that users have to read past and keep in sync with the file name. Existing WGSL files go into a `sources` map unmodified; namespaces mean editing every file to wrap it in a namespace block.
- Translator error line numbers point into the concatenated code blob, rather than the files. Source maps (#4844) for shaders in browsers will help, but relying on them makes source maps foundational for everyone, not just for transpiled shader languages.
- Big projects have many pipelines and several device configurations, each wanting a different subset of the code. With concatenation, the user or a tool has to select that subset first. Otherwise, an unused file containing `enable f16;` can make shader creation fail on a device without `shader-f16` enabled. With modules, we can specify that the browser follows references from a chosen root source, parsing and validating only the sources reached that way. Unreferenced modules can remain unparsed, including modules whose only references are disabled by conditional translation. Relatedly, @dneto0 asked how a translator skips syntax it doesn't understand [here](https://github.com/gpuweb/gpuweb/issues/5140#issuecomment-3293396567).

Libraries have more problems with namespaces. A library is just a subsystem maintained by people with whom you can't coordinate naming:
- Namespaces plus concatenation create collisions once two libraries share a dependency.  Say the app uses `libA` and `libB`, and both use `namespace noise`.  Concatenating includes `noise` twice: without reopening that's a duplicate namespace error,  with reopening every function in it is defined twice.  (C++ dedups by wrapping every header in an `#ifndef` guard. But WGSL has no preprocessor.) Someone has to dedup, by hand or with a tool. And simple deduplication can't help when the two copies are different versions of `noise`. Then there's no fix w/o rewriting the text, so a package manager can't just deliver a bundle of code plus its dependencies and hope things work. It needs a rewriting linker to patch things.
- Library names have to be globally unique. Two unrelated libraries both called `util` can't share a blob, so maybe we'd have to start down the awkward Java path towards `namespace com::foo::bar::util`..
- Adding versions to the names ([as suggested above](https://github.com/gpuweb/gpuweb/pull/7310#issuecomment-5530930402)) puts a version tag in every reference, so changing a version means editing app code, and other libraries' code too. But whether Lygia v0.2.2 is compatible with v0.2.3 ought to be a package manager decision, not something to settle in shader source. Better to let the package manager handle versioning and hand the browser the resolved choices.
- Also, if we want to preserve the possibility of ESM style loading (fetching shader code from the internet like browsers load JS/CSS modules), we'll want clean borders between code units, not one blob of code.

Of course, we can have tools convert files, directories and libraries into rewritten namespaced blobs. But then what are we winning? We don't need namespaces to use WGSL as an output format. The win is when users can take what they have and give it directly to the browser, relatively unchanged, in a standard way. 

With code unit names on the outside, and a little more separation between units, the user's side gets simple:
- Files are the units. No wrapper and no name to keep in sync; the directory tree is the hierarchy.
- The browser compiles from the root module and follows references, so each pipeline gets exactly the code it needs, and a module a device can't use is skipped rather than fatal.
- Libraries arrive as sets of files, each with its own view of its dependencies. The package manager decides which dependencies are shared and which aren't, and hands the browser the result, with no library text rewritten. (Later the API can take dependencies too, each with its own sources and dependencies: `createShaderModule({ sources, dependencies: { lygia, noise } })`.)
- Errors name the users' files rather than the code blob.

A browser standard so users can share shader code would be great! Namespaces alone don't give us that, but modules do.

(Combining modules and namespaces is feasible too though it adds the complication of meshing the two. In WESL we think of modules as the bread and butter, and namespaces as a nice possibility to add in the future.) 