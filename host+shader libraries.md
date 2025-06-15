WESL tooling currently supports packaging WGSL and WESL as libraries in npm and crates.io.

But some potential WebGPU libraries want both shader code and host code (e.g. in Rust or JavaScript or C++). In WebGPU, shaders can't allocate buffers or dispatch shaders - that logic needs to be in host code. So libraries that need to internally allocate buffers or internally dispatch shader kernels need host code as well as shader code. 

Despite needing two languages for execution, there are many gpu tasks with relatively simple interfaces that would make good WebGPU libraries. Examples of host+shader libraries in this class include image processing filters, sorting, prefix scan, reduction, and many more.  `blur()` or `sort()` would be widely useful to the WebGPU community if they could be packaged in a way that's easy to use. A standard interface for simple host+shader WebGPU libraries will help. 

I think we can define a standard approach that library authors and users can use today for host+shader libraries. The primary task I think is to define the rust/typescript interface. Future versions of WGSL and WESL will make things smoother, but library users can be enabled now. 
### Notable requirements:
- applications using a host+shader library need to be able to manage GPU buffers to integrate with other application code and host+shader libraries
- performance needs to be reasonably close to a full custom solution for typical cases. 
	- For some libraries, that'll require some level of 'fusion' to avoid the performance cost of spilling data to slower global storage and multiple shader dispatches between libraries. E.g. map() then reduce() shouldn't have to spill data to global storage in between.
- host+shader libraries should be reconfigurable after use. Switching to a larger or different source buffer shouldn't require destroying everything and starting over.
## Proposed Sketch
### TBD
- are the plans below sufficient to enable efficient and sufficiently flexible host+shader libraries?
- binding groups integration - do host+shader libraries need to integrate binding group indices?
- bindless?
### Basic Approach
WESL has a solution today for publishing shader libraries to crates.io and npmjs.com. See [publishing packages](https://wesl-lang.dev/docs/Publishing-Packages). Shader texts are bundled into rust or javascript structures with some metadata to enable shader linking of imported packages or crates. A host+shader library typically includes a shader bundle as well as some rust or javascript host code. But the shader code may be entirely internal to the library, library users may only see the host api.

Host+shader libraries are published separately in crates.io and npmjs.com, as with shader-only WGSL/WESL libraries. The shader code would normally be the same in both versions, but host code would be written twice, once in Rust and once in JavaScript/TypeScript. In the future perhaps we'll develop a host-language neutral declarative way to specify the host 'glue' code, but that's probably best developed after we've some more experience with the needs of this class of library. For now I'm assuming that library authors would simply manually port their host code to/from Rust and TypeScript, and that host code interfaces would be relatively similar to make porting easy.
### Host code drives

Applications will typically initialize and configure a host+shader library from host code. These libraries are 'in charge' of their own setup and execution. Applications use the `commands()` api provided by the host+shader library to integrate execution library into the application's `GPUCommandEncoder` before the application calls `dispatchWorkgroup()`.

(Note that a shader-only function library integrates into applications differently than this kind of host+shader library. In a shader-only library, the application shader code imports library functions but the application remains fully responsible for setup and execution. In contrast, host+shader libraries take more responsibility for their own setup and execution.)

Host+shader libraries implement the following interface:
```ts
	export interface HostedShader {
		/** Add compute or render passes for this shader to the provided GPUCommandEncoder */
		commands(encoder: GPUCommandEncoder): void;
		
		/** cleanup gpu resources */
		destroy?: () => void;
		
		/** optional name for logging and benchmarking */
		name?: string;
	}
```

Users initialize a hosted shader by passing configuration parameters (all this is in host code), e.g.
```ts
	const sortIn: GPUBuffer = ...;

	const sort = new RadixSort({ 
		input: sortIn, 
		keysOnly: true, 
		elemType: "u32" 
	});
```

Users execute the configured `HostedShader` by plugging it into their application's GPU dispatch loop, e.g.:

```ts
	function render() {
		const cmds: GPUCommandEncoder = ...;

		sort(cmds); // execute gpu sort library in application

		cmds.dispatchWorkgroup(...);
	}
```

Multiple hosted shader libraries (and application code) can be dispatched together and share resources like GPU buffers. 
```ts
const sort = new RadixSort({ input: sortIn, output: sortOut, ..});
const scan = new Scan({ input: sortOut, type: "exclusive", ..})

function render() {
	...
	sort(cmds);
	scan(cmds);
	...
}
```
### Reconfiguration
Applications can modify a `HostedShader` at any time to affect future executions:
```ts
const sort = new RadixSort({ input: bufA, output: bufB, ..});
render();
sort.setInput(bufB);
sort.setOuput(bufA);
render();
```

Internally, its up the `HostedShader` to decide what GPU resources need to be rebuilt when configuration changes.
### Optional Resources
Typically a `HostedShader` should optionally construct GPU resources if they're not provided by the caller. e.g. `scan()` can create its own output GPUBuffer if none is provided:
```ts
interface ScanParams {
	input: GPUBuffer;
	output?: GPUBuffer;
	...
}
```

The `HostedShader` should provide readers for those resources. The reader API should use function syntax with `()` rather than property access syntax to remind the caller that the values may change if the shader is reconfigured.

```ts
const outBuf: GPUBuffer = scan.output();
```

### Manual Fusion
To run additional shader code within the same GPU thread, `HostedShader`s may offer API hooks to call application provided shader functions.

```ts
import w from "./myShader.wgsl"; // reflect 

const reduce = new Reduce({ 
	preMap: w.fn.myMapFn, 
	...
});
```
The goal of manual fusion is to increase performance. In this case of a map followed by a reduce, speed is maximized by running a user provided map function over the input data in the same gpu thread as reduce, without a separate dispatch and without a temporary buffer for the map results. 

To effect this fusion, I'm imagining it would be convenient to use some reflection and linking features we've prototyped in WESL. That would be more maintainable, allow pulling in related functions, and potentially allow build time type checking. But that's the fancy version. A hook can also accept a simple string containing the shader function text:

```ts
const reduce = new Reduce({ 
	preMap: `fn reduce_preMap(elem: i32) -> i32 { return abs(elem); }`, 
	...
});
```
### Dynamic Dependencies between Reconfigured Shaders
When a `HostedShader` reconfigures, some of its internally generated resources may be reconstructed. e.g. in the `sort()` then `scan()` example above, the `sort()` output buffer might be replaced if a new input buffer is provided by the user. If a `HostedShader` depends on a reconfigured value, as `scan()` depends on `sort()`'s output buffer in the example, then the depending `HostedShader` will also need to be reconfigured.

#### Update Dependencies every Frame
`HostedShader` application users should defensively set parameters prior to every dispatch to pick up any changes to dynamic fields, unless the libraries are reactive (see below).
```ts
scan.setInput(sort.output())
render();
```
Accordingly, `HostedShader` implementations should be idempotent if a configuration value is reset to the same value.
#### Simplify with Reactivity
To make implementation easier, `HostedShader` implementors are encouraged to consider using a reactive library like `leptos_reactive` in the rust community or the or tc39 `proposal-signals` polyfill in the JavaScript community. Reactive functionality allow for automatically rebuilding internal GPU resources as required when configuration changes.

For users, reactive libraries relieve application users of the need to defensively set dynamic parameters, as reactive tracking will take care of that automatically. 

To enable a reactivity for dependencies between libraries, `HostedShader` initializers should accept functions as well as raw values. Using functions will enable establishing of reactive links between `HostedShader`s without further user intervention. 
So for the `Scan` example, initialization parameters should accept either raw values or functions returning values:
```ts
interface ScanParams {
	input: GPUBuffer | () => GPUBuffer;
	output?: GPUBuffer | () => GPUBuffer;
	...
}

const sort = new RadixSort({ input: sortIn, ..});
const scan = new Scan({ input: () => sort.output(), type: "exclusive", ..})
```
The user can safely drop the manual per frame update of depencies if the `RadixSort` and `Scan` use the same reactive library. 
### Note on encapsulating shader code in libraries
There are challenges to modularity in any WebGPU library with shader code, even if the library itself doesn't include any host code. Especially when two libraries might exist in the same WebGPU shader module, there's opportunity for conflict. For example, since all WGSL functions live in the same global namespace, two libraries that happen to use the same `fn` name will conflict. WESL solves shader name conflict problems with a mangling scheme. As WESL shaders are bundled into WGSL, global names (and the references to the global names) are made unique.

We're aware of several additional encapsulation challenges related to the host interface, which is also shared among all libraries bundled into the same `GPUShaderModule`.
	- conditions integration - how do shader libraries expose conditions for runtime/build time compilation control by applications? [#]
	- injected consts & overrides - how do shader libraries expose injected consts and pipeline overrides to applications without conflicting with each other?
	- uniforms integration - how do shaders libraries expose configurability through uniform buffers without allocating a separate uniform buffer for each library?

We have discussed possible WESL solutions for these additional problems elsewhere. In the interim, libraries can workaround. e.g. for conditions, prefer `@if(foolib_keyvalue)`  rather than `@if(keyvalue)`, because the shorter `keyvalue` is more likely to conflict with another library or application. 

### WESL Features for Host+Shader Libraries
In addition to features generally useful (like generics), a few feature ideas are particularly relevant for this class of libraries:
- reified references [#51](https://github.com/wgsl-tooling-wg/wesl-spec/issues/51) to specify user code to inject for [[#Manual Fusion]] 
- a way for shaders to import from reified references
- pluggable functions [#133](https://github.com/wgsl-tooling-wg/wesl-spec/issues/133)

