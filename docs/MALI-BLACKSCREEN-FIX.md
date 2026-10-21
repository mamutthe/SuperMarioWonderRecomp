# ARM Mali GPU Black-Screen Fix (Android)

## Overview

Before, every graphics pipeline failed to compile on Mali GPUs because the offline-translated shaders unconditionally declared capabilities and built-ins that Mali does not support (`SPV_EXT_fragment_shader_interlock` in fragment shaders, vertex-stage `GroupNonUniform*` subgroup capabilities, and `gl_ClipDistance`/`gl_CullDistance` in `gl_PerVertex`), while `VkDevice` creation omitted `VkPhysicalDeviceVulkan11Features` (`shaderDrawParameters`) and left `-1..1` clip-space Z unmapped without `VK_EXT_depth_clip_control`.

Now, `nvn_spirv_patch.cpp` sanitizes SPIR-V modules at load time by stripping those unused capabilities, extensions, and `gl_PerVertex` clip/cull members, `nvn_vk.cpp` chains Vulkan 1.1 and 1.2 feature structs during device creation, and `nvn_spirv_clip.cpp` automatically enables vertex clip-Z remapping whenever `VK_EXT_depth_clip_control` is absent.

---

## Symptoms Before the Fix

Tested on **POCO X6 Pro** (MediaTek Dimensity 8300-Ultra, **ARM Mali-G615 MC6**, Vulkan 1.3, driver `v1.r44p1`):

- Game audio and UI input worked normally, and the swapchain present pass succeeded, but the game screen remained completely black.
- `nx.log` reported `vkCreateShaderModule` succeeding (ARM Mali defers SPIR-V validation and compilation until pipeline creation), followed by 100% of graphics pipelines failing in `vkCreateGraphicsPipelines`:
  ```text
  nvn/vk: pipeline for d0c055be04d18268 + 38edabcdcbebb338 failed
  ...
  nvn draws: 25577 total, 0 done, 0 no program, 0 no vertex shader, 25577 no pipeline, 0 no indices
  ```

---

## Root Causes

1. **Unconditional `FragmentShaderPixelInterlockEXT` in Fragment Shaders**
   - All 7,839 offline-translated fragment shaders declared `OpCapability FragmentShaderPixelInterlockEXT` (5378) and `OpExtension "SPV_EXT_fragment_shader_interlock"`, even though 0 shaders in *Super Mario Bros. Wonder* emit `OpBeginInvocationInterlockEXT` / `OpEndInvocationInterlockEXT`.
   - ARM Mali does not support `VK_EXT_fragment_shader_interlock` and rejected every fragment shader during `vkCreateGraphicsPipelines`.

2. **Unconditional `GroupNonUniform*` Capabilities in Vertex Shaders**
   - All 12,751 translated shaders declared `OpCapability GroupNonUniformVote` (62), `OpCapability GroupNonUniformBallot` (64), and `OpCapability GroupNonUniformShuffle` (65), although only 3 compute shaders actually use subgroup operations.
   - On Mali-G615, `subgroupProperties.supportedStages` is `0x3f30` (fragment, compute, and raytracing stages, excluding `VK_SHADER_STAGE_VERTEX_BIT`).

3. **Unconditional `ClipDistance` and `CullDistance` Members on `gl_PerVertex`**
   - Every translated vertex shader declared `gl_PerVertex` with 4 members (`Position`, `PointSize`, `ClipDistance`, `CullDistance`), even though no shader in the game writes `gl_ClipDistance` or `gl_CullDistance`.
   - Mali-G615 reports `shaderClipDistance = 0` (`maxClipDistances = 0`) and `shaderCullDistance = 0` (`maxCullDistances = 0`).

4. **Unchained `VkPhysicalDeviceVulkan11Features` / `VkPhysicalDeviceVulkan12Features`**
   - `nvn_vk.cpp` chained `VkPhysicalDeviceVulkan13Features` into `VkPhysicalDeviceFeatures2`, but omitted `VkPhysicalDeviceVulkan11Features` and `VkPhysicalDeviceVulkan12Features`.
   - Under Vulkan 1.3 `pNext` feature activation rules, omitting `VkPhysicalDeviceVulkan11Features` left `shaderDrawParameters` disabled on the `VkDevice`, causing vertex shaders that use `OpCapability DrawParameters` (`BaseInstance` / `BaseVertex`) to be rejected.

5. **Missing Automatic Clip-Space Z Remapping (`-1..1` to `0..1`)**
   - NVN depth mode 0 places clip-space $z$ in $[-w, w]$, whereas Vulkan expects $[0, w]$ unless `VK_EXT_depth_clip_control` is enabled.
   - Mali-G615 does not support `VK_EXT_depth_clip_control`, and `depth_remap()` (`patch_clip_z`) previously required `NX_CLIP_Z_PATCH=1` to be set manually.

6. **Non-Solid Polygon Mode Guard**
   - Mali-G615 reports `fillModeNonSolid = 0`; `rs.polygonMode` now falls back to `VK_POLYGON_MODE_FILL` when `fillModeNonSolid` is not supported by the device.

---

## Code Changes

| File | Change |
| :--- | :--- |
| `runtime/src/nvn/nvn_spirv_patch.cpp` | Added `sanitize_module()` to `patched_spirv()` to strip unused `FragmentShader*InterlockEXT` capabilities and `SPV_EXT_fragment_shader_interlock`, strip unused `GroupNonUniform*` and `DrawParameters` capabilities, and trim unused `ClipDistance`/`CullDistance` members from `gl_PerVertex`. |
| `runtime/src/nvn/nvn_vk.cpp` | Chained `VkPhysicalDeviceVulkan11Features` and `VkPhysicalDeviceVulkan12Features` into `VkPhysicalDeviceFeatures2` during `VkDevice` initialization. |
| `runtime/src/nvn/nvn_spirv_clip.cpp` | Enabled `depth_remap()` by default whenever `!vk_depth_clip_control()` (unless explicitly overridden via `NX_CLIP_Z_PATCH=0`). |
| `runtime/src/nvn/nvn_pipeline.cpp` | Guarded `rs.polygonMode` with `vk_features().fillModeNonSolid` and logged the `VkResult` code if `vkCreateShaderModule` or `vkCreateGraphicsPipelines` fails. |
