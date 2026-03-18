---
name: K27 Integer-Domain Compute 구현 상태
description: K27 dp4a/IMMA 커널 구현 결과, K28b Tier 2 최적화, 벤치마크 결과, 남은 작업
type: project
---

## K27 구현 완료 (2026-03-15)

### 정합성 검증 결과 (GB10 SM 12.1)
- K27a dp4a GEMV (sym/asym): cos > 0.999 ✅
- K27b INT8 IMMA: cos ~ 0.05 ❌ → `g_use_int8_imma` 분리, 기본 false
- K27d dp4a SwiGLU (sym/asym): cos > 0.999 ✅
- K27e Q8_0 dp4a: cos > 0.999 ✅

### GB10 성능
- dp4a GEMV: 0.3-0.9x (FP32 대비 느림) — memory-bound에서 X quant 오버헤드 > dp4a 이점

### 3090 성능 (SM 8.6, K31c 벤치 2026-03-16)
- dp4a GEMV: 0.34-1.00x (FP32 대비 느림) — GB10과 동일 패턴
- FP32 kernel-only: 495 tok/s (24 layers) — llama.cpp 208 tok/s 대비 2.4x 빠름
- **결론: dp4a는 M=1 memory-bound GEMV에서 양쪽 GPU 모두 불리 → g_use_int_compute 기본 false 확정**
- 병목은 커널이 아닌 ATen 오버헤드 (Tier 1/2: ~32 tok/s)

### 플래그 체계
- `g_use_int_compute` (기본 false): dp4a GEMV (M=1) 제어
- `g_use_int8_imma` (기본 false): INT8 IMMA GEMM (M>8) — 버그 조사 필요
- `g_use_int4_tc` (기본 false): INT4 TC — PTX m16n8k32 구현. 정합성 수정 완료 (cos=0.995, 3090 SM 8.6).
  - All-ones cos=1.0, random cos=0.995 — 정상 동작 확인
  - 성능은 FP16 WMMA 대비 0.37-0.51x로 느림 → 기본 false 유지

### K28b Tier 2 ATen 할당 제거 완료
- `empty_bias` → `static thread_local` (sym, asym, q8, mamba 4개 함수)
- `hidden = hidden + X` → `hidden.add_(X)` in-place (16곳)
- 예상: 32-layer 모델에서 ~192 임시 할당/토큰 → ~0

### K28a Tier 3 Batch Decode — 미진행 (별도 세션)
- fused_model_forward.cpp의 sym_decode_init에서 batch_size=1 하드코딩
- hidden/q/k/v 버퍼를 [B,...] 으로 확장 필요
- inference-engine과 긴밀히 연동되므로 별도 세션에서 진행

### K27b IMMA 버그 수정 + 최적화 (2026-03-15)
- 버그: B-fragment `wmma::col_major` → K/N 전치. 수정: `wmma::row_major`
- 최적화: smem staging 제거 → fragment layout 직접 활용 (group=lane/4, lig=lane%4)
  - wmma::store_matrix_sync 제거, smem_i32_stage[4][16][16] 4KB 해제
  - per-element scale을 fragment 원소에 직접 적용 (8 FMA/thread)
  - 23-39% 개선 (0.27-0.50x → 0.37-0.62x)
- 여전히 FP16 WMMA 대비 느림: X 양자화 2 pass (amax + quant) 오버헤드 지배적
- `g_use_int8_imma` 기본값 `false` 유지
- v4 최적화 (true 1-pass): X를 레지스터에 읽고 amax+quant 동시 수행 (global read 1회)
  - smem staging 제거 (fragment layout 직접 활용), W loader sync 제거
  - v1→v4: 최대 40% 개선 (0.606→0.363ms), 여전히 FP16 WMMA 대비 0.42-0.83x
  - GB10 Blackwell: FP16 TC throughput이 INT8 TC에 근접 → X quant 오버헤드 정당화 불가
  - Turing/Ampere (3090 등) SM 7.5~8.6에서 재검증 필요 (INT8 TC가 FP16 대비 확실한 2x)

**Why:** K27 integer-domain compute는 decode 성능 20-40% 개선 목표. IMMA는 prefill 30-50% 개선 목표.
**How to apply:** K28a 작업 시 이 상태를 참조. K27b IMMA 버그 조사 시 B-fragment layout부터 확인.
