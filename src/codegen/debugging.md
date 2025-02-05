## Debugging LLVM

> NOTE: If you are looking for info about code generation, please see [this
> chapter][codegen] instead.

[codegen]: ../codegen.md

This section is about debugging compiler bugs in code generation (e.g. why the
compiler generated some piece of code or crashed in LLVM).  LLVM is a big
project on its own that probably needs to have its own debugging document (not
that I could find one). But here are some tips that are important in a rustc
context:

As a general rule, compilers generate lots of information from analyzing code.
Thus, a useful first step is usually to find a minimal example. One way to do
this is to

1. create a new crate that reproduces the issue (e.g. adding whatever crate is
at fault as a dependency, and using it from there)

2. minimize the crate by removing external dependencies; that is, moving
everything relevant to the new crate

3. further minimize the issue by making the code shorter (there are tools that
help with this like `creduce`)

The official compilers (including nightlies) have LLVM assertions disabled,
which means that LLVM assertion failures can show up as compiler crashes (not
ICEs but "real" crashes) and other sorts of weird behavior. If you are
encountering these, it is a good idea to try using a compiler with LLVM
assertions enabled - either an "alt" nightly or a compiler you build yourself
by setting `[llvm] assertions=true` in your config.toml - and see whether
anything turns up.

The rustc build process builds the LLVM tools into
`./build/<host-triple>/llvm/bin`. They can be called directly.

The default rustc compilation pipeline has multiple codegen units, which is
hard to replicate manually and means that LLVM is called multiple times in
parallel.  If you can get away with it (i.e. if it doesn't make your bug
disappear), passing `-C codegen-units=1` to rustc will make debugging easier.

For rustc to generate LLVM IR, you need to pass the `--emit=llvm-ir` flag. If
you are building via cargo, use the `RUSTFLAGS` environment variable (e.g.
`RUSTFLAGS='--emit=llvm-ir'`). This causes rustc to spit out LLVM IR into the
target directory.

`cargo llvm-ir [options] path` spits out the LLVM IR for a particular function
at `path`. (`cargo install cargo-asm` installs `cargo asm` and `cargo
llvm-ir`). `--build-type=debug` emits code for debug builds. There are also
other useful options. Also, debug info in LLVM IR can clutter the output a lot:
`RUSTFLAGS="-C debuginfo=0"` is really useful.

`RUSTFLAGS="-C save-temps"` outputs LLVM bitcode (not the same as IR) at
different stages during compilation, which is sometimes useful. One just needs
to convert the bitcode files to `.ll` files using `llvm-dis` which should be in
the target local compilation of rustc.

If you want to play with the optimization pipeline, you can use the `opt` tool
from `./build/<host-triple>/llvm/bin/` with the LLVM IR emitted by rustc.  Note
that rustc emits different IR depending on whether `-O` is enabled, even
without LLVM's optimizations, so if you want to play with the IR rustc emits,
you should:

```bash
$ rustc +local my-file.rs --emit=llvm-ir -O -C no-prepopulate-passes \
    -C codegen-units=1
$ OPT=./build/$TRIPLE/llvm/bin/opt
$ $OPT -S -O2 < my-file.ll > my
```

If you just want to get the LLVM IR during the LLVM pipeline, to e.g. see which
IR causes an optimization-time assertion to fail, or to see when LLVM performs
a particular optimization, you can pass the rustc flag `-C
llvm-args=-print-after-all`, and possibly add `-C
llvm-args='-filter-print-funcs=EXACT_FUNCTION_NAME` (e.g.  `-C
llvm-args='-filter-print-funcs=_ZN11collections3str21_$LT$impl$u20$str$GT$\
7replace17hbe10ea2e7c809b0bE'`).

That produces a lot of output into standard error, so you'll want to pipe that
to some file. Also, if you are using neither `-filter-print-funcs` nor `-C
codegen-units=1`, then, because the multiple codegen units run in parallel, the
printouts will mix together and you won't be able to read anything.

If you want just the IR for a specific function (say, you want to see why it
causes an assertion or doesn't optimize correctly), you can use `llvm-extract`,
e.g.

```bash
$ ./build/$TRIPLE/llvm/bin/llvm-extract \
    -func='_ZN11collections3str21_$LT$impl$u20$str$GT$7replace17hbe10ea2e7c809b0bE' \
    -S \
    < unextracted.ll \
    > extracted.ll
```

### Filing LLVM bug reports

When filing an LLVM bug report, you will probably want some sort of minimal
working example that demonstrates the problem. The Godbolt compiler explorer is
really helpful for this.

1. Once you have some LLVM IR for the problematic code (see above), you can
create a minimal working example with Godbolt. Go to
[gcc.godbolt.org](https://gcc.godbolt.org).

2. Choose `LLVM-IR` as programming language.

3. Use `llc` to compile the IR to a particular target as is:
    - There are some useful flags: `-mattr` enables target features, `-march=`
      selects the target, `-mcpu=` selects the CPU, etc.
    - Commands like `llc -march=help` output all architectures available, which
      is useful because sometimes the Rust arch names and the LLVM names do not
      match.
    - If you have compiled rustc yourself somewhere, in the target directory
      you have binaries for `llc`, `opt`, etc.

4. If you want to optimize the LLVM-IR, you can use `opt` to see how the LLVM
   optimizations transform it.

5. Once you have a godbolt link demonstrating the issue, it is pretty easy to
   fill in an LLVM bug.
