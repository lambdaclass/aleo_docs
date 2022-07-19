# Rust Development of SGX Applications
## ¿What's SGX?

- SGX is an Intel ISA extension with TEEs support (Trusted Execution Environments)
- We can think about as **"isolated" processes** which cover a subset of $\{$data integrity, code integrity, data privacy$\}$.

![](https://i.imgur.com/Lb332Bp.png)

## SGX 101

- Intel's TEE's are called **enclaves**.
- **It's not possible to read nor write the enclave's memory space form outside the enclave.** regardless of the privilege level and CPU mode
- In production, it's not possible to debug enclaves by software nor hardware.
- Entering the enclave via function calls, jumps or stack/register manipulation is not possible. To do so you have to use a specific CPU instruction which also does some safety checks.
- **Enclave's memory is encrypted**, and the key usedchanges on every power cycle. It's stored within the CPU and is not accessible.

## ¿Why Rust?
Intel developed an sdk using c++, so why would i use something else:
- The current sdk is complex (The sdk manual is ~400 pages long)
- rust :heart:
    - Fast
    - Memory Safety
    - "Without" race conditions
    - Nice to cover the loss of some security guarantees the OS gives (memory access restrictions and aslr for example).

## The 2 options to develop SGX applications using Rust

There are two alternatives which have different takes:
- Teaclave SGX SDK: Developed by Baidux Labs, now apache is in charge
- Fortanix EDP: Developed by Fortanix, a company with some experience building sgx solutions (they've been using intel sgx since ~2017)

## Teaclave SGX SDK
![](https://i.imgur.com/rzWrjSw.png)

![](https://i.imgur.com/ixchGF6.png)

## Fortanix SGX EDP
![](https://i.imgur.com/M5pt0Zd.png)

## Pros y Cons.

- Teaclave
    - pros:
        - :heavy_check_mark: Uses Intel's libs, and they're supposed to be the experts on that.
        - :heavy_check_mark: There are simulation libraries which expand the support a bit.
    - cons:
        - :x: Uses Intel's libs, and they're supposed to be the experts on that. This might not be a bad thing by itself, but you could think of this as adding an extra dependency with a centralized entity such as Intel. Which is why in a decentralized environment might not be ideal (debatable).
        - :x: Integrating SGX to an existing system using this sdk is a bit tedious, since you need to restructure your application, use some Makefiles to handle linking the enclave with the application and more.

## Pros y Cons.
- Fortanix
    - pros:
        - :heavy_check_mark: Officially target tier 2 of the Rust compiler. 
        - :heavy_check_mark: Add a few lines to your `Cargo.toml` and you are set.
    - cons:
        - :x: Well, sometimes it's not that easy. Not all crates have support for sgx.
        - :x: since it uses `libstd` it assumes that you have implementations for `time/net/env/thread/process/fs`, which sgx does not entirely support. This will generate runtime panics when used and you won't be getting compilation errors.

## Pending Topics (Blockers)
Looking at the big picture, Fortanix seems like the better option in terms of usability. Also, the integration into the rust environment as a compilation target increases its reliability a bit.
However, i still haven't figured out all Security issues. One of teaclave maintainers says:

    - There is no way to audit usercalls in compile time
    - There were others, but i think those are addressed in https://github.com/apache/tvm/issues/2887#issuecomment-492752317.

Fortanix documentation has some Warnings in important places. For example https://edp.fortanix.com/docs/concepts/rust-std/#stream-networking

## Some Links

- The [Issue](https://github.com/apache/tvm/issues/2887) previously mentioned.
- [Documentation](https://edp.fortanix.com/docs/) is quite short (maybe too short?): But it does seem to explain essentials, with some security tips and how to build a production-ready enclave.
- [Get started](https://www.youtube.com/watch?v=bAYcZ2eAHkA) to develop using teaclave (as long as you have hardware support or simulation mode is working in your workstation).
- Teaclave SDK [Paper](https://dl.acm.org/doi/10.1145/3319535.3354241).
