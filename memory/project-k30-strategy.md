---
name: K30 전략 전환 — llama.cpp 백엔드 + opt_kernel 커스텀 포맷
description: K-quant는 llama.cpp에 위임, opt_kernel은 커스텀 양자화에 집중. 추후 자체 구현으로 전환.
type: project
---

## 전략 전환 (2026-03-16 결정, 2026-03-18 방향 재확인)

### 배경

K29 실험 결과 확정:
- Tier 2 (C++ ATen) = Tier 1 (Python) = ~32 tok/s (ATen 오버헤드 지배적)
- Tier 3 (raw pointer): 이론상 맞지만 Mamba+Attention hybrid에서 segfault/dtype 문제 폭증
- llama.cpp: 동일 모델 208 tok/s (Q6_K), 안정적
- K-quant GEMV를 llama.cpp 수준으로 직접 최적화하는 것은 수주~수개월 소요

### 2026-03-18 종합 진단 결과

**K31 Tier 3 현황:**
- Low-level 컴포넌트: 90% 완료 ✅
  - make_hybrid_layer_fn, MambaLayerPtrs, CUDA kernels 모두 존재
  - Allocated allocation utilities 구현 완료
  - Type 0, 1, 2 개별 테스트 통과
- Integration: 실패 ❌ (반복 패턴)
  - Hybrid model (Mamba + Attention): dimension matching 문제
  - 전체 시스템 init: 미해결
  - 남은 예상 시간: 16-26시간

**핵심 발견:**
```
K29: Tier 3 시도 → segfault → 전략 전환
K31: Low-level 작동 ✅, Integration 실패 ❌ (동일 패턴 반복)
```

### 결정 (2026-03-18 재확인)

**Phase 1 (★★★ 현재 우선): llama.cpp 백엔드 통합**
- GGUF 표준 포맷: llama.cpp가 처리 (5-6x 빠름)
- 커스텀 포맷 (Grid VQ, Learned LUT, NVFP4, INT8 Delta 등): opt_kernel 사용
- IE에서 포맷별 자동 dispatch
- **목표: Qwen3.5-0.8B를 llama.cpp로 로드하여 208 tok/s 성능 확인**

**Phase 2 (보류): opt_kernel 자체 고성능 구현**
- llama.cpp의 핵심 기법을 CUDA C++로 직접 구현 (메인 어젠다)
- Triton 불가 최적화: warp intrinsics, Tensor Core 직접 제어, cp.async
- 목표: llama.cpp parity 또는 초과
- **K31 Tier 3 low-level 작업들은 Phase 2에서 재활용**

### 통합 방식

```
inference-engine
  ├─ 표준 GGUF 포맷 (Q2_K ~ Q8_K, IQ*)
  │   └─ llama.cpp 백엔드 (subprocess 또는 shared library)
  │       - llama-cli / llama-server
  │       - 또는 libllama.so + Python binding
  │
  ├─ 커스텀 포맷 (60+ 포맷)
  │   └─ opt_kernel CUDA 커널
  │       - Grid VQ, Learned LUT, NVFP4, INT8 Delta, etc.
  │
  └─ BF16/FP16 (비양자화)
      └─ torch.mm / cuBLAS
```

### opt_kernel의 집중 영역

1. **커스텀 양자화 GEMV/GEMM**: llama.cpp에 없는 60+ 포맷
2. **Fused SwiGLU/RMSNorm/QKV**: 이미 구현됨, 커스텀 포맷 전용
3. **K27 Integer-Domain Compute**: dp4a/IMMA for 커스텀 포맷
4. **추후**: K-quant GEMV 자체 구현 (Phase 2, llama.cpp 기법 참조)

### K31 벤치마크 결과 (2026-03-16)
- GEMV 커널: 2.35ms/token (24 layers) = 전체의 7.6%
- ATen 오버헤드: 28.6ms/token = 전체의 92.4%
- dp4a/preq 최적화: GB10에서 효과 없음 (FP32 ALU가 이미 강력)
- **결론: 커널 최적화 아닌 프레임워크 오버헤드 제거(Tier 3)가 해법**

### Phase 2 (Tier 3) 상세 계획
- `Docs/K31_tier3_raw_launcher_plan.md` 참조
- 핵심: Mamba Tier 3 (make_mamba_layer_fn) 구현 필요
- 예상: 16-26시간, 목표 200-300 tok/s

**Why:** K-quant 최적화에 수개월 투자하는 대신, llama.cpp 통합(수일)으로 즉시 5x 성능 확보. opt_kernel은 차별화 가치가 있는 커스텀 포맷에 집중. Tier 3는 llama.cpp 의존 제거를 위한 장기 투자.
**How to apply:** 새 작업 시 "이 포맷이 llama.cpp에 있는가?" 먼저 확인. 있으면 llama.cpp 위임, 없으면 opt_kernel 구현. Tier 3 작업은 Phase A→B→C→D 순서.
