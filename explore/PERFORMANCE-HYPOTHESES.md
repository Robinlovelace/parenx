# Performance hypotheses, and how to test them later

Written after the first implementation step (#4, PR #7). Robin's steer for this
round: do not spend time benchmarking on a loaded box, choose the code you believe
is fast from reading it, and record what to test when the machine is quiet.

Everything below is a **hypothesis**, not a measurement. The stage numbers quoted
are from the extension pass, taken when the host was at load 15 to 20, so they are
indicative of shape and not of magnitude.

## The measurement protocol to use when the box is quiet

- Pin the process: `taskset -c 2`. On this host an unpinned single-thread run
  swings by up to 2x from P-core/E-core placement alone.
- Report **CPU time** (`time.process_time`) as the primary axis, not wall clock.
  Cache-stall pressure on a loaded box inflates both, but CPU time is the one that
  survives contention best.
- Three repeats, and record `cat /proc/loadavg` alongside each. Discard any repeat
  where the load average moved materially mid-run.
- `perf` counters are blocked on this host (`perf_event_paranoid=4`, no root), so
  there is no instruction-level profile available. Stage timers are the tool.
- Compare like with like: the same input, the same parameters, and the same strip
  height. The 62-pixel strip caveat in `explore/README.md` shows that raster
  boundaries are not always cosmetic.

## What the stage split looks like now

Five-dataset run, 1 km to 5 km circle, 16.3x the network length and 24.7x the
raster area. Indicative CPU seconds:

| stage | 1 km | 5 km | ratio | shape |
|---|---|---|---|---|
| buffer and union | 0.44 | 4.07 | 9x | linear in length |
| rasterise (after the strip fix) | 0.10 | 1.0 | 10x | fixed from ~50x |
| `remove_small_holes` | 0.04 | 1.4 | 35x | raster area |
| `skimage` skeletonize | 0.15 | 6.8 | 45x | **above** raster area |
| vectorise, lossless | 0.27 | 3.3 | 12x | linear in pixels |
| conflict check, per round | 0.06 | 1.1 | 18x | linear-ish |

Empty space is most of the raster: the buffered network fills 20 percent of the
bounding box on Princes Street, 21.6 percent on Leeds, and **2.5 percent** on the
Doncaster rail extent. So a large share of the raster stages is spent on pixels
that will never hold a skeleton.

## H1. Rasterise and skeletonise per component, not once over the bounding box

**Hypothesis.** The area-proportional stages (`remove_small_holes`, `skeletonize`,
and the array allocations) are dominated by empty bounding-box space, so processing
each connected component of the buffered union inside its own bounding box, with a
one-pixel margin, should cost roughly the sum of the component areas instead of the
global area, and should bound memory by the largest component rather than by the
extent.

**Why I believe it.** The affine transform is shared and global, so a crop with the
same transform and a multiple-of-strip-height offset holds the same pixels. The
existing strip rasteriser is the same idea in one dimension and it removed a 50x
superlinearity without changing the raster except by 62 pixels.

**Expected.** On dense urban circles, components still spread across the circle, so
the sum of component areas may be 40 to 60 percent of the global area: 1.5x to 2.5x
on those stages. On sparse, extensive networks like Doncaster it should be far
more, because the fill is 2.5 percent.

**Test.** First the cheap one, no timing needed: sum the buffered components'
bounding-box areas and compare with the global bounding box area, on all five
datasets. If that ratio is near 1 the win disappears and the hypothesis is dead. If
it is 0.5 or lower, implement the crop and time the two stages.

**Falsification.** Summed component area close to global area, or the crop boundary
changing the raster on more than a handful of pixels, which `bench/check_raster_parity.py`
is already set up to detect.

## H2. The thinning is worse than linear in area

**Hypothesis.** `skimage.skeletonize` grew 45x for a 24.7x area increase, which is
above area-proportional. If it is genuinely superlinear, then H1's win is larger
than area-proportional and the stage is the first thing to attack.

**Why I believe it.** One run under load is not evidence, but the direction is
consistent with cache effects on large sparse boolean arrays.

**Test.** A synthetic filled rectangle of increasing size, thinned in isolation,
pinned and repeated. Fit the exponent from the CPU times. If it is above 1.05, H1
gains more than expected; if it is 1.0, this was cache noise.

## H3. The first `unary_union` in the buffer is the expensive part

**Hypothesis.** `get_geometry_buffer` unions all input lines, buffers the result,
then unions the buffers. The first union exists to make mitre joins at shared
endpoints correct. For 18,530 input lines on the 5 km circle this is likely most of
the 4.07 s, and buffering each line with round joins and then unioning would be
faster.

**The catch.** That changes the geometry at junctions, from mitre to round, so it
is a quality trade-off rather than a free win. Test it as a speed/quality pair:
time both, and measure how much the buffered area and the resulting skeleton differ
at junction clusters. Only worth taking if junctions are visibly unaffected at the
default 8 m buffer.

## H4. The Python chain walk is not worth vectorising yet

**Hypothesis.** The plain-Python chain walk costs about 0.7 s on Leeds' 1.73M
skeleton pixels, which is under 5 percent of the run once the raster stages are
fixed. It only becomes interesting after H1 and H2.

**Test.** Re-run the stage split on the quiet box. If the vectorise stage is still
under about 10 percent of CPU, leave it. This is also the point where a Rust core
for the pixel and graph stages becomes worth costing, since the array work would
move wholesale.

## H5. Parallelism is a scheduler problem, not a thread-pool problem

**Hypothesis.** The existing `skeletonize_tiles` loop is serial and its tiles are
about 1 km, not the 2 km the argument name suggests. Adding a process pool without
fixing the seam semantics would trade correctness for speed: `get_gap_fill` joins
endpoints within `buffer / 2` of each other near a boundary and can add links that
were never in the network.

**Test.** A matrix of tile origin, tile size and worker count, with the output
compared against the single-tile result, on the 3 km circle. Correct seam semantics
first; the speed claim is only meaningful afterwards.

## H6. The conflict check rebuilds its spatial index every round

**Hypothesis.** `simplify_network` rebuilds an STRtree over all lines on each pass.
At 5 km only 11 of 28,955 lines are reverted, and at 3 km only 2, so almost all of
that work is thrown away. Building the tree once and re-querying only the
neighbourhood of the changed lines should remove most of the per-round cost.

**Why I believe it.** The measurement is in the code path: the tree is built from
the full array on every call, and E is nearly all unchanged between rounds.

**Note.** This is the one hypothesis whose implementation should be output
identical by construction, since the candidate pair set is unchanged. That makes it
the safest next change and the best candidate to land while the box is busy, with
the oracle re-run once afterwards rather than the raster path being touched at all.

## Order I would do these in

1. **H6**, because it is small, local, and output-preserving by construction.
2. **H1**, because it is the largest remaining win and it also fixes the memory
   ceiling that stops London being run untiled at all.
3. **H2**, to decide whether H1 is worth more than area-proportional.
4. **H4 and H5** together, as the bridge to a compiled core if the raster fix is
   not enough on its own.
5. **H3** last, as it costs fidelity at junctions.

## Untested, explicitly

None of the above has been implemented or timed on a quiet box. The only
performance claim that has been verified in this work is the strip rasteriser,
which the tests in `bench/` cover directly.
