# Project 4 Part 4


### 1) Introduction and Goal
The goal of this project is to investigate how 3D Gaussian Splatting (3DGS) can be integrated into a modern game engine workflow — with Unreal Engine 5 (UE5) as the primary platform — in the context of AR rendering, and to evaluate whether such an integration is practically usable on Android, the most widely deployed mobile AR platform. Besides, the project re-implements **MobileGS** (ICLR 2026.3), a recently proposed 3DGS variant that redesigns the rendering (inferring) to target mobile GPUs, and ports its core pipeline into UE5 so that Gaussian-splat scenes can co-exist with traditional rendering pipeline (mesh and AR). In Mobile-GS paper, they claimed that their framework achieved 120 FPS in rendering 3DGS on mobile devices. Given the performance challenge on rendering 3DGS on mobile devices, it is nontrivial to integrate MobileGS
  
The central research question therefore is:
> *Under the real constraints of a UE5 Android AR build — how much of the photorealistic quality and performance of 3DGS can be preserved, and on what extent would MobileGS acceleration contribute to close the gap?*

The project delivers three coupled contributions:

1. **A UE5-native 3DGS renderer** that consumes standard `.ply` splat assets and composites them with UE5's mobile renderer and ARCore passthrough.
2. **A re-implementation of MobileGS's core acceleration ideas** adapted from the paper's reference CUDA code into UE5's RDG + mobile shader stack.







### 2) System Design and Implementation
#### Platform and stack
- Engine: Unreal Engine 5.7 + GoogleARCore plugin + ARUtility plugin
- Target hardware: Windows PC for feasibility verification and Android phones for primary AR testing
- Hardware detail: PC (with RTX 3060 laptop), Android Phone (Samsung Galaxy S23+)
#### 3DGS 
Initially (in part 3), the project is built on top of the open-source **GaussianSplattingForUnrealEngine** plugin, extended (added SH evaluation support for better visual quality) to suit our AR / mobile evaluation goals. The core of the system is a Niagara-based 3DGS renderer that consumes `.ply` files produced by an upstream 3DGS training pipeline and draws them inside a standard UE5 scene, composited with conventional triangle-mesh content.
  
 When a `.ply` is loaded, the plugin parses it and pushes the per-Gaussian attributes (position, scale, rotation quaternion, opacity, SH color) into a Niagara emitter. The emitter allocate one Niagara particle per Gaussian and copy the raw attributes onto it. On every frame, an `UpdateSpriteSizeAndRotation` compute view data for current camera pose. Data were then passed to NiagaraSpriteRenderer (i) globally sorts the particles back-to-front , (ii) expands each particle into a camera-facing billboard quad, (iii) rasterizes the quads through the hardware rasterizer, and (iv) evaluates the 2D Gaussian footprint inside the pixel shader (custom material blueprint). Front-to-back compositing is delegated to the fixed-function ROP — it is **not** a shader-side loop.
![system architecture](./3DGSUE.png "system architecture")
The Niagara-based pipeline is *not* a fully faithful implementation of the original 3DGS rasterizer. The reference CUDA implementation (Kerbl et al., SIGGRAPH 2023) performs **software rasterization**: every Gaussian is duplicated across the tiles it overlaps, the tile-splat pairs are radix-sorted by `(tileID, depth)`, and inside each pixel a hand-written serial loop iterates the tile's sorted Gaussian list and accumulates. In most of the UE5 build, each Gaussian becomes an independent **sprite proxy**, and the per-pixel serial α-blend loop disappears — the GPU's hardware rasterizer handles fragment generation, and the ROP handles ordered alpha compositing. The trade-offs are visible up-front and drive the evaluation.
  
The original 3DGS rasterizer is written in software not because software rasterization is intrinsically better, but because the entire pipeline has to be *differentiable*. That is what forces the hand-written per-pixel loop — the ROP of pixel shader has no gradient. In the mobile-AR deployment target, however, swapping to hardware rasterization becomes the rational choice — it is simpler to integrate into an engine.
  
The rendering result is mathematically correct since they use same blending equation, while the capturing area of Gaussian may differ due to the gap between software and hardware resterization.


#### MobileGS Reproduce and Reimplementation
**Mobile-GS** (Du et al., ICLR 2026) is a latest improvement to 3DGS that reaches 116 FPS at 1600 × 1063 on a Snapdragon 8 Gen 3, on rendering 3DGS models. The paper identifies the **depth-sorting step of traditional alpha blending as the dominant bottleneck** (up to 50% of total frame time on the 3DGS CUDA rasterizer) and contributes four jointly-designed components to remove it. It proposed **Depth-aware order-independent rendering.** Replaces sort + sequential α-blending with an order-free weighted accumulation `C = (1−T) · Σ cᵢαᵢwᵢ / Σ αᵢwᵢ + T·c_bg`, where `wᵢ = φᵢ² + φᵢ/dᵢ² · exp(s_max/dᵢ²)` down-weights far Gaussians and up-weights near ones. In their implementation, αᵢ and φᵢ both come from a per-gaussian MLP output. For this project, Mobile-GS is particularly attractive as the re-implementation target because the claimed improved rendering performancing with no decrease in quality. 
  
Niagara itself was designed as a *general-purpose VFX system* — per-emitter scripting, interpreter overhead, spawn/kill bookkeeping, lifetime tracking, and the full attribute binding graph all run on every particle every frame, even though a static 3DGS scene needs none of them. On top of that, the `NiagaraSpriteRendererProperties` path is difficult to extend.
  
For the actual Mobile-GS implement we therefore switch the base from the Niagara plugin to the **NanoGaussianSplatting (NanoGS)**. NanoGS follows the same quad-proxy idea (one camera-facing quad per Gaussian, hardware rasterization, fragment-shader 2D Gaussian evaluation), but does the work inside a **custom post-processing render pass** rather than inside Niagara.


![system architecture 2](./3DGS_png.png "system architecture 2")

We first reproduce Mobile-GS's training pipeline to obtain the distilled assets — the compressed `.ply` (1st-order SH, VQ-quantized, contribution-pruned) paired with the trained view-enhancement MLP weights. We then extend NanoGS in UE5 so its render pipeline consumes these assets directly: a new `FGaussianMLPWeightsAsset` is loaded at asset-import time, and on every frame an MLP forward pass (`DispatchMLPForward`) runs per visible Gaussian to produce the per-splat `(φ, o)` pair that feeds the depth-aware weight `w = φ² + φ/d² + exp(s_max/d)` and alphas consumed by the OIT raster stage described below. The rest of the NanoGS pipeline — cluster culling, compaction, hardware rasterization — is reused unchanged.
  
To verify the correctness of our UE5 RDG implementation of Mobile-GS components, we implemented a "unit test" module to ensure the compute shader MLP generate identical output with the official implementation under given camera pose. For verifying the OIT rendering, we extended graphdeco-inria/gaussian-splatting/SIBR_viewers for a Mobile-GS interactive viewer and manually check the visual similarity with similar camera poses between UE and SIBR_viewers. Pixel-wise image compare or image reconstruction metrics are not suitable for it since it is defficult to share the same background color in both scenario.

### 3) Build Instructions
For GaussianSplattingForUnrealEngine: Install UE 5.7.4, place plugin in UE projects' **Plugins** directory and compile the project with Visual Studio. In Unreal editor, click "Import" to import 3DGS ply files.
  
For NanoGaussianSplatting: Install UE 5.7.4, place Plugins/NanoGS in UE projects' **Plugins** directory, and compile the project Visual Studio. In Unreal editor, click "Import" to import 3DGS ply files or mlp_weights.mgsbin generated by Mobile-GS/export_mlp_weights.py. Run `r.GaussianSplat.UseOIT 0` or `r.GaussianSplat.UseOIT 1` in Unreal Editor command line to alter between 3DGS and Mobile-GS rendering. 
  
For Mobile-GS: run pretrain.py / train.py with same argument from readme of its original repos to get 3DGS/Mobile-GS data (C++ compiler + NVCC required to install the dependency).
  
For MobileGS data, run `python export_mlp_weights.py [opacity_phi_nn.pt path]` to consume the "opacity_phi_nn.pt" to generate .mgsbin file required by NanoGS. Run `python bake_for_ue5.py --source [MobileGS ply dir]` to fetch the base color of Mobile-GS data and generate a new ply file to be imported by NanoGS. (color was not stored in raw output .ply files since they are also encoded with MLP in Mobile-GS's official implementation)


### 4) Metrics and Results
We evaluate the implementation along two axes, deliberately chosen to match an **inference-only AR deployment**:

1. **On-device performance** — measured on both PC and Android target introduced above. We report end-to-end **FPS** plus a per-stage **render-time breakdown** so that the contribution of each Mobile-GS component can be isolated.
2. **AR compositing realism** — a subjective read on how well the rendered 3DGS content "fits" in the live camera feed.

We **do not** report PSNR, SSIM, or LPIPS. Those metrics assume a held-out test view from a *training* pipeline.

**Test scenes.** 2 `.ply` assets, chosen to span the two regimes a mobile AR application is most likely to encounter:

| Scene | Source | Regime |
|---|---|---|
| `play_room` | 3DGS model trained on the **Playroom** scene (Tanks & Temples + Deep Blending benchmark suite) | **Indoor scene** | 
| `cluster_fly` | Artist-authored 3DGS model (downloaded, not trained in-house) | **Single-object scan** |





  
In reproducing Mobile-GS we noticed that its headline FPS numbers appear to exclude a substantial portion of the per-frame work. Although the method relies on several MLPs — and crucially on a view-dependent enhancement MLP that must run **every frame** — the benchmarking code in the official repository times only the CUDA rasterization kernel that consumes the already-computed `(φ, α)`, not the MLP forward pass that produces them. In our local reproduction this matches: measuring only the post-MLP kernel on the same RTX-class GPU gives ~600 FPS, matching the paper's 1098 FPS claim, whereas an end-to-end timing that includes the per-frame MLP forward is dominated by the MLP itself. The project's Github issue page indicates that the authors hold an internal Vulkan build with unannounced MLP inference optimizations withheld for company-policy reasons; without access to or an implementation of that optimization, our UE5 implementation cannot further improve the rendering performance.



## References 
- 3DGS (SIGGRAPH 2023)
  - Paper: [https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_high.pdf](https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/3d_gaussian_splatting_high.pdf)
  - Repository: [https://github.com/graphdeco-inria/gaussian-splatting](https://github.com/graphdeco-inria/gaussian-splatting)
- Mobile-GS (ICLR 2026)
  - Paper: [https://arxiv.org/abs/2603.11531](https://arxiv.org/abs/2603.11531)
  - Repository: [https://github.com/xiaobiaodu/Mobile-GS](https://github.com/xiaobiaodu/Mobile-GS)

- UE 3DGS repositories as engineering references:
  - [https://github.com/Italink/GaussianSplattingForUnrealEngine](https://github.com/Italink/GaussianSplattingForUnrealEngine)
  - [https://github.com/TimChen1383/NanoGaussianSplatting](https://github.com/TimChen1383/NanoGaussianSplatting)
