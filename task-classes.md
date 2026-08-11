Let's revise our 'example user' description in for [WESL](https://wesl-lang.dev/spec/Designing#who-are-wesl-programmers) 

- [ ] [revise this based on survey info too!]

## Complexity levels

1. **writing a few shaders for my project** - writing your own shaders, consuming libraries along the way. e.g. adding a WebGPU effect on a web site, implementing a class assignment in graphics class, etc. Calling a library function should feel like calling a plain WGSL function, plus occasionally picking a named option (an algorithm, a type). The host-side version should be casual too, e.g., setting a library configuration, filling in a uniform value.

2. **everyday library building** - writing shader code meant for reuse: shared utility files inside an app, or a small published library. Writing one function body that works across several types; implementing an interface that someone else defined, e.g. a BlendMode.

3. **architecture** - building a substantial library or an engine integration: defining the interfaces and extension points that others implement, structuring a multi-file library, curating what applications and host code can configure.

Level 1 tasks and users should stay blissfully ignorant of more specialized / expert concerns.
Making a fun little library shouldn't require becoming an architect.
Error messages shouldn't drag casual users into more expert concerns, etc.

## User background
### General programming knowledge 
Our programmers will likely know JavaScript/TypeScript or Rust, though not both. They’ll also have been trained in at least one other popular language like Python, Java, or C++. Overall, we can expect that our programmers will know general concepts available in most mainstream languages, but we can’t count on them to know the details from any particular language other than WGSL.
### Syntax from Rust and TypeScript?
Avoid trying to imitate a TypeScript or Rust language feature syntactically unless it can fully match the source language semantics. If it _looks just like_ a Rust or TypeScript feature, programmers will expect it to _work like_ the source language. But fully matching the source language will often be impossible in WebGPU, or undesirable for complexity reasons. 

Aim for simple instead of imitative.
