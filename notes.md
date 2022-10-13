Some notes (and guesses) based on the old commit: https://github.com/rust-lang/rust/pull/15698/files

## Data structures

### Rust-side enum (not yet introduced in that old commit)

`compiler/rustc_target/src/spec/mod.rs`

```rust
pub enum CodeModel {
    Tiny,
    Small,
    Kernel,
    Medium,
    Large,
}
```

### LLVM-side enum

Obsoleted:

```rust
llvm::CodeModelDefault
llvm::CodeModelSmall
llvm::CodeModelKernel
...
```

Now they are grouped within an enum of the following pattern (`compiler/rustc_codegen_llvm/src/llvm/ffi.rs`):

```rust
llvm::CodeModel::{
    Default,
    Small,
    Kernel,
    ...
}
```

* `llvm`: mod under `compiler/rustc_codegen_llvm/src/llvm/`
* I guess this, and the whole file is sort of (manual, auto, or hybrid) `bindgen`ed results from LLVM source. Perhaps `llvm/include/llvm/Support/CodeGen.h`.

### Command line

```rust
sess.opts.cg.code_model
```

* `sess`: one invocation of `rustc` (`compiler/rustc_session/src/session.rs`)
* `opts`: options (`compiler/rustc_session/src/options.rs`)
* `cg`: codegen options (`compiler/rustc_session/src/options.rs`)

```rust
/// Represents the data associated with a compilation
/// session for a single crate.
pub struct Session {
    pub target: Target,
    pub host: Target,
    pub opts: config::Options,
    ...
}
```

```rust
    /// The top-level command-line options struct.
    pub struct Options {
        ...
        cg: CodegenOptions [SUBSTRUCT],
        ...
    }
```

```rust
options! {
    CodegenOptions, CG_OPTIONS, cgopts, "C", "codegen",
    ...
    code_model: Option<CodeModel> = (None, parse_code_model, [TRACKED],
        "choose the code model to use (`rustc --print code-models` for details)"),
    ...
}
```

* The `options!` macro will define a `struct`, named by the first entry, the rest of entries being members.
* Member type: `Option<RustSideEnum>` etc
* Member contents, of pattern (default value, parsing function):
    * `bool = (false, parse_bool`: boolean with a default value
    * `bool = (true, parse_bool`: boolean with a default value
    * `Option<bool> = (None, parse_opt_bool`: boolean without a default value
    * `bool = (false, parse_no_flag`: only flag, without arguments
    * `Option<CodeModel> = (None, parse_code_model`
        * This parsing function will invoke `from_str` implemented for `RustSideEnum` (e.g. `CodeModel`)

Differences than the old code:

* No Rust-side enum, `String` instead, parsing function is thus `parse_string` (command-line string to Rust `String`)

## Run passes

`src/librustc/back/link.rs`, `write` mod, `run_passes` function:

1. Map Rust-side (internal) objects to LLVM-side (foreign) objects
2. Create target machine
3. Create managers
4. Add passes
5. Run managers
6. Codegen-specific 3, 4 and 5

Code in this file mostly goes to `compiler/rustc_codegen_llvm/src/back/write.rs` now

Aside: how LLVM functions are called
* Type 1: `LLVMRust*`
    1. LLVM function `createTargetMachine`
    2. wrapped by `LLVMRustCreateTargetMachine` (`compiler/rustc_llvm/llvm-wrapper/PassWrapper.cpp`)
    3. introduced as a foreign function `LLVMRustCreateTargetMachine` into Rust (`compiler/rustc_codegen_llvm/src/llvm/ffi.rs`)
* Type 2: other `LLVM*`
    1. LLVM function `LLVMCreatePassManager`
    2. introduced as a foreign function `LLVMCreatePassManager` into Rust
