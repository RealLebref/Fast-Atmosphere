# Fast Atmosphere for Nuke — Project Brief and Technical Plan

**Purpose:** A portable project brief for continuing this work in another ChatGPT conversation, with another AI agent, or with a VFX/technical collaborator.

**Status:** Planning decisions made; implementation has begun with a separate procedural terrain prototype. The first prototype code was produced after the decisions summarized here and has not yet been confirmed to compile or render in Nuke.

---

## 1. Project goal

Convert the working GLSL rendering code in the GitHub project [RealLebref/Fast-Atmosphere](https://github.com/RealLebref/Fast-Atmosphere), especially its `shadertoy` folder, into Foundry Nuke BlinkScript kernels.

The long-term goal is a reusable **Nuke Atmosphere gizmo** that applies atmospheric scattering and aerial perspective as a post-processing effect to an external renderer's CG render. The render may provide an RGBA beauty and AOVs such as depth, world position, and normal.

The original Shadertoy renders a complete procedural environment, including terrain, water, clouds, sun, moon, stars, and atmosphere. We will not reproduce every feature in the final production gizmo. Instead, we will use two related tools:

1. **Procedural Terrain / Environment Prototype — development only.** Generates a synthetic scene so the algorithms can be developed and tested without needing an external CG render. Terrain is required; water is not. It can also provide a convenient test bed for sky and celestial rendering.
2. **Atmosphere Gizmo — production target.** Takes an external CG beauty plus depth and/or position data, applies aerial perspective and atmospheric in-scattering, and optionally adds atmospheric/celestial effects. It does not need to render terrain or water.

Keep the two tools separate. Their shared parameters (for example sun position, observer altitude, planet radius, and atmosphere settings) should use matching names, ranges, units, and conventions. The user will match values manually; automatic knob synchronization is not required.

## 2. Source code and source-of-truth rules

- Always refer to the actual Shadertoy source files supplied in the `Fast-Atmosphere` repository ZIP. Do not rely solely on descriptions or assumptions.
- Keep the port as close to the original algorithms and structure as practical, because the source is working and intended to be fast and efficient.
- This is an algorithmic port, not a syntax-only translation. Adapt GLSL functions, uniforms, buffer dependencies, coordinates, and data packing to BlinkScript/Nuke as required.
- The supplied ZIP contains the Shadertoy tabs/buffers. Relevant source areas include `Common`, `Buffer A`, `Buffer B`, `Buffer C`, `Buffer D`, and `Image`.
- Do not add optimizations or redesigns before they are needed. First establish a faithful, testable baseline; then improve it based on measured results.
- Shadertoy-specific frame-to-frame feedback/caching should not be assumed necessary in a deterministic Nuke kernel. Remove it where appropriate, while preserving the underlying algorithm.

## 3. Confirmed source findings

### Buffer A — terrain data

- `Buffer A` evaluates the procedural terrain and marches camera rays to find terrain/planet intersections.
- It stores data in a packed RGBA buffer, including the ray distance in the red channel, packed normal information in the green/blue channels, and a terrain-detail/auxiliary value in alpha.
- A negative distance indicates a miss in the source's terrain-hit representation.
- Buffer A is not itself the shaded terrain beauty.

### Buffer B — screen-space visibility, shadows, and god rays

- Buffer B does **not** directly evaluate the procedural `Terrain()` or `MarchTerrain()` function.
- It reads the terrain data from Buffer A, reconstructs points along camera rays, projects candidate sample points back into the image, and compares their camera distance against sampled Buffer A depth.
- Sun-shadow visibility uses a coarse set of samples along the sun direction and screen-space depth comparisons.
- The god-ray/atmospheric-visibility calculation samples along a screen-space path toward the projected sun and checks whether sampled pixels contain a terrain hit.
- Therefore, Buffer B's depth-based logic can be adapted to an external renderer's depth/position AOVs, with changes for depth encoding, camera reconstruction, coordinate conventions, and image boundaries.
- It is a screen-space approximation, not a complete geometric visibility solution. It cannot reliably account for occluders outside the image or hidden geometry absent from the camera depth pass.
- Buffer B's shadow result in the original shader modulates the procedural terrain's direct lighting. For an external beauty, do not automatically multiply the entire beauty by the shadow matte; expose visibility separately or use it selectively.
- Remove ambient occlusion and cloud-shadow logic from the target version. Clouds are not required.
- Remove Shadertoy temporal accumulation/corner-pixel metadata unless a demonstrated need emerges.

### Buffer C — clouds

- Clouds are not required for either the initial terrain prototype or the production atmosphere gizmo. Omit cloud rendering and cloud-shadow dependencies.

### Buffer D / Image — shading and final display

- Buffer D uses Buffer A's terrain data to shade the procedural terrain. The prototype needs the relevant terrain shading, but not water rendering.
- Keep the terrain intersection/data stage distinct from the terrain beauty shading where this avoids repeating the expensive terrain march.
- Avoid baking Shadertoy-specific display tone mapping into the production compositing path. Nuke compositing should retain scene-linear HDR values.

## 4. Production gizmo: intended inputs

The production Atmosphere gizmo should support these input modes:

| Input mode | Intended use |
|---|---|
| Beauty RGBA + depth | Reconstruct camera-ray surface positions using the fixed initial camera model and depth settings. |
| Beauty RGBA + world position | Use world-space XYZ directly and derive camera-to-surface distance. |
| Beauty RGBA + depth + world position | Prefer world position for atmospheric path calculations and use depth for screen-space visibility tests/validation. |
| Optional normal AOV | Available for later normal-dependent effects, but not mandatory for the core atmospheric integration. |

Position is assumed to be a **world-space XYZ** pass, not object-space position. Position coordinates and gizmo spatial parameters must use a consistent coordinate system and scene-unit scale.

### Depth conventions and units

- Internally use **metres** for all distances.
- Provide a scene-unit menu supporting at least metres, feet, centimetres, and kilometres.
- Provide a depth-encoding menu distinguishing:
  - **Ray distance:** distance along the normalized camera ray.
  - **Camera-space Z:** distance along the camera's forward axis.
- Buffer A stores ray distance, so camera-space Z must be converted to ray distance before applying equivalent logic.
- When world position is available, derive the camera-to-surface distance from the camera and position.
- A zero depth may be used as the agreed no-surface sentinel in depth mode. Do not silently treat a zero path length as full atmosphere or no atmosphere; handle it explicitly.
- In position-only mode, world position `(0,0,0)` can be a valid surface point. Do not use that as an invalid sentinel unless explicitly configured; beauty alpha or an optional validity rule may be needed if invalid positions must be identified.

## 5. Camera and atmosphere assumptions for the initial port

The first version should reproduce the Shadertoy's original camera/atmosphere assumptions as closely as practical. Generalization can follow after a faithful baseline works.

- Use the original fixed pinhole-camera model and orientation for the first prototype.
- Assume the camera's horizontal position is zero, consistent with the original code's assumptions.
- Expose observer altitude as a knob, including useful values such as 0 m and 2,000 m.
- Expose planetary centre, planet radius, atmosphere thickness/radius, and relevant atmospheric parameters as knobs.
- Keep camera, position, depth, observer altitude, planet dimensions, and atmospheric distances consistent in units.
- Share matching knob names, ranges, and conventions between the two gizmos. They do not need to be linked.
- More general camera transforms/projections can be a later extension, not a requirement for the first faithful port.

## 6. Sky, sun, moon, eclipse, and stars

The desired long-term feature set includes:

- Generated sky/atmospheric radiance.
- Configurable sun direction and sun disk, with an enable control for the visible disk.
- Moon rendering.
- Eclipse from the relative apparent positions/sizes of the sun and moon.
- Stars.
- Optional atmospheric god rays/sun shafts based on the adapted Buffer B visibility technique.

The user has clarified that the production tool does **not** need a separate sky-rendering stage. The intended operation is atmosphere applied over the incoming RGB. For a pixel with no actual surface, the atmosphere can contribute against a zero-RGB pixel; there is no requirement for a separate background-mask input or separate sky-generation pass.

However, the implementation must distinguish a valid surface distance from the chosen no-surface sentinel. If the input depth uses zero for no surface, the atmosphere should integrate along the viewing ray to the atmospheric boundary for that pixel, rather than treating zero as a zero-length path. In position-only mode, validity cannot be inferred from `(0,0,0)` alone.

Celestial features can be optional contributions in the same image-based rendering path. Ensure stars and the moon/sun are behind foreground CG surfaces when compositing them; do not blindly overlay them on every foreground pixel. This visibility/compositing behavior needs an explicit implementation decision during production-gizmo development.

## 7. Atmospheric compositing model

The basic surface compositing relationship is:

`C_out = T(d) * C_CG + L_scatter(d)`

where:

- `T(d)` is transmittance along the camera-to-surface path.
- `C_CG` is the input scene-linear beauty color.
- `L_scatter(d)` is the accumulated in-scattered atmospheric radiance along that path.

For pixels without a surface, integrate to the atmosphere boundary instead of using a nonexistent surface endpoint. Preserve scene-linear HDR values and establish an explicit alpha policy. Avoid applying atmosphere twice to renders that already include it.

The core atmosphere model should be ported from the supplied source before making substantial physical-model changes. Screen-space god rays/shadow visibility should be treated as a separate, optional effect rather than conflated with the primary aerial-perspective integration.

## 8. Procedural terrain prototype: required outputs

The development prototype should render terrain only (no water) and produce a visible RGBA beauty plus auxiliary channels needed for testing.

Minimum desired outputs:

- `rgba`: shaded procedural terrain beauty.
- `depth.Z`: ray distance in metres, consistent with Buffer A's ray-distance convention. Define a stable miss/sentinel convention and document it.
- `P`: world-space position XYZ in metres, where valid.
- `N`: world-space normal XYZ, where valid.

Use Nuke's multilayer/channel workflow so these passes are accessible from the same output stream. Each BlinkScript kernel has one output image; a Nuke Group may combine multiple kernels and use channel Copy/Shuffle operations to assemble the passes.

The terrain-data kernel should perform the expensive intersection/march once, and subsequent beauty/depth/position/normal operations should reuse its output wherever practical. Do not repeat terrain raymarching independently for every pass unless there is a specific technical reason.

## 9. Step-by-step plan from start to finish

### Phase 1 — Source audit and conventions

1. Map the original Shadertoy tabs/buffers and their dependencies.
2. Identify reusable functions and data layouts in `Common`, `Buffer A`, `Buffer B`, `Buffer C`, `Buffer D`, and `Image`.
3. Document camera, coordinate, depth, units, and output conventions.
4. Create a parameter contract shared by the prototype and production gizmo.

### Phase 2 — Procedural terrain prototype

5. Port Buffer A's terrain intersection/data generation into BlinkScript.
6. Produce and validate terrain ray distance and packed normal/detail data.
7. Port the relevant Buffer D terrain shading to create a visible beauty.
8. Expose basic camera and terrain quality settings; no water or clouds.
9. Extract/reconstruct `depth.Z`, `P`, and `N` and validate their values and channels in Nuke.
10. Test low quality settings first, then compare silhouette and appearance against the original Shadertoy.

### Phase 3 — Environment and celestial rendering

11. Port the common atmosphere and sky calculations while preserving original assumptions.
12. Implement sun direction and sun disk.
13. Implement moon and eclipse geometry.
14. Implement stars.
15. Validate the generated environment against the original source, using matching camera/sun parameters.

### Phase 4 — Buffer B adaptation

16. Isolate the god-ray/atmospheric-visibility calculation from the sun-shadow logic.
17. Replace Buffer A depth reads with the prototype depth image.
18. Remove AO, cloud shadows, and unnecessary temporal feedback.
19. Validate screen-space projections, depth comparisons, screen boundaries, and occlusion limitations.
20. Keep the resulting shadow/visibility information separate from the beauty by default.

### Phase 5 — Production atmosphere core

21. Implement depth/position input modes and unit/depth-encoding conversions.
22. Integrate atmospheric transmittance and in-scattering along the camera-to-surface path.
23. Handle no-surface rays by integrating to the atmosphere boundary.
24. Add optional depth-aware god rays and screen-space visibility effects.
25. Validate with the prototype's synthetic beauty/AOVs before requiring real Arnold/RenderMan renders.

### Phase 6 — Assemble, validate, and optimize the gizmo

26. Add the production knobs with matching names and conventions.
27. Establish RGBA, alpha, scene-linear, and sky/celestial compositing behavior.
28. Add diagnostic outputs: depth, position, normal, transmittance, atmosphere contribution, and visibility as practical.
29. Test near/far surfaces, silhouettes, zero/missing depth, invalid position, unit conversions, and off-screen occlusion cases.
30. Profile BlinkScript sampling and raymarch loops; expose quality controls and optimize only after correctness is established.
31. Package the production kernel(s) into a reusable Nuke gizmo and document input requirements.

## 10. Working method and collaboration guidelines

- Work in small milestones and validate each stage before moving on.
- Before coding a new stage, inspect the corresponding original source functions.
- Keep changes minimal and localized where possible; avoid large refactors or arbitrary renaming.
- Preserve original algorithmic structure unless a documented BlinkScript/Nuke limitation requires a change.
- When a shader does not appear to work, add diagnostic outputs and isolate the failure instead of immediately expanding the architecture.
- Clearly distinguish between code that has been inspected, code that has been written, and code that has actually compiled and rendered in Nuke.
- Do not claim a kernel compiles until it has been tested in the target Nuke version.
- Use the procedural prototype to validate the production path without waiting for an external render.
- Record parameter changes and any departures from the original shader.

## 11. Immediate next steps

The first coding milestone is the **procedural terrain prototype**, not the production Atmosphere gizmo.

1. Validate the first prototype code in the target Nuke installation.
2. Fix BlinkScript compile errors and verify the terrain image.
3. Verify that `depth.Z`, `P`, and `N` contain correct and consistent data.
4. Compare terrain silhouette and shading with the original Shadertoy.
5. Once the prototype works, begin the sky/sun/moon/stars port, then adapt Buffer B using the prototype depth output.

The first prototype package has been generated, but it has **not yet been confirmed to compile or render in Nuke**. Treat it as a candidate implementation requiring validation, not as a verified working baseline.

---

## Key decisions at a glance

- Two tools: a development-only procedural terrain/environment prototype and a production Atmosphere gizmo.
- Terrain prototype: terrain required; water and clouds omitted.
- Production input: CG beauty plus depth and/or world-position AOV; normal is optional.
- Buffer B is screen-space depth reprojection, so its visibility method is adaptable to renderer AOVs.
- Screen-space shadows are approximate; no requirement for full scene-accurate shadows.
- Ambient occlusion and cloud rendering are omitted.
- Original camera model/orientation and atmospheric assumptions are preserved initially.
- Distances are internally converted to metres; unit and depth-encoding menus are required.
- Shared controls match manually between the two tools.
- Production atmospheric composite: `T * beauty + in-scattering`.
- No separate production sky-rendering stage is required; no-surface pixels integrate to the atmospheric boundary and can receive atmospheric radiance over zero RGB.
- Port the source faithfully first, validate in Nuke, then optimize and generalize.
