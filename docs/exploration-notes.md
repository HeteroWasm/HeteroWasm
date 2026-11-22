# Exploration notes: what we tried, and why it did not pay

This repository grows capability first and measures second. Several directions were
implemented far enough to run against real Draco and Basis corpora and then closed
when the numbers did not justify the machinery. This note records them, because a
negative result with data is part of the design record.

## How a direction is judged

Every direction starts with a measurement, not an implementation: how many loops hit
the shape in the real corpora, how many of those the legality stage would ever
release, and what the transfer cost would be. A direction is closed when that ceiling
is below roughly five percent of the candidate pool, or when the real workload never
executes it.

## Closed directions

### Gather with statically bounded indices

`C[i] = A[B[i] & mask]` is the only sound subset of indirect indexing without a
per-element GPU bounds check. On the real corpora it does not exist: of 60
gather-shaped loops in `draco_decoder` and 18 in `basis_transcoder`, zero combine a
mask with an affine index load. The non-affine accesses are pointer chasing instead,
so no emitter was written.

### Loop-invariant loads as uniform fields

An in-memory load whose address does not depend on the induction variable is already
handled: the scalar evolution treats it as a symbolic constant, so addresses stay
affine. 381 loops in `draco_decoder` and 21 in `basis_transcoder` contain such bases,
but every one of them is rejected earlier by the dependence, control or effects
obligation. Hoisting those loads into fields would have unlocked no loop at all.

### Memory-carried reductions

A stack slot updated as `s = load(slot); s = s + x; store(slot, s)` is the in-memory
form of an integer reduction. Only six loops in both corpora match it, and none has a
recovered induction variable, so the narrow path had nothing to emit.

### mem2reg / SSA promotion

Almost half the loops are outside the committed coverage because their induction
variables live in memory stack slots (427/959 in draco, 230/459 in basis). A stack
slot only needs promotion when the existing memory-recurrence recogniser fails while
the store value is already `load + constant`; that intersection is empty in both
corpora. Promotion alone would turn the remaining shapes into non-affine loop-carried
values, which the dependence obligation still rejects, so the pass would add analysis
cost without coverage.

### Device residency / multi-kernel chains

With the default cost gates a real decode runs zero GPU dispatches. Lifting the gates
emits 64 kernels, and a real decode then executes nine dispatches before a later
kernel is correctly refused by the bounds contract. The total transferred data is
1.68 MB and no dispatch consumes another one's output, so residency would save
nothing measurable today. The capability is parked until the emission surface covers
producer-consumer chains.

### Interprocedural effects / leaf inlining

98.5% of the side-effect loops contain calls (391/397 in draco, 87/87 in basis).
Restricting to callees that are leaf functions, free of side-effect operators, small
enough to inline and light on memory operations, only 18 loops in draco and 3 in basis
have every callee inlineable - 4.5% and 3.4% of the candidate pool, below the closing
threshold. Those loops would still have to pass the remaining obligations afterwards.

## The recurring lesson

Kernel benchmarks are not a workload result. A kernel can be fast, bit-exact and never
execute. For every direction we now ask first: does the real path reach it, and does
the host-side cost model leave anything to win? Two of this project's shipped
capabilities (reductions, fused producers) remain behind explicit flags for exactly
this reason: they are correct, but the mandatory upload and readback dominate.

The one positive result of the architecture pass was not a new kernel: the runtime host
was building trace events that the sink then discarded, which cost 66 ms of a 100 ms
decode. Asking `Trace::enabled` before building removed it and made the rewritten path
as fast as raw wasmtime again.
