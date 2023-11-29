# W3C audio group question

Is it possible for a third-party library to add an audioworklet module generated at runtime in a secure way ?

## Context

Astrone ([link](https://www.astrone.app/playground)) is a web based audiometric system. the application might be considered as a medical device (Software As a Medical Device). As such, we need to reduce cybersecurity risks as much as possible.

Faust ([link](https://faust.grame.fr/)) is a functional programming language for sound synthesis and audio processing created at the GRAME-CNCM Research Department. Faust program can be deployed to the web thanks to faustwasm library ([link](https://github.com/grame-cncm/faustwasm)).

## Problem

I tried to implement a faust node in Astrone with the faustwasm library. the node is generated at runtime and registered as an audioworklet module.

The node can be generated in two ways:

- directly from faust program
- or from a wasm module and metadata build by faust toolchains

In both cases, everything works fine with regard to sound processing, BUT it requires to modify the Content Security Policy (CSP):

In the best case (generated node via wasm module), i need to modify my CSP from:

```
script-src: self 'wasm-unsafe-eval';
```

to:

```
script-src: self 'wasm-unsafe-eval' blob:;
```

This modification is a security issue since it opens the application to XSS attacks.

https://www.w3.org/TR/CSP2/:

> As defined above, special URL schemes that refer to specific pieces of unique content, such as "data:", "blob:" and "filesystem:" are excluded from matching a policy of \* and must be explicitly listed. Policy authors should note that the content of such URLs is often derived from a response body or execution in a Document context, which may be unsafe. Especially for the default-src and script-src directives, policy authors should be aware that allowing "data:" URLs is equivalent to unsafe-inline and allowing "blob:" or "filesystem:" URLs is equivalent to unsafe-eval.

the CSP needs `blob:` scheme because faustwasm generates the audioworklet module from a blob.

```ts
const processorCode = `
// DSP name and JSON string for DSP are generated
[...]
(${getFaustAudioWorkletProcessor.toString()})(dependencies, faustData);
`;
const url = URL.createObjectURL(
  new Blob([processorCode], { type: "text/javascript" }) // forbiden
);
await context.audioWorklet.addModule(url);
```

## Possible solution [not implemented yet]:

Generates the audioworklet script on the server side, and hydrates the generated script with the wasm module. This solution is complex, and cannot be implemented (easily) on the library side.

## Note

- Astrone has already a "dynamic" audioworklet node called Custom node. The script is hydrated at runtime via `postMessage` api.
  In the case of faustwasm, things are more complicated because there is parameterDescriptors that is defined in the meta data.
- the 'wasm-unsafe-eval' policy is required to compile and instantiate the wasm module, without enabling JavaScript's eval keyword ([link](https://github.com/WebAssembly/content-security-policy/blob/main/proposals/CSP.md)).
