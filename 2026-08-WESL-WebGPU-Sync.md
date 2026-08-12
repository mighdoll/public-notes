- [ ] attach issue links
- [ ] attach links to web site
# Keep WESL features aligned as  WebGPU proposals

#### WESL overview 
- key language features today
- key tools today (wesl-rs, wesl-js, wgsl-analyzer, playground, wgsl-test, wgsl-format, wgsl-play/edit)
- brief sketch of what's on the radar next: module parameters, generics, contracts [TBD]
#### WESL and WebGPU direction
My sense is that most widely useful language 'ergonomic' features, from a module system to reflection to generics are in scope for potential browser implementation. Is that right? Clarity that e.g. modules are in scope for WebGPU/WGSL eventually would help users, e.g. in [gpuweb#5456](https://github.com/gpuweb/gpuweb/issues/5456). 

Our current stance is that any WESL feature that _could_ be upstreamed to browser WebGPU/WGSL _should_ be designed so that it can upstreamed. Let us know if that's wrong for any planned features.
#### WESL to WebGPU compat goals/requirements 
[draft: Designing WESL for WebGPU compatibility](./webgpu-compat-requirements.md)
- Is this what we should do to keep the WESL design aligned for WebGPU/WGSL? 
#### WESL / WebGPU target users 
[draft: user/task classes](./task-classes.md)
- As we consider new features and inevitably add complexity, is this user/task class definition a useful guide? What design assumptions / use targets do you think about for WebGPU? How are WebGPU uses / users different from other shader language communities?
#### Module system
Please review the WESL imports spec and design doc.  [land mathis's polish, plus one more pass to remove old stuff]
- Can this work someday in browser WebGPU? is ESM the right analogy?
- Is our current syntax aligned with WGSL sensibilities? 
- Consider: current abstract module paths vs urls? `::` vs `/`? 
#### Visibility 
Please review the WESL visibility spec and design doc. [we should have a working implementation to try!]
- 3 levels, does that seem like the right balance of power vs. complexity?
- Note that default *package* visibility allows legacy WGSL to be imported in an app, but requires adding `public` annotations to publish WGSL from a library.
- Is our syntax/semantics aligned with WGSL sensibilities?
#### Conditions 
Please review the WESL conditions spec and design doc. [need a cleanup pass for old stuff in the docs]
- [add new block level conditions work]
- [ref module parameters unifying consts and conditions?]
- [ref extending conditions to const expressions?]
- Is our syntax/semantics aligned?
# Requests for WebGPU from WESL
* source map support in the browsers
* ignore unrecognized `@` annotations?
* const_eval variant that's within a function rather than global?
* help spread the word
* help connect us to potential users and contributors
# Use cases for WESL's next design rev
- Here's our list [TBD create use case issue list] - what's missing?
# Future WESL
- syntax question: tagging experimental features with `requires wesl_*`. 
- demo available WESL experiments: [do blocks? wesl-rs eval? reflection? module parameters?]
- [share feature ideas doc for side discussions on generics, etc.]
# Demos
- wgsl-analyzer, playground(s), wgsl-format, wgsl-test, wgsl-edit, etc.
