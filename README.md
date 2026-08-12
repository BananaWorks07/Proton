# Introduction

**Proton-EM** is a fork of [Proton](https://github.com/valvesoftware/proton) with a modified version of Wine, DXVK, and other submodules to implement the following features:

**NOTE:** This README only applies to Proton-EM 11.0 (or derivatives)

- Video codec fixes
    - Thanks to (in no paticular order) the CodeWeavers team, LoathingKernel, GloriousEggroll, NelloKudo, and many others for their work on getting videos working with Proton.
- Protonfixes and UMU support
    - Thanks to GloriousEggroll and LoathingKernel for providing a reference on how to add this to the Makefile.
- FSR 4.1.1 (and prior) support, including redstone features like machine learning frame generation (MLFG)
    - FSR4-I8 upscaling support on RDNA2/3, FSR4-FP8 + redstone on RDNA4
    - Anti-Lag 2 on D3D12 games (Upstreamed in Proton experimental)
- Improved implementation of AMD Driver DLLs `amdxc64.dll`, `atiadlxx.dll` and `atidxx64.dll` to allow native AMD GPU Services to function under Proton.
- winewayland support with various improvements compared to upstream Wine
    - EDID emulation support (with CTA-861 HDR static metadata extension for HDR)
    - Color accurate windows HDR using `windows_bt2100` and `windows_scrgb` image descriptions.
        - Planned work: Implement `VK_EXT_swapchain_colorspace` using these image descriptions to avoid any reliance on drivers. (Allows HDR on Nvidia r580 without any Vulkan Layers)
    - Server side decorations support
    - Moving fullscreen windows between different outputs
    - Heuristic to determine which monitor is most likely the primary one
    - Fractional scaling protocol support (Upstreamed in wine 11.12)
    - Window activation support through `xdg-activation-v1`
    - Locked modifier key support
    - Improved keyboard layout support, dead key support, and various other keyboard fixes
    - Pointer warp protocol support (Now upstreamed by someone else in Wine 11.9)
    - Pointer rawinput support (Upstreamed in wine 11.13, thanks Rémi Bernon)
    - Touchpad scrolling support
    - Touch input support
    - Window minimization and focus loss support
    - Improved window tiling support
    - xdg-popup support
    - Explicit fullscreen support
    - Hack to move windows entirely on screen (resolves being unable to interact with portions of windows)
    - Wayland client surface caching (fixes DMABUF modifier feedback loops when playing some Vulkan games with direct scanout)
    - Various bug fixes
    - All of this is being upstreamed right now! (I'll add links to MRs later)
    - Not Planned: Cross Process Rendering, Client Surface Composition (afaik these are being worked on upstream)
- Automatic HDR enablement for DXVK (DXGI) when the display supports it
- Some ntoskrnl patches for games that use AntiCheatExpert
- Resolve Adapter LUID collisions on multi-GPU systems.
    - You can run whatever Proton version you like after the collision is resolved because the LUID are stored in the registry.
- Various other patches to fix bugs or improve performance

This project aims to provide a good user experience with the end goal of no longer needing to exist due to its code being in the respective upstream sources. A small number of the above features have already been upstreamed, with plans on being upstreaming more in the future.

**Wayland protocol requirements for winewayland:**  
Some protocols are required, some are optional, some are highly recommended (things will explode if you don't have them). 
There has never been a definitive list for what is required for winewayland so here is one. 
The numbers in each box represent the version number of each protocol.

| Protocol | Required | Recommended | Optional | Notes |
| :---------------- | :--- | :--- | :--- | :-------------------------------------------- |
| `wl_seat`       | 3 | 5 | &nbsp; | `wl_seat::release`. Upstream wine optionally uses up to version 8 for high precision mice. Unfortunately, enabling that causes regressions in some apps that weren't designed for high precision mice so this has been reduced to version 5. |
| `wp_viewporter` | 1 | &nbsp; | &nbsp; | &nbsp; |
| `wl_compositor` | 4 | &nbsp; | &nbsp; | &nbsp; |
| `wl_subcompositor` | 1 | &nbsp; | &nbsp; | &nbsp; |
| `xdg_wm_base`   | 3 | &nbsp; | &nbsp; | `xdg_popup::reposition` |
| `wl_shm`        | 1 | &nbsp; | &nbsp; | &nbsp; |
| `wl_output`     | 3 | &nbsp; | &nbsp; | `wl_output::release` |
| `wl_fixes`      | &nbsp; | &nbsp; | 2 | `wl_fixes::ack_global_remove` | 
| `zxdg_output_manager_v1` | &nbsp; | 2 | 3 | It may become a requirement later (when a wl_output is present?) |
| `wl_data_device_manager` | 2 | &nbsp; | &nbsp; | If you want clipboard support of any kind |
| `ext_data_control_manager_v1` | &nbsp; | 1 | &nbsp; | This protocol is considered privileged. If possible, I hope to eventually make everything work on just `wl_data_device_manager` |
| `xdg_toplevel_icon_manager_v1` | &nbsp; | &nbsp; | 1 | Window icons |
| `wp_fractional_scale_manager_v1` | &nbsp; | &nbsp; | 1 | May be phased out in the future in favor of implementing windows per monitor DPI scaling. |
| `zxdg_decoration_manager_v1` | &nbsp; | &nbsp; | 2 | Version 2 is required for server side decorations. |
| `wp_color_manager_v1` | &nbsp; | &nbsp; | 3 | Required for HDR. With support for `windows_scrgb` and `windows_bt2100` image descriptions for accuracy. winewayland utilizes the image descriptions of each wl_output rather than the preferred one. | 
| `xdg_activation_v1` | &nbsp; | 1 | &nbsp; | Useful for `FlashWindow` (in Proton-EM) and `ActivateWindow` |
| `wp_alpha_modifier_v1` | &nbsp; | 1 | &nbsp; | Required for full window alpha on SLWA/ULW windows. The alpha modifier protocol is not worded properly. The alpha multiplier must also apply to subsurfaces for it to work the way winewayland expects. |
| `wp_cursor_shape_manager_v1` | &nbsp; | &nbsp; | 1 | Required to use the host system's cursor theme when possible | 
| `zwp_relative_pointer_manager_v1` | &nbsp; | 1 | &nbsp; | Required for mouse rawinput. |
| `zwp_pointer_constraints_v1` | &nbsp; | 1 | &nbsp; | Required for mouse confinement/locking. |
| `zwp_text_input_manager_v3` | &nbsp; | &nbsp; | 1 | Required for IME support. |
| `wp_pointer_warp_v1` | &nbsp; | 1 | &nbsp; | Recommended to implement this to allow for SetCursorPos without hacks. |
| `wp_fifo_v1` | &nbsp; | 1 | &nbsp; | Not used by winewayland directly, recommended for WSI to function correctly. FIFO implementation needs to be able to handle roleless surfaces (i.e prevent them waiting forever when a fifo barrier is set) |
| `wp_presentation` | &nbsp; | 1 | &nbsp; | Not used by winewayland directly, recommended for WSI to function correctly. Required for `VkWaitForPresentKHR` to work as intended. Also ensure that the compositor sends `wp_presentation::discarded` for every commit (when a buffer is attached) with a roleless surface. | 
| `zwp_linux_dmabuf_v1` | &nbsp; | 5 | &nbsp; | Not used by winewayland directly. The specific version doesn't matter much, but DMA-BUF is required for good performance while rendering | 
| `wp_tearing_control_manager_v1` | &nbsp; | &nbsp;  | 1 | Not used by winewayland directly. Tearing helps reduce latency with immediate presentation mode. |

Driver Requirements: Mesa 26.2 (or higher).  
The Nvidia driver has a bunch of issues but if you have to use it then use the r595 or higher.  
Recommended Compositor: KWin 6.7

When using the winewayland driver, before reporting a bug please consult the list of known compositor bugs: https://github.com/Etaash-mathamsetty/Proton/issues/82

**Tag Naming Methodology:**

When working on Proton-EM I choose to name tags in a certain way:

| Tag | Description                               |
| :--------------------- | :---------------------------------------  |
| `EM-11.0-nightly`      | Nightly build, treat like Proton Experimental bleeding-edge |
| `EM-11.0-(i)-alpha(j)` | Known issues, still some missing features | 
| `EM-11.0-(i)-beta(j)`  | All features present, either known issues or needs testing |
| `EM-11.0-(i)-rc(j)`    | Basically ready for release. Probably going through rebase and testing |

## Contribution Policy

The usage of AI or Large Language Models (LLM) is not allowed to generate or assist in generating code for Proton-EM. This decision is not one I made, but is required for me to enforce to allow the code in this project to make it to upstream sources. Note that using LLMs to look through logs/find potiential issues in logs is fine (I will probably just ignore it so don't waste your tokens). 

TLDR: I may ignore your PR if I have suspicion that it was written by an LLM.

## Environment Variables

| Compat config string  | Environment Variable               | Description  |
| :-------------------- | :--------------------------------- | :----------- |
| `wayland`             | `PROTON_ENABLE_WAYLAND=1`,`PROTON_USE_WAYLAND=1`  | Enables the winewayland driver |
| `sdlinput`            | `PROTON_USE_SDL=1`                                | Uses SDL2/3 for gamepad input instead of steam input (or hidraw) |
| `nativeags`           | `PROTON_NATIVE_AGS=1`                             | Uses AMD's proprietary AMD GPU services binary instead of Proton built-in one and also enables useage of statically linked AMD GPU services. Useful to enable HDR in some UE4 DX11 games (on AMD). | 
|                       | `PROTON_EMULATE_STEAMINPUT=1`                     | Emulate Steam Input hooks with SDL input. This does not emulate the ISteamInput interface used by a handful of games on Steam (https://partner.steamgames.com/doc/api/isteaminput). Emulation for that interface is beyond my scope, since it would just be easier to hack steam overlay to make it work on wayland native apps. |
|                       | `FSR4_UPGRADE=1`                                  | Upgrade FSR3/4 to newer FSR4 (FP8 or I8) using AMD's FSR 4.1.1 amdxcffx64.dll taken from Proton Experimental. This option is not needed on RDNA2-4 discrete GPUs |
|                       | `MLFG_UPGRADE=1`                                  | Enables FSR4 MLFG upgrade to use redstone frame generation. Can be used in tandom with FSR4-I8 on RDNA3 using `DXIL_SPIRV_CONFIG=wmma_rdna3_workaround`. |
|                       | `FSR_WATERMARK=1`, `FSR_FG_WATERMARK=1`           | Enables watermarks for FSR Upscaling and FG. FG watermark is incorrect on 4.1.1 (reports MLFG even when FP8 is not supported). Upscaling watermark works correctly. You can use this to determine which FSR3/4 type you are using (I8 or FP8) |
|                       | `WAYLANDDRV_PRIMARY_MONITOR`                      | Tell the wayland driver what the primary monitor name is (Example: `eDP-1`). This is a workaround for a missing Wayland Protocol. |
| | `WAYLANDDRV_SSD=0` | Disable winewayland server side decorations implementation for debugging. |
| | `WAYLANDDRV_IME=1` | Enables IME support. It is only disabled by default because many games don't even try to support IME properly, which causes issues for wayland. However, please enable it if you know that you need this feature and the game supports it. |
| | `WAYLANDDRV_SHORTCUT_INHIBIT=1` | Inhibit keyboard shortcuts on fullscreen apps. I use this on KDE when playing Overwatch to avoid KRunner from opening when I press `alt+space` |
| | `WAYLANDDRV_RAWINPUT=` | Adjust rawinput sensitivity, you can use any value that is in the positive real numbers. Pass in 0 to disable rawinput completely. Primarily intended for debugging. |
| | `DXVK_NO_HDR=1`    | Disables automatic HDR support for debugging. |

Please check the [original Proton README](README-ORIG.md) for the launch options provided by upstream Proton.
