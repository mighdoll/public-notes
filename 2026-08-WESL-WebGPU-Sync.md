- [ ] attach issue links
- [ ] attach links to web site
# Keep WESL aligned as a viable set of WebGPU proposals

### WESL overview
- key language features today
- key tools today (wesl-rs, wesl-js, wgsl-analyzer, playground, wgsl-test, wgsl-format, wgsl-play/edit)
- brief sketch of what's on the radar next: module parameters, generics, contracts [TBD]
#### WESL to WebGPU compat goals/requirements 
[draft: Designing WESL for WebGPU compatibility](./webgpu-compat-requirements.md)
* Our stance is that any feature that _could_ be upstreamed to browser WebGPU/WGSL _should_ be designed so that it can upstreamed. Is that right? 
- Does this keep WESL design/users aligned? 
#### WESL / WebGPU target users 
[draft: user/task classes](./task-classes.md)
- As we consider new features (and inevitably add complexity) is this user/task class definition first as useful way to guide design choices? 
- How are WebGPU uses / users different from other shader language communities?
- What design assumptions / targets to WebGPU committees use today?
#### Module system
Review the WESL imports spec and design doc.  [land mathis's polish, plus one more pass to remove old stuff]
- Can this work someday in browser WebGPU? is ESM the right analogy?
- How should we we think about resource loading? can it be parser driven? [probably an issue prior to F2F]
- Is our keyword syntax aligned with WGSL sensibilities? 
- Balancing cross ecosystem library systems vs user convenience with publisher opt in wildcard support..
- Direct mapping of abstract modules paths to url/file paths..
- Note inline module path references, not just import statements
- Consider: current abstract module paths vs urls? `::` vs `/`? 
#### Visibility 
Review the WESL visibility spec and design doc. [we should have a working implementation (or 2 or 3) to try!]
- 3 levels, does that seem like the right balance of power vs. complexity?
- - 'main' module controls host visibility
- Default *package* visibility allows legacy WGSL to be imported in an app, but requires adding `public` annotations to publish WGSL from a library..
#### Conditions 
Review the WESL conditions spec and design doc. [need a cleanup pass for old stuff in the docs]
- [add new block level conditions work]
- [ref module parameters unifying consts and conditions?]
- [ref extending conditions to const expressions?]
# Requests for WebGPU from WESL
* source map support in the browsers
* flexible `@` annotation syntax?
* const_eval variant that's within a function rather than global?
* help spread the word
* help connect us to potential users and contributors
# Use cases for next major design
- Here's our list [TBD create issue list], what's missing?
# Future WESL
- syntax question: tagging experimental features with `requires wesl_*`. 
- current WESL experiments: do blocks? wesl-rs eval? reflection? module parameters? 
- generics ideas? [probably not a discussion, unless there's specific things we want feedback on - they don't want open ended discussions] 
- Feature ideas doc for side discussions?
# Demos
- wgsl-analyzer, playground(s), wgsl-format, wgsl-test, wgsl-edit, etc.
