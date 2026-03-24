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


## References 
- Mobile-GS (latest paper as the primary academic reference for mobile-oriented 3DGS)
  - Paper: [https://arxiv.org/abs/2603.11531](https://arxiv.org/abs/2603.11531)

- UE 3DGS repositories as engineering references:
  - [https://github.com/Italink/GaussianSplattingForUnrealEngine](https://github.com/Italink/GaussianSplattingForUnrealEngine)
  - [https://github.com/xverse-engine/XScene-UEPlugin](https://github.com/xverse-engine/XScene-UEPlugin)
