Users might import modules with sometimes different module parameters.
What should be shared semantically between the concrete module variants? 

*TL;DR: types should shared unless they actually differ and var state should be unique per module parameterization.*

---
## Types across concrete modules with different module params 

First, looking at types:

```wgsl
/// bar.wesl
module <const DEBUG = false>

struct Bar {
  b: u32,          // Bar never mentions DEBUG
}

@if(DEBUG)
fn check(b: Bar) { /* does some checking */ }
@else
fn check(b: Bar) { }
```

```wgsl
/// a.wesl
import package::bar::Bar; // imports bar.wesl (DEBUG false)

fn aFn(b: Bar) { ... }
```

```wgsl
/// b.wesl
import package::bar<DEBUG = true>::{Bar, check};
import super::a::aFn;

fn m() {
  let b = Bar(1);
  aFn(b);      // works: a's Bar and b's Bar are the same type
  check(b);
}
```

This example shows that for types, we want to share across module concretizations (to use @k2d222's term). 
WGSL types are nominal, and so we want types to be sharable when possible, otherwise
we'll see problems like `aFn` where types are oddly incompatible. 
To see if a type is shared, we look at the module parameters that it depends on (transitively),
same parameters = same type across the concrete modules.
`Bar` never uses `DEBUG`, so there's one `Bar` type no matter how many parameterizations of `bar.wesl` are in the link, and `aFn(b)` type checks. 

Some types can't be shared. The module parameter might dictate that we get a new type.
A struct that does use a module parameter forks per parameterization. That's required of course,
its layout is different: `struct Lights { l: array<u32, NUM_LIGHTS> }` must be a new type for
every value of NUM_LIGHTS.


## Vars 
Okay, enough types, how 'bout variables?
If there's a `var` in a parameterized WESL module, how many `var`s do we emit to WGSL?

### concrete modules with different module params 

```wgsl
/// rng.wesl
module <const SEED = 1>

var<private> state: u32;    // declaration never mentions SEED...

fn init() { state = hash(SEED); }    // ...but the contents sure depend on it
fn next() -> u32 { state = lcg(state); return state; }
```

```wgsl
import rng<SEED = 1> as r1;
import rng<SEED = 2> as r2;    // two independent streams
```

Each concrete module with different parameterization should emit a different `var`. 

This example suggests too that we want to fork all of the module state, not try to trace parameters to individual declarations. `state`'s declaration never mentions `SEED`. 
The dependency is in `init()`'s body. So tracing would have to follow writes through function bodies, and maybe know which functions each importer actually calls. 
And even if tracing worked perfectly, it would be fragile: add one write somewhere and a var silently forks for every user.

### concrete modules with matching module params 

The previous example showed two import statements with different module parameters.
But what if two import statements have the same module parameters? 
Do we fork for every unique set of module parameters? Or fork at every import regardless?

```wgsl
/// lights.wesl - per-invocation light list
module <const MAX = 8>

struct Light { pos: vec3f, color: vec3f }

var<private> list: array<Light, MAX>;
var<private> count: u32;

fn push(l: Light) { list[count] = l; count += 1; }
fn get(i: u32) -> Light { return list[i]; }
```

```wgsl
/// cull.wesl - one library: fills the light list
import lights as l;
fn cullLights(...) { ... l::push(light); ... }

/// pbr.wesl - another library: reads the light list
import lights as l;
fn shade(...) -> vec3f { ... let light = l::get(i); ... }
```

I think this example shows we should emit a new set of `var`s for each unique module parameterization,
not new vars for every import statement. 
I imagine we'd const-eval and allow defaults for the parameters, so `import lights`, `import lights<MAX = 8>`, and `import lights<MAX = 4 + 4>` are equivalent.
So there's one list of lights, and `cull` fills the same list that `pbr` reads. 
Modules with defaulted parameters are handled consistently with modules with no parameters.

### Authors can share more or less if they want

With those rules for when to share vars, there's still room for author control:

- State is unique per module parameterization, but if an author wants to share state across concrete modules,
  they can do so by hoisting the var into another module w/o the module parameter.
- State is shared per module-parameterization set, but if an author wants to enforce unique state per importer, 
  they have two options: pass in the state via a pointer; or add an additional module parameter
  `rng<STREAM = 47>` vs `rng<STREAM = 19>`.

## Module parameters designs in other languages

Sanity checking this design sketch vs other languages..
- _types unique by used parameters_: 
  the sketch is pretty close to Zig, where struct identity keys on the parameters captured by the comptime declaration. 
  The sketch is finer grained than C++ or D templates, which fork types on unused parameters too. 
  But a template wraps one struct where a module header covers a whole file. 
  OCaml functors are analogous to templates, AFAICT.
  Anyway, I don't see much upside in emitting more distinct types than strictly necessary. 
- _vars unique per parameter set_: same as C++/D template statics (one static per instantiation, program-wide). 
  Zig is finer grained due to their comptime approach, the SEED would be accidentally shared unless you write it differently. (Or maybe our parameterized modules are each one comptime block..)
  Generic statics have been requested for Rust, but it's hard with separate compilation and dyn.