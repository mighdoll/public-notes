Some WebGPU libraries will include both shader code and host code (e.g. in Rust or JavaScript or C++). In WebGPU, shaders can't allocate buffers or dispatch shaders - that logic needs to be in host code. So libraries that need to internally allocate buffers or internally dispatch shader kernels need host code as well as shader code. Despite needing two languages for execution, there are many gpu tasks with relatively simple modular interfaces that would make good libraries (and are available as libraries in other GPU communities). Examples of host+shader libraries in this class include image processing filters, sorting, prefix scan, reduction, and many more.  `blur()` or `sort()` would be widely useful to the WebGPU community if they could be packaged in a way that's easy to use. A standard interface for simple host+shader WebGPU libraries will help. 

I think we can make an interface that library authors and users can use today. Future versions of WGSL and WESL will make things easier for library authors, but library users will be largely unaware of improvements behind the scenes. 
### TBD
- is this sufficient to use/encapsulate key apis
- binding groups integration - do host+shader libraries need to integrate binding group indices?
- bindless?
### Notable requirements:
- applications using a host+shader library need to be able to manage GPU buffers to integrate with other application code and host+shader libraries
- performance needs to be reasonably close to a full custom solution for typical cases. 
	- For some libraries, that'll require some level of 'fusion' to avoid the performance cost of spilling data to slower global storage and multiple shader dispatches between libraries. E.g. map() then reduce() shouldn't have to spill data to global storage in between.
- host+shader libraries should be reconfigurable after use. Switching to a larger or different source buffer shouldn't require destroying everything and starting over.
## Proposed
### Basic Approach
Host+shader libraries would be published separately in crates.io and npmjs.com, as with shader-only WGSL/WESL libraries. The shader code would be the same in both versions, but host code would be written twice, once in Rust and once in JavaScript/TypeScript. In the future I hope we might find a language neutral declarative way to specify the host 'glue' code, but that's probably best developed after we've some more experience with the needs of this class of library. For now I'm assuming that library authors would simply manually port their host code to/from Rust and TypeScript, and that host code interfaces would be relatively similar to make that easy.
### Example use

Applications will typically initialize and configure a host+shader library from host code. Applications use the `commands()` api provided by the library to integrate execution library into the application's `GPUCommandEncoder` before the application calls `dispatchWorkgroup()`.

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

Users initialize a hosted shader by passing configuration parameters in host code, e.g.
```ts
	const sortIn: GPUBuffer = ...;

	const sort = new RadixSort({ 
		input: sortIn, 
		keysOnly: true, 
		elemType: "u32" 
	});
```

Users execute the configured hosted shader by plugging it into their application's GPU dispatch:

```ts
	function render() {
		const cmds: GPUCommandEncoder = ...;

		sort(cmds); // execute gpu sort library in application

		cmds.dispatchWorkgroup(...);
	}
```

Multiple hosted shader libraries (and application code) can be dispatched together and share GPU buffers. 
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
const outBuf: GPUBuffer = scan.getOutput();
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
The goal is to increase performance. In this case of a map followed by a reduce, speed would be maximized by running a user provided map function over the input data without a separate dispatch and without a temporary buffer for the map results. 

To effect this fusion, I'm imagining it would be convenient to use some reflection and linking features we've prototyped in WESL. That would be more maintainable and allow build time type checking. But that's fancy. `preMap` could also accept a simple string containing the shader function:

```ts
const reduce = new Reduce({ 
	preMap: `fn reduce_preMap(elem: i32) -> i32 { return abs(elem); }`, 
	...
});
```
### Dynamic Dependencies between Reconfigured Shaders
When a `HostedShader` reconfigures, some of its internally generated resources may be reconstructed. e.g. in the `sort()` then `scan()` example above, the `sort()` output buffer might be replaced if a new input buffer is provided by the user. If a `HostedShader` depends on a reconfigured value, as `scan()` depends on `sort()`'s output buffer in the example, then the depending `HostedShader` will also need to be reconfigured.

`HostedShader` application users should defensively set parameters prior to every dispatch to pick up any changes to dynamic fields.
```ts
scan.setInput(sort.output())
render();
```
`HostedShader` implementations should be idempotent if a configuration is reset to the same value.

To make managing these dynamic dependencies easier, `HostedShader` implementors are encouraged to consider using a reactive library like `leptos_reactive` in the rust community or the or tc39 `proposal-signals` polyfill in the JavaScript community. Reactive libraries are convenient for internal implementations to track dependencies between configured values and GPU resources. If reactive libraries are adopted, application users won't need to defensively set dynamic parameters, reactive tracking will take care of that automatically. 

To enable a future reactive world, `HostedShader` initializers should accept functions as well as raw values. Using functions will enable establishing of reactive links between `HostedShader`s without further user intervention. i.e. users won't need to defensively reconfigure at every dispatch.

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
And the user can safely drop the manual reconfiguration if the `RadixSort` and `Scan` use the same reactive library. 
### Note on encapsulating shader code in libraries
There are challenges to modularity in any WebGPU library with shader code, even if the library itself doesn't include any host code. Especially when two libraries might exist in the same WebGPU shader module, there's opportunity for conflict. For example, since all WGSL functions live in the same global namespace, two libraries that happen to use the same `fn` name will conflict. WESL solves shader name conflict problems with a mangling scheme. As WESL shaders are bundled into WGSL, global names (and the references to the global names) are made unique.

We're aware of several additional encapsulation challenges related to the host interface, which is also shared among all libraries bundled into the same `GPUShaderModule`.
	- conditions integration - how do shader libraries expose conditions for runtime/build time compilation control by applications?
	- injected consts & overrides - how do shader libraries expose injected consts and pipeline overrides to applications without conflicting with each other?
	- uniforms integration - how do shaders libraries expose configurability through uniform buffers without allocating a separate uniform buffer for each library?

We have discussed promising WESL solutions for these additional problems elsewhere. In the interim, libraries can workaround. e.g. for conditions, prefer `@if(foolib_keyvalue)`  rather than `@if(keyvalue)`, because the shorter `keyvalue` is more likely to conflict with another library or application. 


