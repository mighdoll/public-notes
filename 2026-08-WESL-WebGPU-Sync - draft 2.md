WESL related topics to discuss with the WGSL/WebGPU team, with notes about gpuweb issues and F2F.

**Questions**
- What WESL stuff should we discuss at F2F? I propose 3 sessions: WESL overview, experimental features / open questions, and WESL demos (presumably on demo day). 
- What should we make gpuweb issues for? Proposals below (and deferring several of the existing ones).
- What should we discuss separately from the F2F? I propose a few topics we might discuss in committee below.

**TL;DR;** 
- Catch up the with the WGSL committee on the details of our major current WESL designs: modules, visibility, and conditions.
- Discuss WESL design constraints and use cases
- Refresh WESL related gpuweb issues 
## Aligning WESL Design for WebGPU/WGSL

_Goal: Keep WESL designs aligned so that they serve as viable prototypes for WebGPU/WGSL._
#### Overview of WESL
Present an update on what's in WESL and what's changed. Language features, tools, implications for future WebGPU/WGSL, future tools/features on the WESL roadmap, etc. 
- *todo: give an update at the F2F.*
#### Core current WESL features
We'd like the committees feedback on core spec'd WESL features.
- *todo: polish docs to remove some legacy cruft*
- For each:
	- Can this work someday in browser WebGPU? 
	- Is our current syntax/design aligned with WGSL sensibilities? 
- [**Module system**](https://wesl-lang.dev/spec/Imports) and [ImportsDesign](https://wesl-lang.dev/spec/ImportsDesign)
	- Is WESL's current abstract module path approach with `::` separators. Should we instead align with URLs or partial URLs for modules? Or use `.` as a separator? Is ESM the right analogy?
	- *todo: describe how to implement in a browser in design doc*
	- *todo: make gpuweb issue. discuss in WGSL committee?*
- **[Visibility](https://wesl-lang.dev/spec/Visibility)** and [VisibilityDesign](https://wesl-lang.dev/spec/VisibilityDesign)
	- 3 levels, does that seem like the right balance of power vs. complexity?
	- Consider: 'main' module controls host api (vs. any user or library modules may expose entry points, bindings, and overrides)
	- *todo: make new gpuweb issue. discuss in WGSL committee?*
- **[Conditions](https://wesl-lang.dev/spec/ConditionalTranslation)** and [ConditionsDesign](https://wesl-lang.dev/spec/ConditionalTranslationDesign) we have revised this feature a bit since last year. the new features are not earthshaking, but conditional compilation is much requested for writing shaders.
	- *todo: use gpuweb#5140 to continue discussion (discussion can combine with wesl#222 if that ripens). discuss in WGSL committee?*
#### Use Cases
We've been assembling a list of user challenges in hopes of fixing many of them in future features. There's about 25 so far (and we have a backlog of 5-10 to write up). e.g.  [Blend modes from PixiJS](https://github.com/webgpu-tools/wesl-spec/issues/211) [Conditional expressions from Bevy](https://github.com/webgpu-tools/wesl-spec/issues/213) [fn overloading from Lygia](https://github.com/webgpu-tools/wesl-spec/issues/126), [Quaternions](https://github.com/webgpu-tools/wesl-spec/issues/192). I think it might be interesting for the committee to see/discuss a summary, and consider how we might store or cross reference the issues. 
- *todo: make gpuweb issue for how to share use cases.*
- *todo: finish backlog of use case issues prior to F2F, make summary*
- *todo: discuss at WGSL committee?*
#### WESL / WebGPU target usage 
[draft: user/task classes](./task-classes.md)
Some thoughts on balancing power user features with design simplicity.
What design assumptions / use targets do you think about? 
Are the salient differences between WebGPU uses and users that are different from other shader language communities?
- *todo: finish draft for WESL spec. file gpuweb issue and ask for feedback.*  
#### WESL to WebGPU compat goals/requirements 
[draft: Designing WESL for WebGPU compatibility](./webgpu-compat-requirements.md)
Is this what we should do to keep the WESL design aligned for WebGPU/WGSL? 
- *todo: finish draft for WESL spec. file gpuweb issue and ask for feedback.*

## Existing gpuweb Issues

**[#5140 - @if for conditions - preview](https://github.com/gpuweb/gpuweb/issues/5140)** (conditions issue - noted above)

**[#5139 - extend the grammar for more attributes](https://github.com/gpuweb/gpuweb/issues/5139)** suggest we wait until a use case pulls for this - probably reflection will want custom attributes. Or maybe a tool like wgsl-formatter.. 
- *todo: comment on the issue*

**[#5070 - WESL Host-visible names that shadow predeclared names](https://github.com/gpuweb/gpuweb/issues/5070)** Open design problem, not ripe to discuss. 
- *todo: comment on issue*

**[#5138 - alternative to const_assert in functions](https://github.com/gpuweb/gpuweb/issues/5138)** It's a corner case, `const_assert` in functions isn't that popular, low priority.
- *todo: comment on the issue*.

**[#777 - consider namespaces for WGSL](https://github.com/gpuweb/gpuweb/issues/777)** WESL is built around modules like javascript/typescript modules, which map directly to files/urls. Namespaces are pretty low on our priority list, but the feature comes up from time to time. 
- *todo: could add to this issue or make a new one to discuss why modules before namespaces makes sense, but not sure if it's priority* 

**[#4905 - bound variables in packages](https://github.com/gpuweb/gpuweb/issues/4905)** wesl#222 module parameters is one approach to this problem. auto binding structs might be another.
- *todo: nothing now, add comment when wesl#222 or deferred binding structs is ripe*

**[#5456 - # link multiple GPUShaderModules](https://github.com/gpuweb/gpuweb/issues/5456)** The WESL module system we hope shows how this could be solved in future WebGPU/WGSL. Best to let the design get beat up in WESL first before considering baking it into browsers. 
- *todo: add comment referring to WESL?*
## Proposed / experimental WESL features 
These will probably be ready to discuss or demo by Paris. 
* *todo: discuss current issues from the frontier of WESL design at the F2F*  

**[wesl#222 Module parameters with < >](https://github.com/webgpu-tools/wesl-spec/issues/222)** This provides the language-integrated replacement for c++ `#define`; and it'd be a significant upgrade to our current `@if` conditions by merging them with consts and const-eval. This would help solve a number of use cases in existing user code. We're convinced we want something like this, but not settled on which variation. (It's a new proposal, needs more WESL vetting.)
- *todo: make issue and discuss with committee (if it progresses in WESL)*

**do blocks** and **deferred bindings**  WESL-js has two optional features that enable creating pipelines and buffers in _shader code_. Users can configure and dispatch multiple shaders without any host code boilerplate. The code savings are dramatic. `do` blocks look like WGSL/WESL functions, but are executed by an interpreter on the CPU. It's very experimental. 
- *todo: document as WESL experimental feature. demo at F2F*

**other open designs**  generics, contracts, reflection, operator overloading, function overloading, etc. No reviewable designs yet, but we should mention these issues that we're working on. 
* *todo: present summary at F2F*
## Tools and Libraries
We can demo the growing WESL based tool suite at the F2F. (WESL is a strict superset of WGSL, so the tools support WGSL and WESL.)
- **wesl-rs** - wesl implementation for rust
- **wesl-js** - wesl implementation for javascript
- **wgsl-analyzer** - language server for vscode 
- **wgsl-play** - standalone html webgpu player
- **wgsl-edit** - standalone html shader editor
- **wgsl-format** - code formatter
- **wgsl-play.dev** - shadertoy style playground
- **wgsl-test** - test runner for shaders
- **wgsl-studio** - tests and player in vscode

We'll can some community libraries as well: notably Lygia and Bevy.

*todo: present wesl based tools and libraries at F2F*
