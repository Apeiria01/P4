# Project 4 Part 2: Project Proposal and Technical Specification

## Project Goal
The goal of this project is to investigate how 3DGS can be integrated into a game engine workflow (with UE5 as the primary platform), re-implement a latest accelerated 3DGS framework in game engine context, and to evaluate whether this integration is practically usable on mobile devices, which are the most common AR deployment platform. 
This project is interesting because most 3DGS systems currently demonstrate strong quality on desktop GPUs, while mobile AR has much stricter constraints. Therefore, the core question is: how much of 3DGS photorealistic rendering can be preserved under real mobile AR constraints?

## Technical Approach
### 1) Platform and stack
- Engine: Unreal Engine 5

- Target hardware: PC for feasibility verification and Android phones for primary  AR testing



### 2) Engineering references (existing UE repositories)
Implementation will reference existing UE 3DGS repositories:
- `Italink/GaussianSplattingForUnrealEngine` 
- `xverse-engine/XScene-UEPlugin` 

### 3) AR integration tasks
- Align 3DGS content to AR world coordinates (initial placement, scale calibration, re-localization stability).
- Drive rendering with live camera pose updates.
- Integrate latest **Mobile-GS** improvement to better support mobile enviroment. 




### Contingency plan
- If mobile performance is insufficient: preserve AR anchoring + core rendering chain, then aggressively simplify rendering complexity
- If the plugin path is not portable enough: implement a minimal custom loading/rendering path for experiment validity



### A Short Technical Report

#### What Is Currently Implemented

The original project goal was to reproduce **3DGS** and **MobileGS** in **Unreal Engine on mobile** and then evaluate **MobileGS**, whose main contribution is reducing the sorting cost in the 3DGS rendering pipeline. In the original 3DGS pipeline, this sorting stage accounts for more than 25% of the total rendering time.

So far, I have reproduced a **3DGS-style renderer in UE5** based on an existing plugin. Since the plugin is not a faithful implementation of the original 3DGS pipeline, especially because it lacks **tile-based software rasterization** and **spherical-harmonics-based view-dependent color**, I extended the plugin to support **view-dependent color reconstruction** and applied several **data-layout optimizations** to optimize its performance (in GPU memory bandwidth context). I also carried out a **preliminary performance estimation on mobile**.

#### What Remains Unimplemented / To Be Implemented

Several important parts of the original plan remain unimplemented. First, the current UE-based pipeline still does not reproduce the **tile-based software rasterization** stage of the original 3DGS method. Because of this, the costly sorting behavior that **MobileGS** is designed to optimize does not appear in the current implementation. Also, the **MLP-based component** used in MobileGS has not yet been implemented. 

To move forward, a more faithful 3DGS implementation will likely require a more suitable rendering abstraction and a **compute-shader-driven pipeline**, rather than relying on the current plugin structure.

#### Technical Issues

The main technical issue is that the current implementation uses the **Niagara particle system** to generate one GPU instance for each Gaussian. This introduces significant overhead and prevents all Gaussian primitives from being rendered efficiently in a single pass.

A second issue is that Niagara depends on translucent hardware rasterization to perform blending automatically. This design is fundamentally incompatible with the tile-based software rasterization used in the original 3DGS algorithm. As a result, it is difficult to reproduce the original 3DGS pipeline faithfully within this framework.

Because of these limitations, the current UE plugin does not show the high sorting cost that MobileGS specifically targets. This means that the present implementation is not suitable for a fair **performance evaluation of MobileGS**.

#### Anticipated Scope Changes

Given the current findings, I expect the scope of the project to shift. Instead of directly evaluating MobileGS on top of the existing Niagara-based UE plugin, the next stage will likely focus on building a **more faithful 3DGS rendering pipeline** in the mobile UE context using a more appropriate rendering abstraction and compute shaders driven rendering. With such a pipeline, the project could then compare the performance of **tile-based 3DGS**, **non-tile-based hardware-rasterized rendering**, and **MobileGS-style optimizations** more meaningfully.

#### Video
(placed 2 3DGS object with 360k GS points to test performance, while current implement won't handle multiple 3DGS object's occlusion correctly since correct occlusion requires globaling sorting while Niagara objects doesn't share buffer)
https://youtu.be/d36EueXUPNM


## References 
- Mobile-GS (latest paper as the primary academic reference for mobile-oriented 3DGS)
  - Paper: [https://arxiv.org/abs/2603.11531](https://arxiv.org/abs/2603.11531)

- UE 3DGS repositories as engineering references:
  - [https://github.com/Italink/GaussianSplattingForUnrealEngine](https://github.com/Italink/GaussianSplattingForUnrealEngine)
  - [https://github.com/xverse-engine/XScene-UEPlugin](https://github.com/xverse-engine/XScene-UEPlugin)
