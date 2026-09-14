# Windows rendering parity audit

Scope: local ZUI source and integration checkout; no Zeron style changes. Read-only code audit, plus Microsoft documentation and successful `gh repo view`, `gh issue list --search blur`, and `gh pr list` for zeronsh/zui. Subsequent pixel tests confirmed and corrected the fractional clipping defect described below.

## Text on transparent windows: existing protection is correct

`crates/gpui/src/window.rs:4369` rejects subpixel rendering whenever the native window background is nonopaque, before checking platform capability. This guard exists in both local checkouts. Windows `window.rs:856` returning true describes capability, not unconditional use. Changing it would unnecessarily disable ClearType on opaque windows.

Windows subpixel blend state (`directx_renderer.rs:1476`) deliberately leaves alpha untouched. That would be unsafe for a translucent destination, but the shared guard prevents it. macOS reports no subpixel support. Windows monochrome glyphs instead output ordinary straight RGBA (`shaders.hlsl:1214`), so the corrected ordinary alpha blend applies.

Microsoft likewise switches transparent Direct2D targets to grayscale and warns against ClearType on transparent surfaces: [alpha mode documentation](https://learn.microsoft.com/en-us/windows/win32/api/dcommon/ne-dcommon-d2d1_alpha_mode). No further text-policy change recommended without a runtime counterexample.

## Rounded blur edges: shared hard mask

Windows `shaders.hlsl` `backdrop_composite_fragment`, Metal `shaders.metal:1364`, and wgpu `shaders.wgsl:1578` all discard when the rounded-rectangle signed distance exceeds zero and otherwise replace the full blurred pixel. This is a binary mask, unlike the coverage ramp used by ordinary rounded fills. High-contrast content can consequently expose stair steps or edge changes during fractional movement on all three renderers. This is a shared quality limitation, not evidence that Windows fails parity.

Fractional clipping needs pixel coverage tests, especially a content-mask right or bottom boundary landing exactly on a pixel center. Windows uses explicit greater-than clipping while Metal uses hardware clip distances; rasterizer boundary rules can affect exact ties. This remains a hypothesis, not a confirmed defect. [Direct3D rasterization rules](https://learn.microsoft.com/en-us/windows/win32/direct3d11/d3d10-graphics-programming-guide-rasterizer-stage-rules).

## Crop and filter coordinates

Windows backdrop.rs snapshots actual cached width/height and uses the same source rectangle when mapping back. Horizontal step is downsample/width; vertical step is 1/ceil(height/downsample). This matches wgpu_renderer.rs:1360-1383. No missing half-texel correction or crop-origin shift was found.

A small shared approximation relative to Metal remains: when source height is not divisible by downsample, the effective vertical sigma is sigma * height / (downsample * ceil(height/downsample)). Metal uses a full-resolution MPS Gaussian. The discrepancy is small and does not justify changing only Windows without a visible reproduction.

The integration paired-tap optimization is valid at unit stride: the final missing partner for odd kernel radius is zero-initialized, its first partner remains positive, and the 132-entry storage covers all accesses up through index 129. Large radii use the separate uncached loop. No divide-by-zero case found for the admitted finite sigma range.

Recommendation: retain current semantics; prioritize real renderer tests for monochrome/polychrome alpha, fractional clips, movement, and resize. Do not add opacity compensation or change shared rounded-mask behavior on speculation.

## Pixel validation and correction

The fractional clipping hypothesis reproduced against the real quad rasterizer: at a right clip edge of 22.5 pixels, the blur changed a pixel that normal primitives excluded. Changing the blur composite upper-bound comparison from `>` to `>=` fixes the leak and follows the top-left rasterization rule. The regression passes at 100%, 125%, 150%, and 200% scale.

All 21 integration Windows tests pass, including a new real monochrome/polychrome sprite alpha test at those scales. Existing tests cover nested composition, moving snapshot regions, resize, and resource recovery. These WARP pixel tests do not establish hardware presentation, font appearance, or scrolling performance parity. Zeron opacity and styling remain unchanged.

## Animated cache differential check

A further WARP test compares retained blur resources against freshly recreated resources over 36 frames at three window sizes (193x131, 137x99, 211x143). It moves partially offscreen nested panels across changing striped content, alternates blur radii 3 and 16, and clips the child to its parent. Outputs match within two byte levels per channel; no cache history or resize artifact reproduced. This is a deterministic rendering test, not a measurement of interactive frame timing.
