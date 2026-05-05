# CS431 BrickGPT Assignment: Final Performance Analysis Report

## 1. Project Overview
This report provides a quantitative and qualitative analysis of the BrickGPT (Llama 1B fine-tuned) model in generating 3D LEGO structures via LDraw code. A total of 10 complex tasks were evaluated through a systematic benchmarking process.

## 2. Task Tracking & Metric Table
| Structure ID | Model Description | Semantic Score | Structural Score | Verdict |
|:---|:---|:---:|:---:|:---|
| afe3bd8e... | Angular chair with staggered arms | 8/10 | 8/10 | **PASS** |
| 41a68e76... | High backrest chair | 9/10 | 10/10 | **PASS** |
| a2f9d49d... | High backrest with square seat | 9/10 | 9/10 | **PASS** |
| 07c760a4... | Textured cylindrical pot | 7/10 | 8/10 | **PASS** |
| 6f5da8db... | Minimalistic rectangular table | 10/10 | 10/10 | **PASS** |
| 2e301277... | Tall guitar with tiered blocks | 7/10 | 6/10 | **PASS** |
| 1a5ff837... | Sturdy rectangular table | 9/10 | 9/10 | **PASS** |
| 676e6580... | Minimalistic tiered chair | 8/10 | 8/10 | **PASS** |
| 4bd74b8a... | Pillar table with angled base | 8/10 | 9/10 | **PASS** |
| 9185e1aa... | Complex stepped vessel | 7/10 | 7/10 | **PASS** |

## 3. Judging Methodology (Part C Analysis)
The judging process utilized a dual-model verification system. Gemini 1.5 Pro acted as the "Senior Technical Auditor," cross-referencing the LDraw output against the semantic intent of the prompts.

### 3.1 Key Findings
* **Precision in Primitives**: BrickGPT shows high accuracy in generating foundational furniture (tables, chairs) with symmetrical layouts.
* **Complexity Handling**: In "Staggered" or "Tiered" descriptions (e.g., the Vessel or Guitar), the model successfully identified vertical layering but occasionally struggled with precise coordinate offsets for floating components.
* **Code Reliability**: 100% of the generated files were syntax-compliant with the LDraw standard.

## 4. Qualitative Comparison (Part B Insights)
Comparative testing against frontier models (Qwen3 and Gemini) revealed that while BrickGPT is highly efficient for targeted 3D generation, frontier models provide slightly higher aesthetic "flair" in part selection for non-standard structures.

## 5. Conclusion
Based on the audited metrics, BrickGPT meets the professional requirements for automated LDraw generation. All 10 structures maintained structural integrity and high fidelity to the original design descriptions.