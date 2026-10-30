# IRIS GL renderer enhancement — handoff

Goal: make `libs/libgl` the definitive hardware-accelerated IRIS GL for GLES2/WebGL1, by moving T&L to the GPU, closing fragment-side gaps, and adding API coverage in demand order. The CPU reference rasterizer stays the semantic oracle throughout.

Repos: `github.com/sgi-demos/sgi-demos` (this repo), `github.com/sgi-demos/irix-igl` (reference only — GPL, read for semantics, don't copy code).

Read first: `docs/ARCHITECTURE.md`, `libs/libgl/rasterizer.h`, the header comments of `gl.c` and `gles2_rasterizer.c`, `scripts/ppm_compare.py`, `tests/smoke/README.md`.

## Ground rules

- Never `git commit` or `git push`. End each phase with changes staged and a proposed commit message, then stop for review.
- Every phase ends green on the parity gate (phase 1) for every demo it touches. No exceptions for "cosmetic" diffs.
- The CPU reference rasterizer defines correct output. When GPU and CPU disagree, the CPU path is wrong only if you can show which IRIS GL rule it violates.
- Nothing gets implemented without a demo that calls it (phase 0 decides).
- Keep `GLES2_RASTERIZER=ref` and the new `GLES2_TNL=cpu` (phase 3) working so every stage is bisectable.
- Original demo source is not modified to make the renderer's life easier.
- CPU first: every rasterizer-level feature lands in `reference_rasterizer.c` before `gles2_rasterizer.c`, and the GLES2 version is gated against it. Both backends must fill every `rasterizer_funcs` slot at all times — no ref-side stubs. T&L is the exception: it is shared via `tnl_common.h` (phase 2), not reimplemented.

## Phase 0 — symbol census

Produces the work queue for phases 4–6 and the regression set for phase 3.

1. Parse `include/gl/gl.h` and `include/gl/device.h` into `api.json` (`name, return_type, params`). Add `fm*` names from irix-igl `src/fontmgr.c` tagged `libfm`. Sanity-check the count against `docs/IRIS_GL_API.md`.
2. Classify each name in `libs/libgl/gl.c`: `impl` / `stub` (body has the `printf("%s unimplemented", __FUNCTION__)` pattern) / `missing`. Classify irix-igl `src/*.c` the same way (definitions there split return type and `name (args)` across two lines).
3. Scan `demos/*/` sources for call sites: API name followed by `(`; exclude definitions, demo-local functions of the same name (`draw`, `move`, `pnt` are common), comments, strings. Skip `demos/*/archive/`, `demos/gltest`, `demos/bench`.
4. Binary track: `demos/gview/archive/irix-3-binary/gview` plus any binaries the user supplies — ask for a path, don't assume. `strings -n 3` matched against `api.json`, flagged `approx`. Use Ghidra headless if the user provides it.
5. Emit `scripts/gl_census.py` (stdlib only), `docs/GL_CENSUS.csv` (`name, libfm, used_by, call_sites, libgl_status, igl_status, binary_hits`), and `docs/GL_CENSUS.md` with three ranked lists: **used ∧ stub/missing** (work queue), **used ∧ impl** (regression set), **unused** (out of scope).

## Phase 1 — parity harness

Turns the existing ad-hoc `ppm_compare.py` flow into the gate every later phase runs against.

1. Add `make parity DEMOS=...` : builds native, runs each demo with `GEN_FRAME_PPM_FILES=1` under `GLES2_RASTERIZER=ref` and default gles2 for N frames, runs `ppm_compare.py`, fails over threshold.
2. Frame stepping must be deterministic per demo; where a demo randomizes, seed it via an env var the shim already exposes or add one in the shim, not the demo.
3. Tolerance policy in one place: exact match for CI-mode demos; ±1 per channel on ≤ 0.5% of bytes for RGB (float rounding on edges). Anything else is a failure.
4. Extend the smoke gallery with a third column: ref, gles2, diff heat-map, so failures are reviewable without re-running.
5. Record a baseline for every demo in the phase 0 regression set before phase 3 starts.
6. Third oracle: where a frame from real hardware or MAME exists for a `demos/gltest` case (dither, provoking vertex, CI depthcue ramp, line-style phase), the CPU rasterizer is gated against it. GPU-vs-CPU parity cannot catch a shared misreading of the docs; this can.

## Phase 2 — shared T&L math

Prevents the GPU port from forking the semantics.

1. Extract the pure math from `gl.c` — matrix ops, `light_vertex`, fog, CI depthcue/`lshaderange` — into `libs/libgl/tnl_common.h`, written in the C ∩ GLSL ES 1.00 subset (`float`, `vec3`/`vec4` via macros, no pointers, no structs the shader can't take).
2. Compile it into `gl.c` (CPU path) and inline it into the vertex shader source string at program build time.
3. Parity gate: must be byte-identical to pre-phase output on the CPU path. This phase changes no behavior.

## Phase 3 — GPU T&L

The main event. Incremental behind `GLES2_TNL=cpu|gpu` (native env, `?tnl=` on web), default `cpu` until the regression set passes.

1. Extend `rasterizer.h` with a world-space draw entry: object-space vertex + normal + color/index + texcoord, plus the current matrix, material, light, lmodel, fog and depthcue state as a uniform block struct. Keep the existing screen-space entry; the ref rasterizer continues to use it.
2. Vertex shader: `MSINGLE`/`MVIEWING`/`MPROJECTION` handling, IRIS lighting from `tnl_common.h` (`LOCALVIEWER`, 1/(a0+a1·d) attenuation, `TWOSIDE`, `LMCOLOR` modes, lighting only when a normal was supplied inside `bgn*`), fog, CI depthcue.
3. CI mode: pass the index as a float varying and resolve through the existing 4096-entry LUT texture per fragment; CI gouraud interpolates indices, not colors.
4. Flat shading: no `flat` qualifier in GLES2 — duplicate the provoking vertex's color across the primitive on the CPU at batch time. Match IRIS's provoking-vertex rule, verified against the ref rasterizer.
5. `clipplane()`: signed-distance varying + `discard`. Frustum clipping via the GPU clip volume; confirm CI depthcue is computed pre-clip exactly as the CPU path does.
6. Lines and points keep the CPU quad expansion (GLES2 line/point rasterization rules don't match); only the vertex transform moves.
7. Flip the default to `gpu` when the phase 0 regression set passes the phase 1 gate on native + web (Chromium, Safari — the fp16-varying note in `gles2_rasterizer.c` applies).

## Phase 4 — fragment-side gaps

Order by census. Each item: implement in the ref rasterizer, add a `demos/gltest` case, then implement in GLES2 and gate.

- Line styles (`setlinestyle`, `lsrepeat`, `lsbackup`): per-segment distance varying against a 16-bit pattern uniform, same shape as the screen-door.
- RGB writemasks (`RGBwritemask`, `wmpack`) and `logicop`: snapshot-texture composite, as the CI writemask path already does.
- `linesmooth` / `pntsmooth`: coverage from the expanded quads' edge distances.
- IRIS dither: hardware-matching dither matrix as a texture before byte quantization; needed for parity on 12-bit-plane targets.
- `zsource` / `zdraw` / full `zwritemask`: depth as a color attachment on demand.
- `stencil` / `stensize` / `swritemask`: GLES2 8-bit stencil.
- `acbuf` / `acsize`: FBO ping-pong accumulation.
- `blendfunction` full factor table; `msalpha`/`multisample` as supersampled FBO only if the census shows a caller.

## Phase 5 — API coverage from the work queue

Port semantics, not code. irix-igl is the spec for behavior it got right on real hardware; its implementations are fixed-function-shaped and won't fit this pipeline. Its `NOTE` comments (58) are the trap list — read them all before starting.

Likely order, subject to census:
1. `getgdesc` / `gversion` / `getgconfig` / `getgpos` capability and query tables — pick one target machine profile and document it in `ARCHITECTURE.md`.
2. `rectread` / `rectwrite` / `lrectread` / `lrectwrite` / `rectzoom` / `rectcopy` / `pixmode`: FBO readback and textured-quad writes.
3. Texture beyond the single modulate path: `texdef2d`, `tevdef`, `texbind`, `tevbind`, `t2f`/`t2i`, wrap modes, minification.
4. Window management: `winset`, `winpush`, `swinopen`, `winclose`, `wintitle`, `winat` — one-window-per-demo remains the model; implement as much as the callers need.
5. `fm*` font manager subset the demos use, on top of the BDF path.

## Phase 6 — areas neither renderer has

Only with a census hit.

- NURBS / `patch` / `crv` / `defbasis`: own tessellator honoring `curveprecision`/`patchprecision` subdivision counts; don't route through GLU.
- `pick` / `gselect` / feedback: a mode of the existing CPU geometry pipeline — record hits instead of emitting to the rasterizer.
- Stereo, video, `dgl*`: stubs unless a demo demands them.

## Phase 7 — docs

1. Update `docs/ARCHITECTURE.md`: replace the "TBD: future OpenGLES rasterizer" line, add the T&L split, the `tnl_common.h` shared-math design, the parity policy, the target-machine profile.
2. `docs/GL_CENSUS.md` kept current by re-running `gl_census.py` at the end of each phase.
3. README: note `GLES2_TNL` and `make parity`.

## Phase 8 — upgrade modes

The payoff for GPU T&L. Strictly opt-in; faithful mode stays the default and is the only thing the parity gate checks.

1. Flags: `?hz=` / `SGI_HZ` (render rate), `?res=` (framebuffer size, with supersample factor), `?models=hd` (replacement geometry). All default to the original values.
2. Rate decoupling: where a demo ties simulation to the 30 Hz frame tick (Flight physics/timers, Arena netcode), decouple through the shim or a `patches/` entry, never by editing the original source. If a demo can't be decoupled cleanly, it doesn't get `hz` and says so.
3. Supersampled AA and higher-resolution output: render to a larger FBO and downsample on present; the original-resolution path is untouched.
4. No parity requirement in upgrade modes, but `make parity` must still pass with all flags unset after every change here.
5. Web: per-frame WASM work in upgrade modes should be near zero — profile before accepting any CPU-side per-vertex step.

## Deliverable per phase

Staged diff + proposed commit message + parity report. Then stop.
