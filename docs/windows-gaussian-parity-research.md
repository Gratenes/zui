# Windows Gaussian filter parity research

Scope: ZUI integration checkout, 2026-09-14. Source inspection and primary documentation only; no macOS captures or hardware timing measurements. No renderer or Zeron style changes in this research task.

## What Apple's filter guarantees

Apple describes `MPSImageGaussianBlur` as an approximate Gaussian optimized for common image processing requiring roughly ten bits of precision. It recommends convolution with explicit weights when analytically clean filtering is necessary. Its exact approximation is not documented here. Consequently, matching an ideal Gaussian does not prove bit-identical MPS output. The local Metal comment calling this a “true gaussian” is stronger than the API guarantee. [Apple Gaussian filter documentation](https://developer.apple.com/documentation/metalperformanceshaders/mpsimagegaussianblur).

Sigma means standard deviation. The initializer documentation discusses a rough support estimate based on ignoring small weights; it does not promise an exact implementation cutoff. Treating that discussion as an instruction to enlarge the Windows kernel would be unjustified. [Apple initializer documentation](https://developer.apple.com/documentation/metalperformanceshaders/mpsimagegaussianblur/init(device:sigma:)).

## Proven common behavior

- Metal's `ensure_gaussian_kernel` sets edge mode to clamp; Windows `BackdropResources::new` uses `D3D11_TEXTURE_ADDRESS_CLAMP`. Both extend the nearest edge pixel rather than introducing black beyond the source. [Apple edge modes](https://developer.apple.com/documentation/metalperformanceshaders/mpsimageedgemode), [Microsoft texture addressing](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/ne-d3d11-d3d11_texture_address_mode).
- Both normalize the requested device-pixel sigma to at least one and use `ceil(3*sigma)+2` snapshot padding. See local `crates/gpui_macos/src/metal_renderer.rs` around 1013 and Windows `directx_renderer/backdrop.rs` around 252.
- Metal's layer and scratch textures use `BGRA8Unorm`. Windows render targets and all blur textures use `DXGI_FORMAT_B8G8R8A8_UNORM`. Neither local blur path inserts an explicit RGB transfer-function conversion. The format declarations do not reveal a Windows-only sRGB mismatch. Microsoft distinguishes normalized formats from explicit `_SRGB` formats. [DXGI formats](https://learn.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format), [Metal format capabilities](https://developer.apple.com/metal/capabilities/).
- Both snapshot already-composited window content and replace the blurred RGBA region without ordinary source-over blending. Correct destination alpha must already have been produced by earlier primitives; the existing Windows blend fixes address that separate issue.

## Proven implementation differences, unknown visual impact

Windows performs two separable shader passes with normalized sampled Gaussian weights and a three-sigma truncation. It downsamples at sigma 16 and above, with stride capped at four. Metal submits the full-resolution snapshot to MPS. MPS internals may themselves approximate or subsample; the inspected API documentation does not disclose how.

Windows stores the horizontal intermediate in an eight-bit texture, introducing a quantization step before the vertical pass. Metal exposes an eight-bit destination, but its internal intermediate precision is unspecified here. A float intermediate could reduce numerical error against an ideal Gaussian, but claiming it matches MPS better requires measurements; it also changes memory/bandwidth requirements.

Windows uses `ceil(height/downsample)` output texels while its vertical sigma remains `sigma/downsample`. For non-divisible source heights this slightly narrows physical vertical sigma. This behavior also appears in the local wgpu renderer. It is an analytically identifiable difference from an ideal isotropic full-resolution Gaussian, not yet a demonstrated macOS appearance discrepancy.

## Useful next checks

Compare actual Windows pixels against a full-resolution Gaussian reference for impulses, alternating stripes, translucent colors, and step edges. Cover non-divisible extents, sigma immediately below/above 16, 24, and 32, and cropping at each window edge. Measure axis symmetry, centroid, spread, constant-color preservation, and maximum channel error. This can identify accidental extra approximation without pretending to know MPS's implementation.

Do not add Windows-only gamma, tint, contrast, or arbitrary radius compensation. Do not port a different blur library merely because it labels itself Gaussian. Exact cross-platform identity ultimately requires a shared specified filter on both renderers or matching reference captures; the former would change macOS behavior and exceeds a Windows-only parity fix.

## Reproduced aliasing and correction

A WARP test exposed sparse downsampling aliasing: shifting two-pixel black/white stripes by one pixel produced center RGB 244 instead of roughly gray at sigma 32. The corrected path filters native pixels before reducing each axis. The horizontal intermediate keeps full height; the vertical pass then reduces height. Native-pixel sigma also removes the height-rounding discrepancy. Paired taps are used only when output samples align with source texel centers.

The final regression checks four stripe phases and both orientations on a 256x256 fixture, keeping the center outside clamped-edge influence. All 23 Windows tests pass. This improves Gaussian fidelity, not proven MPS equivalence. Strong blurs require more texture reads and a taller horizontal intermediate; hardware frame timing remains unmeasured.
