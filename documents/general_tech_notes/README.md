# General Tech Notes

## Rust + Wasm notes

### Introduction

- [Rust and WebAssembly documentation](https://rustwasm.github.io/docs.html)
- [Rust and WebAssembly (video)](https://www.youtube.com/watch?v=CMB6AlE1QuI)
- [What is wasm](https://rustwasm.github.io/book/what-is-webassembly.html)
- [Wasm bindgen](https://rustwasm.github.io/wasm-bindgen/)

### Useful crates

- [Crates 1/2](https://rustwasm.github.io/book/reference/crates.html)
- [Crates 2/2](https://rustwasm.github.io/book/reference/tools.html)

### Benchmarking

- [Time Profiling](https://rustwasm.github.io/book/reference/time-profiling.html)

### Optimizing

- [Code Size](https://rustwasm.github.io/book/reference/code-size.html)

### RSA Encrypt/Decrypter

- The functions with macro `#[wasm_bindgen]` are pretty limited with the parameters that receive and the return type
    - To receive a custom type it's necessary that implements `RustWasmAbi`, if don't it won't work.
    - To return a custom type it's necessary that implements ***, if don't it won't work.
- If you want to test something that returns a `Result`, it's necessary that the error type is `JsError`, but this error doesn't implement the `Debug` trait so instead you have to use another function containing the logic of that function and then a wrapper of that function containing the macro `#[wasm_bindgen]` that calls that function and returns the result with the error type `JsError`. There is an [example](https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen/struct.JsError.html#complex-example) in the documentation. 
