---
name: K29 Qwen3.5 Tier 3 상태 (2026-03-18)
description: Mamba FP16 변환 완료, 다음 단계 정리
type: project
---

## 2026-03-18 완료

### ✅ Mamba Weight FP32→FP16 변환
**문제**: `expected Half but found Float` error in Tier 3 init
- 원인: Mamba layer의 conv_weight, A_log, dt_bias, norm.weight가 FP32로 저장됨

**해결**: `forward_cpp.py` 수정
- `_fp16()` → `_ensure_fp16()` 리팩토링
- dtype 검증, 명시적 FP16 변환, 로깅 추가
- 모든 Mamba/Attention weights에 적용

**결과**:
- ✅ All 18 Mamba layers: FP16 weights verified
- ✅ Tier 3 forward: PASS
- ✅ Prefill: 861ms for 10 tokens
- ✅ Decode: 39.5ms/tok (25.3 tok/s)

**커밋**: `11b40b3 feat(K29): Tier 3 Mamba weight FP32→FP16 conversion`

---

## 2026-03-18 추가 완료

### ✅ Tier 3 sym_decode_step Mamba 분기 구현 확인
**작업**: C++ `make_hybrid_layer_fn`에서 Mamba/Attention 분기 확인
- `launch_conv1d_decode_raw`: conv_state update
- `launch_gated_delta_rule_raw`: recurrent_state SSM computation
- 18 Mamba + 6 Attention 모두 처리

### ✅ CUDA Graph Capture 성공
**결과**:
- Capture: ✓
- Without Graph: 6.675 ms/step
- With Graph: 5.907 ms/step
- Speedup: 1.13x

### ✅ Tier 2 vs Tier 3 Benchmark
**결과** (GB10):
- Tier 2 (C++ ATen): 157.1 tok/s (6.366 ms/token)
- Tier 3 (Raw Kernels): 151.9 tok/s (6.585 ms/token)
- Speedup: 0.97x (Tier 3 slightly slower on GB10)

**분석**: GB10은 CUDA kernel 최적화가 부족하여 Tier 3가 느림. RTX 3090에서는 더 빠를 것으로 예상.

---

## 다음 단계 (Next Steps)

### 1. RTX 3090 재벤치마크
**목표**: RTX 3090에서 Tier 3 성능 검증
- GB10: Tier 3가 0.97x로 느림
- RTX 3090: CUDA kernel 최적화로 1.5-2x expected

**작업**:
- RTX 3090 환경에서 동일 벤치마크 실행
- CUDA Graph speedup 확인
- Tier 2 vs Tier 3 비교

### 2. Mamba Kernel 최적화
**현재**: FP16 weights 사용
- llama.cpp는 SSM 연산에 FP32 사용
- FP32 SSM 연산으로 정확도 향상 가능

**작업**:
- SSM state를 FP32로 변경 (recurrent_state, conv_state)
- FP32→FP16 변환 최적화
- 정확도 vs 속도 트레이드오프 분석

### 3. 긴 시퀀스 성능 테스트
**목표**: 512+ token 시퀀스에서 성능 확인
- 현재: 1 token decode 기반
- 장기 시퀀스에서의 성능 저하 확인

**작업**:
- 128, 512, 1024 token 벤치마크
- 메모리 사용량 모니터링
- KV cache + SSM cache 효율 확인

---

## 기술 세부사항

### Mamba Layer 구조 (Qwen3.5-0.8B)
```
- 18 Mamba layers + 6 Attention layers
- Mamba dimensions:
  - inner_dim = 2048
  - n_heads = 16
  - head_dim = 128
  - state_dim = 128 (recurrent_state)
  - conv_kernel = 4
- Weights requiring FP16:
  - conv1d.weight [6144, 4]
  - A [16] (A_log)
  - dt_proj.bias [16]
  - norm.weight
```

### SSM Cache 설정
```python
SSMCacheManager(
    n_layer=24,
    mamba_layers=[0,1,2,4,5,6,8,9,10,12,13,14,16,17,18,20,21,22],
    inner_dim=2048,
    conv_kernel=4,
    n_heads=16,
    head_dim=128,
    state_dim=128,  # Must equal head_dim!
)
```

---

## Why
Tier 3가 완전히 작동해야 llama.cpp 대비 3-6x 속도 향상 가능.

## How to apply
1. sym_decode_step에 Mamba 분기 추가
2. BF16 모델로 테스트
3. 프로파일링으로 병목 확인
