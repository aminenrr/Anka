# AGENTS.md: Anka fork (aminenrr/Anka)

Notes for any coding agent working on this fork. Read this first.

## What this fork is for

Run **Donkey Kong Bananza** (Switch 2, title `040074A01BF12000`) as close to 60 fps as possible on the
owner's PC. Do it by porting *general* fixes from **Fritchii** (github.com/Frizti/Fritchii, a Lichii
fork, branch `nx2`) into Anka (upstream github.com/EddizEge/Anka, an Eden fork, GPL-3.0).

- Test PC: Windows 11, i7-12700H (14C/20T), 32 GB RAM, laptop RTX 3070 Ti (Ampere).
- The owner builds and plays. Agents port code, push branches, open **draft** PRs, and keep CI green.
  Never mark a PR ready for review, and never merge into `anka`, without the owner's OK.

## Background findings (2026-11-11)

- Fritchii has **no Zelda lock**. No code checks the OoT title ID `0400486024DEC000`. Its program-ID
  gates are for Mario Kart World, Mario Tennis Fever, Star Fox, Ninja Gaiden and FE upscale. Fritchii
  runs OoT at 40-60 fps but Bananza at 10-20 fps. Why is unverified. It is most likely because Bananza
  is far heavier and lacks Anka's Bananza-specific work.
- Anka already bundles Lichii nx2 as `core2`, used only for OoT. Everything else runs on the main core.
- Anka's Bananza speed comes from Bananza-specific work: host-GPU terrain indirect draws
  (`EDEN_NX2_HLE_INDIRECT`), about 60 per-draw shortcuts, the GPU thread stage split (`nx2_stages.cpp`,
  `gpu_split.cpp`), macro JIT filtering (`EDEN_NX2_JIT_SAME`), the DK dynamic-fps patch
  (`EDEN_NX2_DK_DYN`), and a **PGO** link trained on Bananza (+16-22% in upstream's measurements).
- **Already in Anka, do not port:** `macro_memo.h`, `EDEN_NX2_ASTC_BC1`, chunked texture uploads
  (`UploadInChunks`). Always grep Anka before proposing a Fritchii port.
- Fritchii patches **do not apply cleanly** (`git apply --check` fails). Every port is hand-adapted.

## Done

| PR | Branch | What | Result on the test PC |
|----|--------|------|------------------------|
| #1 | `port/fritchii-rumble` | HID cmd 214 `SendNx2VibrationValues` now forwards handle/value pairs to `ResourceManager::SendVibrationValue` (it was a stub that dropped them). Adds 213 (no-op) and 220 (replies a zero byte). From Fritchii `efe9283d`. | **Works great**: rumble confirmed in Bananza. |
| #2 | `port/fritchii-zero-copy-uploads` | Large chunked uploads unswizzle straight into mapped staging (`PlanUnswizzleImageRows` / `ExecuteUnswizzlePlan` in `texture_cache/util.*`), with `runtime.Finish()` between chunks. From Fritchii `073b8779` + `b547bda6`. The old `UnswizzleImageRows` is left in place. | **No measurable fps change**, as expected. It only touches textures of 64 MiB or more, and Bananza's texture cache sits around 1.7 GiB of 5.4 GiB expected. |

Both PRs are still drafts, targeting `anka`.

## CI

- `.github/workflows/anka-test-build.yml` builds on every push to `anka` (the baseline) and to
  `test-build` (`anka` with every port branch merged in). It runs on `windows-2022` with MSVC, Qt 6.8.3
  and Ninja. Build time is roughly 1-1.5 h.
- The artifact is an `Anka/` folder with `anka.exe`, `anka-cli.exe`, Qt DLLs, the `start-*.bat`
  launchers, `anka-config.ps1`, `Games/` and an empty `user/` (portable).
- It has **no LTO, no PGO and no core2**, so it is about 15-20% slower than the official release.
  Compare `test-build` artifacts against `anka` artifacts from this workflow, never against the release.
- Markdown-only changes do not trigger it (`paths-ignore`).

## What the first test log shows (test-build f9713f5, Bananza, about 150 s)

- Settings: Vulkan, Res1X, GPU accuracy Low, FIFO vsync, frame gen **off**, the Anka Bananza preset
  (`BACK_STAGES=1`, `JIT_SAME=1`, `DK_DYN=1`, `MODE_PACE=2`, ...).
- nx2:vi2 present stats per 600 frames: the average frame is 17.6-34.4 ms (about 29-57 fps), and p95
  runs up to 45 ms. Hitches of 170-377 ms appear early, alongside "Draw skipped (no pipeline)", which
  is async shader building.
- **"Blocked on the GPU fence" is 0 us throughout.** The host GPU is *not* the bottleneck. Frames are
  limited on the CPU side: the guest CPU JIT and the GPU command thread (macro/engine processing).
  So GPU-side or upload-side ports won't raise fps. Work on the GPU command thread or the CPU side.

## Plan, in order

1. **GPU-thread drain fixes from Fritchii** (`9479d8fa`, `9960298c`, `e3d8e9c6`). This is the most
   likely real fps win, given the CPU-bound log. Risky: these commits target Fritchii's GPU split, and
   Anka has its own (`nx2_stages.cpp` A | BC, `gpu_split.cpp`). Read both designs first, port the
   *idea*, not the diff. Gate the change behind a new env var (for example `EDEN_NX2_DRAIN_FIX=1`) so the
   owner can A/B it in one build.
2. **`2940d9fa` native nanosecond timestamps on Ampere.** Small correctness/overhead fix. Low risk.
3. Smaller ones, undecided: the SDL motion-sensor restart backoff, `StubReplyZeroed` for HID commands,
   and a null guard in `ControllerUpdate`.
4. Not evaluated: `f8f29b57`, `c9ebb5c3` (game-specific fixes for other titles).
5. Known Anka issues worth looking at later: missing grass (#1), white/black bands (#4).
6. Shader hitches: check whether the Vulkan pipeline cache carries across runs (`user/cache/shader`),
   and whether a second run of the same area still shows "Draw skipped (no pipeline)".

Realistic target (an estimate, not measured): about 45-50 fps in heavy areas on this laptop, with 60
on screen through FSR3 frame generation. Frame gen raises displayed fps, not game-logic fps.

## How to work here

- One branch per port, cut from `anka`: `port/fritchii-<name>`. Commit message credits the Fritchii
  commit hash. Open a draft PR into `anka` with a test plan.
- Then merge the branch into `test-build` and push, so CI produces a build with everything together.
- No local compile is available to the agent. Check every signature against Anka's headers before
  committing, and use the CI log as the compiler.
- Ask the owner for numbers from the in-game OSD (`EDEN_NX2_OSD=1` + RTSS) or the
  `nx2:vi2 last 600 presents` lines in `user/log/eden_log.txt`. Compare the same area on both
  artifacts.
- After big performance changes, the PGO profile must be **retrained** on Bananza
  (`dist/anka/build/pgo_link.sh`, see `BUILDING_ANKA.md`). Otherwise the release-style build loses its
  16-22%.
