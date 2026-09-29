# SelShader — Forward Shading Toon/Cel Lighting (SM5 + ES3.1)

Anime/toon **cel shading** for Unreal Engine **forward shading**, working identically on
**desktop (Vulkan/SM5)** and **mobile (Vulkan ES3.1)** feature levels — including VFX/Niagara
point lights, spot lights, and directional sun, with crisp band edges and per-pixel
ShadowColour handling in both preview paths.

Tested on **Unreal Engine 6.0** (custom source build).

## Credits & sources

This is a forward-shading, engine-level rework of the original Sel shading model:

- **Original Sel Shader** — envieous: <https://github.com/envieous/UnrealEngine-SelShader>
  (the original deferred-shading toon shading model for UE5, and the forum thread where it
  was released and discussed:
  <https://forums.unrealengine.com/t/ue5-anime-toon-cel-shading-model-works-with-launcher-engine-versions/544226>)
- **SEL+Fixed community rework** — bug fixes for light interaction (TotalLight accumulation,
  LightmapFactor zero-lightmap guard, mobile branch) which this repo builds on.

What this repo adds on top:

1. **Forward shading support** — the original targets deferred; here the toon light math is
   wired into the forward light paths (`ForwardLightingCommon` grid loop, mobile clustered
   forward) so it works with `r.ForwardShading=1` and in the ES3.1 mobile preview.
2. **ES3.1 parity** — the mobile radial branch uses the desktop math verbatim, plus a
   `rcp(DistSqr + 1)` compensation because mobile's `SelShaderBxDF` lacks the capsule path's
   baked-in physical falloff. Verified pixel-identical band edges vs SM5.
3. **Mobile local-light delivery fix (engine-side)** — the UE6-reworked light-grid cull
   stage delivered zero lights into cells on `VULKAN_PCES3_1`. Root cause: the
   **specialization-constant variant** of `LightGridInjectionCS` misreads its StructuredBuffer
   SRVs on that platform. Fix (in the engine tree, `LightGridInjection.cpp`):

   ```cpp
   static inline bool UseSpecializationConstants(EShaderPlatform ShaderPlatform)
   {
       // Sel: the spec-const variant of LightGridInjectionCS misbehaves on VULKAN_PCES3_1
       // (StructuredBuffer SRV reads return zeros -> culled cells come out empty).
       if (ShaderPlatform == SP_VULKAN_PCES3_1)
       {
           return false;
       }
       return FDataDrivenShaderPlatformInfo::GetSupportsSpecializationConstants(ShaderPlatform) &&
           CVarLightGridInjectionUseSpecializationConstants.GetValueOnAnyThread() == 1;
   }
   ```

   With that gate, **stock culling works** on ES3.1 and `LightGridInjection.usf` is
   unmodified (`CULL_LIGHTS 1`). Do NOT use the old `CULL_LIGHTS 0` passthrough: it writes
   raw grid indices that resolve to the directional light's buffer slot, shading a phantom
   point light at the world origin.

## How it works (material contract)

The material uses **CLEAR_COAT** shading model; its pins are repurposed:

| Material pin        | Meaning                                            |
|---------------------|----------------------------------------------------|
| **Metallic**        | ShadowRange — the toon light's snap threshold      |
| **Tangent**         | ShadowColour — the color shown in shadowed areas   |
| (BaseColor)         | Lit toon color                                     |

Per-pixel carries:

- `ShadowColour` (Tangent pin) → `GBuffer.CustomData.rgb`
- Scene indirect → `GBuffer.PrecomputedShadowFactors.rgb` (acts as `LightmapInfo`)

Lighting per light (radial):

```
FinalLightMask = saturate(lerp(0, InvShadowRange, saturate(Diffuse * LightMask * rcp(d^2+1))))
Color = lerp(ShadowColour, BaseColour, smoothstep(0.5 - r/2, 0.5 + r/2, SurfaceShadow))
        * LightColor * FinalLightMask   (+ 20% BaseColor modulation + rim term)
```

## What's in this repo

Drop-in replacements for `Engine/Shaders/Private/` files, based on **UE 6.0**:

| File | What changed |
|---|---|
| `ShadingModels.ush` | `SelShaderBxDF` (toon BxDF) + dispatch for `SHADINGMODELID_CLEAR_COAT` |
| `DeferredLightingCommon.ush` | Sel toon light accumulation for desktop + `SHADING_PATH_MOBILE` branch, both `TotalLight` and `TotalLightDiffuse` written |
| `BasePassPixelShader.usf` | ShadowColour/indirect carries for the desktop base pass (incl. forward) |
| `MobileBasePassPixelShader.usf` | Same carries for the mobile base pass |
| `MobileLightingCommon.ush` | Restored stock UE6 signatures (mobile forward entry point) |
| `DBufferDecalShared.ush` | Decal ShadowColour via Metallic/Specular/Roughness pins |
| `LightGridInjection.usf` | **Stock / unmodified** (`CULL_LIGHTS 1`). Kept only so the folder mirrors the engine tree; the ES3.1 fix is the engine-side spec-const gate described above |

## Installation (source build)

1. Copy `Engine/Shaders/Private/*` over your UE6 source engine's `Engine/Shaders/Private/`.
2. Rebuild (shader changes hot-reload in-editor via `recompileshaders changed`).
3. Apply the `UseSpecializationConstants` gate from section 3 above to
   `Engine/Source/Runtime/Renderer/Private/LightGridInjection.cpp` and rebuild the editor —
   without it, local lights will not reach grid cells on VULKAN_PCES3_1.
4. Set your material's Shading Model to **Clear Coat** and drive the pins as above.

> These files are for a **UE 6.0** engine tree. Do **not** paste them onto other engine
> versions wholesale — signatures differ between versions and you'll get shader compile
> errors (or worse, SpirV entrypoint asserts). Port the Sel edits instead.

## Known limitations

- ES3.1 (VULKAN_PCES3_1) requires the engine-side `UseSpecializationConstants` gate
  (see section 3); a shader-only install cannot fix the light grid on that platform.
- Exponent-falloff lights: the `rcp(d^2+1)` compensation matches the desktop *inverse-square*
  integration; with `bInverseSquared=false` lights, SM5 itself uses Falloff=1, so if you want
  strict per-falloff matching, gate the compensation on `LightData.bInverseSquared`.
