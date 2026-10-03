# 1br on THC

[THC](https://github.com/ekmett/thc), the Turbo Haskell Compiler, takes
the Core that GHC produces after optimisation and runs it on
Truffle/GraalVM, where Graal partially evaluates the interpreter into
machine code for whatever runs hot
([announcement](https://comonad.com/reader/2026/turbo-haskell/)). It is
the missing half of GHCi this repository's Readme talks about: an
interpreter that profiles itself and compiles what it finds. So the
question is how far that half gets on the same unmodified
[src/Aggregate.hs](../src/Aggregate.hs).

## Results

Same laptop as the scoreboard (Ryzen AI 7 350, 8 cores, 16 threads),
GHC 9.14.1 as the frontend for every THC row, THC at revision
`0ae57cbf`. The native and THC rows were measured on 3 October 2026 on
an idle host. The GHCi rows and the GHC 9.14.1 native row are from 2
October, when other containers kept the load average around 6 to 9.
Every number below produced byte-identical output.

| implementation | 10M rows | 100M rows | 1B rows |
|----------------|----------|-----------|---------|
| native -O2, GHC 9.12.2 (the repository's pin) | 0.040 s | 0.151 s | 1.32 s |
| native -O2, GHC 9.14.1 | 0.040 s | 0.149 s | |
| THC with its defaults | 37.7 s | 243 s | ~39 min extrapolated |
| THC, graph budget 400 000 and OSR off | 21.0 s | 39.4 s | 2.6 to 2.8 min |
| the same plus the word-read prototype | | 36.9 s | 1.8 to 1.9 min |
| GHCi bytecode, 16 capabilities | 69.8 s | | ~1.9 hours extrapolated |
| GHCi bytecode, 1 capability | 124.5 s | | ~3.5 hours extrapolated |

THC rows are wall time of the whole process: the mean of three runs at
10M and for the tuned rows at 100M, one run for the default at 100M, two
or three runs at a billion. Starting the JVM and loading the Core of base,
bytestring, containers and friends costs 10.8 s on its own (the same
launch on a 20-row file), so the default's extrapolation to a billion is
that start-up plus ten times the rest of the 100M run. The tuned row
needs a runtime that reads the graph budget from a system property
([nix/thc-runtime-tunable.nix](nix/thc-runtime-tunable.nix)), because
THC's launcher hardcodes it; turning OSR off is a plain Truffle option.
The update from the previous pin, `6610dc01`, moved neither default
number beyond noise (38.2 s and 235 s there); the tuned row was only
measured on `0ae57cbf`. GHCi rows are the `:set +s` time of `Aggregate.main` with
`-ignore-dot-ghci -fbyte-code -fforce-recomp`, which excludes loading
the modules. With GHC 9.12.2 the single-capability run took 114.0 s
there. The Readme's 75.8 s does not record its capability count, and it
sits between these 1- and 16-capability figures.

With its defaults THC would finish a billion rows about 3x sooner than
GHCi on equal cores; tuned, it finishes about 40x sooner, and about
120x later than native GHC. The default's win is wall time, not
efficiency: it used 3 196 s of user CPU time on 100M rows, where
single-capability GHCi, at 124.5 s per 10M on one core, would need
about 1 245. Tuned, THC used 422 s.

## Why it is slow

With the defaults, the lambda that walks a chunk exceeds Graal's
graph-size budget, which THC's launcher fixes at 100 000. THC's graph
recovery then replaces the lambda with a version split into smaller
call targets, and the redirect from the old version deoptimizes
compiled code on most loop iterations: 4.2 million deoptimizations per
10M rows, roughly half of the CPU time. A budget of 400 000 lets the
lambda compile whole, until the failed install of its
on-stack-replacement variant makes THC throw that compilation away again. That
is why a raised budget alone only reached 181 s at 100M; with OSR off
the compilation stays. What remains is start-up, an 8.4 s compile of
the lambda, and locking on every memory read in THC's `Addr#`
implementation, which a 17-line prototype patch cuts from a billion
rows' 161 s to 109 s. [PERFORMANCE.md](PERFORMANCE.md) has the
measurements, the instruments, and the candidate fixes with an estimate
of how hard each is.

## Reproduce

Building GHC with complete Core takes about half an hour; everything
else is minutes.

```sh
thc/build-ghc.sh ~/thc-toolchain              # GHC 9.14.1, complete Core
thc/acquire.sh ~/thc-toolchain ~/thc-checkout # THC driver + 1br's Core
~/thc-toolchain/bin/onebr-thc measurements.txt
ONEBR_THC_BIN=~/thc-toolchain/bin/onebr-thc cabal test

# the tuned configuration
ONEBR_THC_RUNTIME=$(nix-build thc/nix/thc-runtime-tunable.nix)/bin/thc \
JAVA_OPTS="-Dthc.tune.MaximumGraalGraphSize=400000 -Dpolyglot.engine.OSR=false" \
  ~/thc-toolchain/bin/onebr-thc measurements.txt
```

The test suite then runs every official sample and a generated file
through THC as well; all 13 cases pass. CI does not, since it would
have to build GHC from source.

Getting here took these changes, recorded as `Decision:` comments
where they live:

- THC's JVM runtime is a nix build ([nix/thc-runtime.nix](nix/thc-runtime.nix)),
  Gradle's Maven downloads locked through `gradle.fetchDeps`.
- THC validates the configured GHC source tree it regenerates foreign
  declarations from, so nixpkgs' GHC needed three changes
  ([nix/ghc-complete-core.nix](nix/ghc-complete-core.nix)): hadrian
  built by 9.14.1 with Cabal 3.16, the perf flavour instead of release,
  and no `--with-gmp-includes`.
- primitive is built as a local package
  ([cabal.project.thc](../cabal.project.thc)), because THC compiles C
  adapters for its cbits in a store package's temporary unpack
  directory after Cabal has deleted it.
