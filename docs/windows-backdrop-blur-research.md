# Windows nested within-window backdrop blur

Researched 2026-09-14 against local commit `08834f425b0b633a782f84e5bc6c48e50bca964c`, Microsoft documentation on the web, and first-party source retrieved with `gh api`. Static inspection only; no renderer implementation or GPU tests were performed.

Recommendation: extend the existing native D3D11 renderer with ordered snapshot → separable Gaussian filter → clipped replacement composite operations. Verify blur boundaries in shared scene batching and make targeted fixes where needed. The initial Metal/wgpu scheduling exposes ordering risks to cover in the Windows port.

Concurrent work: another agent is implementing the Windows port and scene tests. The absence of Windows blur described below refers to the initial commit, not the current working tree. Source citations are pinned to that initial revision; Microsoft GitHub sources are pinned to revisions resolved during research. Learn URLs are live documentation. The uncommitted example below intentionally uses a local link. This research does not review or validate the completed port.

Port considerations: DWM window materials cannot provide nested GPUI paint checkpoints. Verify batch splitting, same-layer ties, terminal blurs and dependencies in the padded sampling footprint; child captures must include parent tint/marks. Unbind overlapping D3D11 SRVs/RTVs and composite filtered RGBA as replacement.

## Initial rendering paths (before the concurrent port)

| Path | Finding and local evidence |
| --- | --- |
| Windows | `window.rs` constructs `DirectXRenderer`, not wgpu. Its draw loop handles only `scene.batches()` and never consumes `scene.backdrop_blurs`. The main target is BGRA8 UNORM; swap chains are single-sampled. DirectComposition attaches the swap chain to one visual. See [window.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_windows/src/window.rs) (`new`, line 137) and [directx_renderer.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_windows/src/directx_renderer.rs) (draw loop 344, visual 927, swap chains 1203 onward). |
| Window material | `set_background_appearance` selects HWND composition attributes or DWM Mica/Mica Alt. This is separate from scene-local blur. See [window.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_windows/src/window.rs), line 866. |
| wgpu | Already snapshots a padded region, filters horizontally/vertically into separate textures, resumes the main pass with Load, then composites without blending. It supplies a copyable intermediate frame when surface COPY_SRC is unavailable. It currently skips blur indices ≥32. See [wgpu_renderer.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_wgpu/src/wgpu_renderer.rs) (`process_backdrop_blur`, 1274; intermediate frame, 1767; scheduling, 1876) and [shaders.wgsl](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_wgpu/src/shaders.wgsl), 1423 onward. |
| Metal | Ends the encoder, copies the padded region, invokes MPS Gaussian blur, resumes with Load, and replaces the rounded region. Scratch textures and kernels are cached. See [metal_renderer.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_macos/src/metal_renderer.rs), 999–1104 and 1357 onward; [shaders.metal](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_macos/src/shaders.metal), 1339 onward. |

The “macOS only / other renderers ignore” comments in Scene and Window are stale relative to wgpu's implementation. Existing `blur_radius` is treated as Gaussian sigma in device pixels, clamped to at least 1 in both reference renderers; Window scales logical bounds, radius and clipping before scene insertion. [Window paint API](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui/src/window.rs), 4055–4094; renderer functions above.

## Windows API choices

- **DWM is insufficient for this task.** System backdrop types describe material behind the window, including its non-client area; they provide no GPUI paint-order checkpoint. Legacy `DwmEnableBlurBehindWindow` no longer produces blur starting with Windows 8. [System backdrop types](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/ne-dwmapi-dwm_systembackdrop_type), [legacy blur API](https://learn.microsoft.com/en-us/windows/win32/api/dwmapi/nf-dwmapi-dwmenableblurbehindwindow).
- **Composition backdrop brushes are real within-visual backdrop effects.** Microsoft's sample connects a backdrop source to Gaussian blur and tint, with a capability fallback. However, this repo flattens GPUI primitives into one swap-chain visual: inferring from that architecture, composition cannot recover intermediate GPUI layers from the final texture. Using composition for nested blur would require exposing corresponding content/effect layers and ordering to the compositor. [CompositionBackdropBrush](https://learn.microsoft.com/en-us/uwp/api/windows.ui.composition.compositionbackdropbrush), [Microsoft sample inspected via gh](https://github.com/microsoft/WindowsCompositionSamples/blob/92e5ee73e78ff81bfe46b8045f50c948dc55d6e1/SampleGallery/Samples/SDK%2015063/BrushInterop/BackdropTintBlurBrush.cs).
- **D3D11 filtering fits the current renderer.** Copy the current target region into an independent shader-readable snapshot, filter through ping-pong render targets, then restore the main target and draw the result. Copies require compatible formats, valid texel bounds and matching sample constraints; they do not resize or filter. Explicitly unbind conflicting SRVs/RTVs: binding an SRV overlapping an output causes D3D11 to bind NULL. [CopySubresourceRegion source inspected via gh](https://github.com/MicrosoftDocs/sdk-api/blob/554e06be52a53ae819b1011353303a6c72fbdb5d/sdk-api-src/content/d3d11/nf-d3d11-id3d11devicecontext-copysubresourceregion.md), [shader resource binding](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nf-d3d11-id3d11devicecontext-pssetshaderresources).
- **Direct2D Gaussian blur is an alternative filter**, still requiring an ordered snapshot and interop. Its sigma uses DIPs; soft borders introduce transparent black and hard borders use mirrored samples. Those semantics need reconciliation with the existing device-pixel sigma and edge-clamp behavior. [Direct2D Gaussian blur](https://learn.microsoft.com/en-us/windows/win32/direct2d/gaussian-blur).

## Nested order requirements and gaps

Required sequence: background → parent snapshot/blur/composite → parent tint/content → child snapshot/blur/composite → child tint/content. Each snapshot must include all prior content in its sampling footprint, including earlier blur results, and exclude later foreground. One frame-start snapshot reused for every blur cannot satisfy this.

Static inspection of the initial revision identifies these risks, not runtime-confirmed failures. They are requirements for the concurrent port, not a verdict on its changing implementation:

1. **Batches can span blur orders.** [Scene](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui/src/scene.rs), 299–492, merges primitives by kind/order/texture without consulting the separate blur list. Both GPU implementations trigger blur only when `blur.order <= batch_first_order`; neither drains pending blurs after the batch loop. A direct scene blur with no later primitive can therefore be skipped.
2. **The invisible shadow is not a universal barrier.** [Window](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui/src/window.rs), 4061, inserts a transparent Shadow before the blur. Outside a layer these get separately assigned overlap orders, so a terminal splitter can precede its blur with no later batch to trigger it. Inside a layer, Scene gives both the layer's order; multiple blur/content operations can share that order, causing the renderer to process blurs before intervening same-layer content. Consecutive shadow splitters can also batch together.
3. **Spatial ordering must include sampling dependencies.** [BoundsTree](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui/src/bounds_tree.rs) assigns overlap-based orders, while Scene inserts only clipped visible blur bounds. The filter samples beyond those bounds (approximately three sigma plus padding). Previously painted nearby content may be reordered despite contributing samples. Add a focused halo-dependency regression before changing scheduling; if it fails, preserve contributing prior draws across the blur boundary or account for the padded sampling region in dependency bounds.

Recommended scope: test and repair the existing batch boundaries first, preserving ordinary batching elsewhere. Define equal-order behavior, empty/trailing blur processing, and cached `Scene::replay` preservation. Use an explicit sequence/barrier mechanism only if a focused boundary fix cannot preserve these dependencies; a broad scene redesign is not a prerequisite for this port. Resolve/composite pending paths before a snapshot; Windows already has a separate MSAA path intermediate ([path rendering](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_windows/src/directx_renderer.rs), 592 onward).

For filtering, reuse the reference implementations' padded, viewport-clamped capture and rounded/content-mask composite. Replace the filtered RGBA region rather than source-over blending it again; draw tint afterward. Validate premultiplied alpha and color-space behavior explicitly. Reuse bounded scratch resources, avoid CPU readback in production, and recreate resources on resize/device loss. Treat these as design recommendations, not measured performance claims.

## Testing recommendation

- **Use the new toggleable [backdrop_blur example](../crates/gpui/examples/backdrop_blur.rs):** with blur enabled, background blue stripes should soften in the outer panel, orange parent marks should soften under the inner panel, and its later white foreground line should stay sharp. Toggle off to expose the source pattern while retaining panel tint/borders. The example uses distinct nested `paint_layer` calls; add separate same-layer and trailing-blur cases to cover the scheduling gaps above. Example inspected during the concurrent port, not executed here.
- **First, backend-independent scene scheduling tests:** parent/child with intervening content; two blurs in one layer; shadow-only neighbors; direct trailing/blur-only scenes; nearby content inside the sampling halo but outside visible bounds; clipped/disjoint siblings; cached replay. Assert event ordering, not just batch counts.
- **Then Windows GPU readback tests:** high-contrast background, parent tint and text, overlapping child and sharp child foreground. Compare against a sequential CPU reference with a documented tolerance. Include alpha, rounded clips, window edges, DPI 100/150/200%, zero/large radii, different scratch sizes, and more than 32 regions. A linear gradient alone can pass when blur is missing; retain sharp-edge checks.
- **Reuse existing evidence:** Metal has headless gradient, sharp-edge, window-edge and scratch-reuse tests in [metal_renderer.rs](https://github.com/zeronsh/zui/blob/08834f425b0b633a782f84e5bc6c48e50bca964c/crates/gpui_macos/src/metal_renderer.rs), 2400 onward. Extend those scenarios for nested ordering; MPS and wgpu approximations need tolerance rather than byte-identical cross-backend output.
- **Native validation:** run with the D3D11 debug layer, hardware and WARP, composition enabled/disabled, resize/minimize/restore and device recreation. Require no resource-binding/copy errors. Measure GPU frame time and scratch-memory retention at increasing overlap depths, then visually check scrolling and animated nested popovers. Existing [macOS validation notes](macos-resource-usage.md) distinguish offscreen correctness from foreground presentation; apply the same distinction on Windows.

Research access used successful `gh api` reads of the Microsoft sample and SDK copy-method source linked above, plus Microsoft Learn pages. No performance results or Windows visual correctness are claimed by this note.

## Implementation and validation follow-up

The working branch adds the native filter in
[`directx_renderer/backdrop.rs`](../crates/gpui_windows/src/directx_renderer/backdrop.rs),
with a fresh snapshot for each blur, a separable Gaussian filter, and a clipped
replacement composite. Shared scene batches now stop at blur orders, including
the shadow-only case where the invisible splitter previously merged with earlier
shadows. The Windows draw loop also processes trailing blurs.

Validation commands:

```powershell
cargo test -p gpui_windows --lib
cargo test -p gpui --lib scene::tests -- --nocapture
cargo check -p gpui --example backdrop_blur
cargo run -p gpui --example backdrop_blur
```

The Windows tests use WARP through the production draw loop and read pixels back
before presentation. They compare nested blur, intervening content, sharp
foreground, clipping, and premultiplied transparency with a CPU reference.
All 16 Windows tests passed, including scratch reuse, idle release, resize,
device recovery, non-finite radii, and more than 32 blur-only operations.
The new HLSL entry points also compile with the Windows SDK's optimized `fxc`
compiler. Hardware presentation and performance still need interactive validation;
the final command opens a toggleable nested-panel example for that purpose.

The four focused scene tests passed after temporarily supplying two font fixtures
missing from this extracted repository: `assets/fonts/ibm-plex-sans/IBMPlexSans-Regular.ttf`
and `assets/fonts/lilex/Lilex-Regular.ttf`. Unrelated SVG tests reference these at
compile time. The temporary upstream fixtures were removed afterward.

This port retains existing layer-order semantics. Multiple sequential blurs sharing
one layer order, and spatial ordering outside the visible bounds but inside the
sampling halo, remain separate shared-scene concerns. Use distinct paint layers
for nested panels, as demonstrated by the example.
