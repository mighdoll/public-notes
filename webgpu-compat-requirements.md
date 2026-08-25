
WESL is intended to be both a practical extension of WGSL today and a plausible
future version of WGSL.

1. **Preserve valid WGSL.** Every valid WGSL program must remain valid, with the
   same parse and meaning. All of the existing WGSL CTS must-accept cases must pass.
   It's ok for some previously invalid programs to become valid, but those
   cases should be identified explicitly.

2. **Preserve WebGPU Compatibility.** WESL can propose features with associated WebGPU
   host code API extensions, but existing APIs must continue to work.

3. **Today's valid WGSL identifiers stay identifiers.** An unconditional keyword
   must use a word WGSL already reserves. An unreserved word may only have a
   contextual meaning where the grammar can distinguish it from an ordinary
   identifier (WGSL calls these context-dependent names); it must remain usable
   as an identifier everywhere WGSL currently permits it. New punctuation must
   satisfy the same compatibility rule.

4. **Design for future WGSL.** Attributes, directives, builtins, and
   other named language features occupy WGSL's future design space. Propose features
   as potential WGSL syntax, not as a private WESL namespace. Design for
   a future with no separate WESL mode.

5. **Keep the complete grammar LALR(1).** The grammar, including a proposed
   feature, must be parseable with one token of lookahead after WGSL's normative
   template-list token transformation. Parsing must not depend on types,
   resolved names, imports, or other semantic information.

6. **Resolve one URL without probing.** URL-based module loading must map each
   module path to one URL.
   (A browser resource loader is not aware of package/library boundaries; it
   loads modules one at a time.)
   A failure to resolve or fetch that URL is an error; the
   loader must not try alternate extensions, directories, or fallback URLs.

7. **Explain errors without showing lowered WGSL.** Diagnostics must be
   expressible in terms of what the author wrote: their file, their names, their
   call chain. (Lowered source varies by implementation, and especially across browsers.)