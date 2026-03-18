---
name: K29 Qwen3.5 Tier 3 Decode 최우선 과제
description: llama.cpp 대비 3-6x TG 갭 해소 — GEMV 프로파일링 → Tier 3 활성화 → CUDA Graph
type: project
---

## 최우선 과제 (2026-03-15 지정)

상세 계획: `Docs/K29_qwen35_tier3_plan.md`

### 실행 순서

1. **Step 0**: GEMV 커널 프로파일링 — 31ms/tok 중 GEMV vs overhead 비율 확인
2. **Step 1a/1b**: 병목에 따라 GEMV 최적화 또는 Tier 2 model forward 활성화
3. **Step 2**: Tier 3 sym_decode_step Qwen35 분기 (15 raw kernel launches)
4. **Step 3**: CUDA Graph 활성화

### 이미 완료된 작업
- qwen35_helpers_kernel.cu: 4개 CUDA 커널 (sigmoid_gate_mul, head_rms_norm, partial_rope, q_gate_split)
- O proj dimension 버그 수정 (sym/asym/q8)
- IE forward_fused.py: output_gate 차단 해제 + mixed format 수정
- IE forward.py: hybrid model 재양자화 skip

### 핵심 통찰
- Tier 2 (C++ ATen) ≈ Tier 1 (Python) ≈ 32 tok/s → Python dispatch가 주 병목이 아닐 수 있음
- GEMV 커널 자체가 llama.cpp 대비 느릴 가능성 → **프로파일링 필수**

### Step 0 결과 (2026-03-15)
- GEMV(aten::mm) = 1.5%, **dequant copy = 34.5%** = 최대 병목
- 6 attn layers → fused K-quant matmul 사용 (55 calls)
- 18 Mamba layers → Python fallback (fused_mamba_layer_forward_sym returns None)
- **Mamba fused 실패 원인**: alpha/beta/out_proj가 Q8_0 format → sym check 실패
- _ensure_sym_cached로 Q8_0→Q6_K 변환 캐시 추가했으나 여전히 None 반환
- **다음 조사**: fused_mamba_layer_forward_sym C++ 내부에서 어떤 체크가 추가 실패하는지
- Qwen35 attn fused: OK (300/300 성공), Mamba fused: 0/900 실패

### K29 Step 2 진행 (2026-03-16)
- 2a~2g 전부 구현 완료 (opt_kernel + IE 양쪽)
- Tier 2 C++ model forward: **활성화 성공** (`cpp_model_forward: True`)
- Tier 3 sym_decode: **init 실패** (`expected Half but found Float`)
  - 원인: Mamba layer의 conv_weight, A_log가 FP32
  - sym_decode_init이 모든 weight를 FP16으로 기대
- TG: 29.8 tok/s (Tier 2, 이전과 동일)

### 다음 필요 작업
1. Mamba weight FP32→FP16 변환을 _try_pack_cpp_model_weights에서 수행
2. 또는 sym_decode_init에서 FP32 tensor도 처리하도록 수정
3. sym_decode_step에 make_mamba_layer_fn 추가 (Tier 3 Mamba)

**Why:** Tier 3 활성화가 llama.cpp 갭 해소의 유일한 해법이지만 Mamba+Attention hybrid에서는 Mamba도 Tier 3여야 동작.
**How to apply:** _try_pack_cpp_model_weights의 Mamba packing에서 conv_weight, A_log를 _fp16()으로 변환. 또는 sym_decode_init에서 dtype 자동 변환.
