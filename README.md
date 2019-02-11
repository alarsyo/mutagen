# Breaking your Rust code for fun & profit

[![Build Status](https://travis-ci.org/llogiq/mutagen.svg?branch=master)](https://travis-ci.org/llogiq/mutagen)
[![Downloads](https://img.shields.io/crates/d/mutagen.svg?style=flat-square)](https://crates.io/crates/mutagen/)
[![Version](https://img.shields.io/crates/v/mutagen.svg?style=flat-square)](https://crates.io/crates/mutagen/)
[![License](https://img.shields.io/crates/l/mutagen.svg?style=flat-square)](https://crates.io/crates/mutagen/)

This is a work in progress mutation testing framework. Not all components are there, those that are there aren't finished, but it's already somewhat usable as of now.

This repo has three components at the moment: The mutagen test runner, a helper library and a procedural macro that mutates your code.

### Mutation Testing

A change (mutation) in the program source code is most likely a bug of some kind. A good test suite can detect changes in the source code by failing (killing the mutant). If the test suite is green even if the program is mutated (the mutant survives), the tests fail to detect this bug.

Mutation testing is a way of evaluating the quality of a test suite, similar to code coverage.
The difference to line or branch coverage is that those measure if the code under test was *executed*, but that says nothing about whether the tests would have caught any error.

### A Word of Warning

Mutagen will change the code you annotate with the `#[mutate]` attribute. This can have dire consequences in some cases. However, functions not annotated with `#[mutate]` will not be altered.

*Do not use `#[mutate]` with unsafe code.* Doing this would very probably break its invariants. So don't run mutagen against modules or functions containing unsafe code under any circumstances.

*Do not use `#[mutate]` for code that can cause damage if buggy*. By corrupting the behavior or sanity checks of some parts of the program, dangerous accidents can happen. For example by overwriting the wrong file or sending credentials to the wrong server.

*Use `mutagen` as `dev-dependency`, unless otherwise necessary.* Compiling `mutagen` and its plugin is time-intensive and library-users should not have to download `mutagen` as a dependency.

*Use `#[mutate]` for tests only.* This is done by always annotating functions or modules with `#[cfg_attr(test, mutate)]` instead, which applies the `#[mutate]` annotation only in `test` mode. If a function is annotated with plain `#[mutate]` in every mode, the mutation-code is baked into the code even when compiled for release versions. However, when using `mutagen` as `dev-dependency`, adding a plain `#[mutate]` attribute will result in compilation errors in non-test mode since the compiler does not find the annotation.

### Using mutagen

You need a nightly `rustc` to compile the plugin. Add the plugin and helper library as a `dev-dependency` to your `Cargo.toml`:

```rust
[dev-dependencies]
mutagen = "0.1.2"
mutagen-plugin = "0.1.2"
```

Now, you can add the plugin to your crate by adding the following to your imports:

```rust
#[cfg(test)]
use mutagen_plugin::mutate;
```

Now you can advise mutagen to mutate any function, method, impl, trait impl or whole module (but *not* the whole crate, this is a restriction of procedural macros for now) by prepending `#[cfg_attr(test, mutate)]`. The repository contains an example that shows how mutagen could be used.

The use of `cfg_attr` ensures the `#[mutate]` attribute will only be active in test mode.

### Running mutagen

Install `cargo-mutagen`, which can be done by running `cargo install cargo-mutagen`. Run `cargo mutagen` on the project under test for a complete mutation test evaluation.

The mutants can also be run manually: `cargo test` will compile code and write the performed mutations to `target/mutagen/mutations.txt`. This file contains ids descriptions of performed mutations.
Then, the environment variable `MUTATION_COUNT` can be used to activate a single mutation as defined by `mutations.txt` file. The environment variable can be set before calling the test suite, i.e. `MUTATION_COUNT=1 cargo test`, `MUTATION_COUNT=2 ..`, etc. For every mutation count at of least one, the test suite should fail

You can run `cargo mutagen -- --coverage` in order to reduce the time it takes to run the mutated code. When running on this mode, it runs the test suite at the beginning of the process and checks which tests are hitting mutated code. Then, for each mutation, instead of running the whole test suite again, it executes only the tests that are affected by the current mutation. This mode is specially useful when the test suite is slow or when the mutated code affects a little part of it.

If you want the development version of `cargo-mutagen`, run `cargo install` in the runner dir of this repository. Running `cargo install --force` might be necessary to overwrite any existing `cargo-mutagen` binary.

### How mutagen Works

1. The attribute `#[mutate]` transforms the source code at compile-time
2. Mutations are introduced at run-time

#### Source code transformations

The attribute `#[mutate]` is a procedural macro that transforms the code on AST-level. Only code annotated with `#[mutate]` gets altered. Known patterns of code (e.g. arithmetic operations, boolean logic, loops, ...) are changed into mutators that behave identically to the original code unless activated. The code transformed code then gets compiled.

#### Run-time selection of mutations

The compiled test suite contains all mutators. However, all tests should succeed by running the test suite via `cargo test` since no mutators are active by default.

A single mutator can be activated by setting the environment variable `MUTATION_COUNT` to a positive number (e.g. `MUTATION_COUNT=1 cargo test`). In this case, the test suite fail, killing the mutant. If it succeeds, the mutant survives.

### Trade-offs

Mutagen is implemented as a procedural macro, which means that only code annotated with `#[mutate]` is considered for mutations. This is a limitation but also a great feature (see warnings above).

This library is designed to be fast. We cannot afford re-compiling the test suite for every mutation. This means that all mutations have to be baked in at compile-time. This means we must avoid mutations that break the code in a way that it no longer compiles.

This project is basically an experiment to see what mutations we can still apply under those constraints.

### Contributing

Issues and PRs welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) on how to help.
