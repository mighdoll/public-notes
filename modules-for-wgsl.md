- update imports.md for WGSL committee readers
- write reply on 7310
- propose modules for WGSL

----
Benefits of a module system: split code into reusable pieces
- ergonomics for users that split shaders across files
  - trivial mapping to filesystems (or urls)
  - no user side transpilation tools required
  - errors are trivially mapped back to sources (even w/o embedded source maps)
- ergonomics for code sharing via libraries
  - code can be 'position independent', sharable w/o editing
  - library scoping / versioning control stays with package manager, not the code. diamond dependencies, multi-version all work.
  - browser standard for modules is key primitive for external library standard
- ergonomics for access to overloaded wgsl functions 
- enables optional futures
  - conditionally unparsed modules
  - esm style fetching of resources (js library for now)

Future stuff
- wildcard imports vs named module imports (mention?)
- visibility (skip?)
- conditions (skip?)

-----

7310 reply
> I think this idea would make a great proposal. If we have multiple ways to do the similar thing it would be nice to see them written up so we can discuss them together. It lets us make a better decision on if we want both, neither or either of the proposals. So, please, write up a module proposal to discuss how it would work.
> 
> Answering the last few bullet points first:
> 
> * I don't think we want files as a unit in WGSL. We don't know about files, we get a data string. If you want to write your entire WGSL source into a single file with separation of the code in that file, you should be able to do so.

The overwhelming case in practice of course is that users with code organizational needs split their code across files.
As we design ergonomic features into WebGPU, I think that practical user workflows are in scope for us to consider. 
(Even though the browser API is just strings, never the fileystem directly)

> * What happens if your libraries require different versions of their dependencies? Do you import it multiple times?

The common case is two different libraries requiring different versions of a third library (or an app and a library). The package manager scopes the dependencies to each and passes those dependencies to the API.

For the unusual case where a single app or library directly requires two versions of library, the user would give one a different name via package manager and writes their import/using statements with two names.

> 
> I tried to answer each of your points below, giving a named heading to, what I think, were the main points.
> 
> ## Project organization
> I don't think we want to prescribe how a project organizes its source. It may use multiple files in a directory hierarchy. It may use a flat folder, it may use a single file. All of that, I think, are a higher level then WGSL and don't really affect how either of these systems work.

The mapping of the filesystem to module or namespace strings is handled outside the browser. It isn't prescribed by the browser,
though of course our API tool choices will influence tools.

A trivial mapping tool works really well for modules. 
But with namespaces alone, I think a robust mapping tool will need to do some transpilation/rewriting.

> 
> Having the namespace in the file [...] also allows multiple namespaces in a single file.

True! We've often been tempted in WESL, but so far not fully convinced that the additional complexity is worth the additional convenience. Thus far we've deferred while we see how it might interact with other future features.

> 
> ## Error mapping
> I think you have an error issue either way, but it's a little simpler with modules. In both cases you just have a blob of code, the module name doesn't necessarily have to map back to a given file either. It's a little simpler in the module case in that your limiting the scope to that named thing, but that named thing could have come from anywhere as well.

Yep, in the module world the line numbers and column positions would be perfect already, and the remaining gap would be translating `foo::bar` to `./foo/bar.wgsl`.

> 
> ## Multifile handling
> For big projects, I'd assume they already have a bundler or a rollup or some other process which is running over their source code, combining it and checking it in various ways. How they put the code together to feed to the browser is more interesting then what they ingest as they'll be doing some kind of modification on the input.

There's a difference between a tool that collects the sources into a string or set of strings versus a tool that rewrites the sources. But the goal of native ergonomics would be to minimize the need for sophisticated external tools.

> 
> I'm not sure if we'd want to have "unparsed modules" as you then get spooky action of, I add this `foo::bar` call and suddenly I get compile errors in an unrelated source file. Having them parsed when you call createShaderModule allows us to know that they're valid and usable by the time we go to create the pipeline.

Conditional compilation is one of the top user needed features we found in extant code when got started with WESL.
If WGSL gets some form of conditional compilation (WESL's `@if`, or some variant of `#ifdef` from other preprocessors),
we might want conditionally unparsed modules:
- some modules will be wholly unused at runtime. Users don't tend to think about whether the MOBILE module is fully configured when they're using the desktop entry module and its dependencies.
- an app or library can include a module for a new experimental WGSL feature that's only implemented in one experimental browser, and conditionally ignore it elswehere.

But that's a double maybe (maybe WGSL decides to adopt conditions, maybe WGSL decides to skip unused modules), so
even if I guess that both are likely, maybe it doesn't add up to a very strong argument :-). 

The analogy is to setting an `override`, some things might break after you set runtime variables, or runtime conditions.

> 
> ## Namespace collisions
> I'm not sure I understand your example. If you have `libA` and `libB` and both have a submodule called `noise` then I'd assume you'd have `libA::noise` and `libB::noise` which are both addressable as the fully qualified name. The libraries typically would wrap their code in their top level namespace. (Similar to how everything in WGSL is in the `wgsl` namespace.)
> 
> If the suggestion is they both do a `using blah` which pulls in `noise`, then yes that's an issue, but also you probably don't want to do a `using` call in a library just in general.

[TBD]

> 
> ## Naming Uniqueness
> I don't think name uniqueness is an issue. Packages in Cargo or on NPM already have to have unique names. Projects typically have unique names. Just give it a top level namespace of the project name. You don't have to go fully to the `com.foo.bar.baz` Java style. Just the top level library name.

[some ecosystems w/o multiple package managers (js, c++). 
 js world often prefixes with the package manager to avoid, also we aim to have libraries work cross platform,
 name conflicts across pkg managers are possible. To swithc a library, namespaces would require renaming inside the files to point to a deconflicted name. modules with scoped dependencies require no rewriting]

> 
> ## Versioning
> This is the case that runs counter to what I said about using in a library. In this case, I think you would have a `using`. You'd have something like:
> 
> ```wgsl
> namespace foo {
>   namespace v1 { fn a() -> i32 }
>   namespace v2 { fn a() -> f32 }
> 
>   using v2;
> }
> ```
> 
> So, the current version gets a using so if you just do `foo::a` you get the current one. If you're on V1 and require that `i32` return then you can do `foo::v1::a` to access the original version. For anyone just accessing it as `foo::a`, no changes needed on upgrade only if you care about the backwards compatible version.

[Old package is using v1 w/o a prefix, fixing requires rewriting the source, maybe the source in a library you don't control]

> 
> ## ESM Loading
> I think that this is done at a higher level than WGSL. A library in JS would be a better place to handle ESM style loading and feed code blobs into the source code.

For now, absolutely. But shouldn't we design around the option in the future? 
The js experience suggests we'll eventually be pressed for deeper integration with web urls and community libraries, and that retrofit is painful w/o forethought in the design.

(One detail to think about later is how complicated that JS library would need to be. e.g. does it require parsing or name binding to know what to fetch?)