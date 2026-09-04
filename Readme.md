[![https://jappie.me](https://img.shields.io/badge/blog-jappie.me-lightgrey)](https://jappie.me/tag/haskell.html)
[![Github actions build status](https://img.shields.io/github/actions/workflow/status/jappeace/1br/ci.yaml?branch=master)](https://github.com/jappeace/1br/actions)
[![Jappiejappie](https://img.shields.io/badge/discord-jappiejappie-black?logo=discord)](https://discord.gg/Hp4agqy)

> Speed is the essence of war.

The [one billion row challenge](https://github.com/gunnarmorling/1brc):
a billion `name;temperature` lines go in, min/mean/max per weather
station comes out, and the only question is how fast. The official
record is 1.5 seconds (Java compiled ahead-of-time with GraalVM
Native Image, eight server cores; eight of the top ten skipped the
JIT, though a plain OpenJDK JIT entry took fourth at 1.88s). This repo does it in
Haskell in **1.27 seconds** on a laptop, then keeps going, because one
implementation is never enough when you have questions.

## The scoreboard

Six implementations, byte-identical output, one shared test suite.
Numbers from an 8-core Ryzen AI 7 350 (cold single shots, because the
poor thing thermal-throttles if you run it twice):

| implementation | instructions per 100M rows | 1B wall |
|----------------|----------------------------|---------|
| Haskell, plain IO ([src/](src/)) | 17.8B | **1.27s** |
| Haskell, effectful (static) | 17.8B, +0.000% | same as IO |
| Haskell, mtl capability class | 18.5B, +3.8% | same as IO |
| Haskell, effectful (dynamic dispatch) | 32.6B, +83% | 2.3x slower |
| Rust, the control group ([rust/](rust/)) | 11.8B | **1.05s** |
| Haskell in GHCi, object code -O0 | n/a | ~7 min extrapolated |
| Haskell in GHCi, true bytecode | n/a | ~2.1 hours extrapolated |
| MicroHs ([mhs/](mhs/)) | bless its heart | ~10 hours |

## Things we learned so you don't have to

The fast path is mmap-free: workers `pread` chunks into their own
padded buffers, because the kernel copies from page cache faster than
it faults pages in. Everything per-line is SWAR bit tricks in 64-bit
words, the hash table slots are exactly one cache line, and the hot
loop allocates nothing. The full war stories live as `Decision:`
comments in [src/Aggregate.hs](src/Aggregate.hs) and in
[rust/README.md](rust/README.md).

The Rust port exists to price GHC's calling convention: GHC nails ten
registers to the STG machine and spills what doesn't fit, Rust gets
the whole register file, and that's roughly the whole 1.27 vs 1.05
difference. We also emitted the LLVM IR and tried to hand-beat the
compiler. We could not. Nobody has to know how long we tried.

effectful is free. Genuinely, provably free: the static variant is
instruction-identical to plain IO at two billion effect binds. Until
you use *dynamic* dispatch on something tiny, at which point the
`send` round trip (~148 instructions) costs more than Rust spends on
the entire line, and your billion rows take 2.3x longer. mtl spells
the same reinterpretable-effect idea as a typeclass and GHC
specializes it down to +3.8%. Late binding costs exactly when you
bind late. effectful's docs say "when in doubt, use dynamic dispatch",
which is advice about flexibility and it's good advice: swappable
interpreters, mockable tests, code that survives change. Performance
is simply a different axis, and this table adds the datapoint for the
rare case where it dominates: the send round trip is ~148 instructions,
so a file read or a request never notices it and a forty-cycle parsed
line very much does.

GHCi turned out to be two rungs, and measuring them honestly took an
adversarial reviewer: this repo's `.ghci` sets `-fobject-code -O0`,
so a naive `cabal repl` session runs native unoptimized code (about
4s per 10M rows, ~30x slower than -O2, output byte-identical) while
looking like an interpreter. Actual bytecode needs
`-ignore-dot-ghci -fbyte-code -fforce-recomp` and costs about 76s
per 10M rows, roughly 6000x the compiled binary's steady-state
throughput (12.7ms per 10M): every register-resident micro-op
becomes a boxed trip through a dispatch loop. GHCi is the
interpreter half of a JIT with the profile-and-compile half missing,
and that 2500x is the gap the missing half would have to close. Even
so it beats MicroHs (362s per 10M, correct throughout) by about 5x,
since GHCi bytecode at least calls into compiled primops and
libraries while MicroHs interprets combinators all the way down.
Combinator self-optimization, it turns out, does not extend to
register allocation.

## Usage

```
nix-shell
cabal build all
cabal run generate -- 1000000000 measurements.txt
cabal run exe -- measurements.txt
```

## Tests

```
cabal test
```

Every official 1brc sample (rounding, boundaries, emoji station
names, the 10k-key stress case) against every implementation, plus
generator round trips. CI builds and tests all of it via
`nix-build nix/ci.nix`.
